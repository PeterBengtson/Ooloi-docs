# gRPC Server Streaming: Threading and Backpressure in Clojure

## Abstract

gRPC server streaming often works with **in-process transport** but fails over **network transport**, producing `RST_STREAM` cancellations and `CANCELLED` errors. In-process transport is forgiving: a write is little more than a method call. Network transport puts every write through HTTP/2 flow control, and a `StreamObserver` is not thread-safe, so a stream written from more than one thread, or written without regard to whether the transport can accept the message, is exposed there.

This guide shows the Clojure drainer pattern Ooloi uses. Each stream has exactly one writer, every write waits for the transport to be ready, and per-client bounded queues keep clients isolated from one another.

---

## The Problem

### A naïve background-thread approach

A sketch of the shape that fails, not Ooloi code:

```clojure
(defn broken-streaming-approach
  [event-queue response-observer]
  (let [consumer-thread 
        (Thread. 
         (fn []
           (try
             (while (not (.isInterrupted (Thread/currentThread)))
               (let [event (.take event-queue)]
                 (.onNext response-observer event))) ; ignores .isReady; not the only writer
             (catch InterruptedException _
               (.onCompleted response-observer))
             (catch Exception e
               (.onError response-observer e)))))]
    (.setDaemon consumer-thread true)
    (.start consumer-thread)))
```

* **With in-process transport**: appears to work.
* **With network transport**: `RST_STREAM` cancellations, dropped clients, degraded connections.

### Why it fails

* **A `StreamObserver` is not thread-safe.** The gRPC-java contract says implementations "are not required to be thread-safe", and that "if multiple threads will be writing to a StreamObserver concurrently, the application must synchronize calls". Nothing in this shape makes the consumer thread the stream's only writer: any other path that writes to the same observer — a registration confirmation sent from the handler thread, say — writes concurrently with it, unsynchronised.
* **It ignores readiness.** `.take` followed by `.onNext` writes as fast as events arrive, whether or not the transport can accept them. `ServerCallStreamObserver` exposes `.isReady` and an `onReady` callback for exactly this; without them, outbound messages pile up behind HTTP/2 flow control instead of in a bounded application queue.
* **It ends the stream from the server side.** One exception calls `.onError` and an interrupt calls `.onCompleted`, both from the background thread, so a single bad event terminates the client's feed.
* **`Context/wrap` addresses none of this.** It carries request-scoped values (deadline, cancellation, context keys) onto another thread; it does not serialise writes or wait for readiness.

---

## The Clojure Drainer Pattern

The drainer gives each stream exactly one writer and gates every write on readiness, while application code still enqueues events from any thread. Each client gets:

* A bounded queue for its events
* A single-thread executor whose thread, `event-drain-<client-id>`, is the only thread that calls `.onNext` on that stream; it is wrapped by `Context/currentContextExecutor`, so its tasks run with the call's gRPC `Context`
* A drainer that writes only while `.isReady` holds and the call is not cancelled, and that the `onReady` handler restarts when the HTTP/2 window refills
* An `AtomicBoolean` mutex, so that kicks arriving from several threads at once still produce a single drain loop

The function below is `start-stream-drainer!` in `backend/src/main/clojure/ooloi/backend/grpc/server.clj`, with the comment at the final `catch` condensed to `;; …` (its content is given in the next section):

```clojure
(defn- start-stream-drainer!
  "Backpressure-aware, single-writer sender for one client.
   Uses a single-thread executor bound via Context.currentContextExecutor,
   so no explicit (.wrap ctx ...) is needed."
  [client-id ^java.util.concurrent.ArrayBlockingQueue event-queue
   ^io.grpc.stub.StreamObserver response-observer server-component]
  (let [registry (:connection-registry server-component)
        ^io.grpc.stub.ServerCallStreamObserver server-obs
        (cast io.grpc.stub.ServerCallStreamObserver response-observer)
        ;; capture for clarity; exec below already propagates this Context
        ctx (io.grpc.Context/current)
        ;; single-thread executor for this client's outbound path
        raw-exec (java.util.concurrent.Executors/newSingleThreadExecutor
                  (reify java.util.concurrent.ThreadFactory
                    (newThread [_ r]
                      (doto (Thread. r (str "event-drain-" client-id))
                        (.setDaemon true)))))
        ;; executor that propagates the current gRPC Context
        exec (io.grpc.Context/currentContextExecutor raw-exec)
        draining (java.util.concurrent.atomic.AtomicBoolean. false)
        drain! (fn []
                 ;; DEFENSIVE CHECK 1: the cheap fast path. It skips the submit in the
                 ;; common case, but it cannot make the submit safe — .isShutdown is a
                 ;; sample of a concurrently-mutating executor, not a lock, so the state
                 ;; can change between this answer and the .execute below. The catch is
                 ;; what actually covers that; this only avoids paying for it usually.
                 (when-not (.isShutdown raw-exec)
                   (try
                     (.execute exec
                               (reify Runnable
                                 (run [_]
                                   ;; CONCURRENCY MUTEX: Only one drain operation per client
                                   ;; Prevents race conditions when drain! called from multiple threads
                                   ;; (send-server-event, send-piece-event, onReady handler)
                                   (when (.compareAndSet draining false true)
                                     (try
                                       (loop []
                                         ;; DEFENSIVE CHECK 2: Stop draining if executor shuts down mid-process  
                                         ;; Component may be halted while processing 1000+ queued events
                                         ;; Combined condition: connection ready AND not cancelled AND executor alive
                                         (when (and (.isReady server-obs)
                                                    (not (.isCancelled server-obs))
                                                    (not (.isShutdown raw-exec)))
                                           (if-some [ev (.poll event-queue)]
                                             (do
                                               ;; A throw here would otherwise escape the drain
                                               ;; thread to the JVM's default handler, killing the
                                               ;; thread and abandoning everything still queued
                                               ;; behind this event. Record it and carry on with
                                               ;; the next: one event is lost, not the feed.
                                               (try
                                                 (.onNext server-obs ev)
                                                 (catch Exception e
                                                   (tp/log-error e)))
                                               (recur))
                                             ;; queue empty while ready
                                             nil)))
                                       (finally
                                         ;; MUTEX CLEANUP: Always release draining lock
                                         (.set draining false)))))))
                     ;; …
                     (catch java.util.concurrent.RejectedExecutionException _ nil))))]

    ;; when HTTP/2 window refills, gRPC calls this → try sending again
    (.setOnReadyHandler server-obs
                        (reify Runnable
                          (run [_] (drain!))))

    ;; on client cancel, tear down and remove from registry (identity-aware)
    (.setOnCancelHandler server-obs
                         (reify Runnable
                           (run [_]
                             (handle-stream-cancel server-component client-id
                                                   server-obs raw-exec))))

    ;; store handles for publishers and cleanup
    (swap! registry update client-id assoc
           :server-obs server-obs
           :drain drain!
           :exec raw-exec)

    ;; initial kick (transport may already be ready)
    (drain!)

    ;; return drain function for immediate use
    drain!))
```

### Why the submit carries its own catch

The `.isShutdown` check before `.execute` is a fast path, not a guard. It samples an executor that another thread can shut down at any moment — a cancellation, a Disconnect or a server halt — so the state can change between the answer and the submit. The `catch` of `RejectedExecutionException` is what covers that window: the rejection is the race-free answer that no amount of pre-checking can give. It is the ordinary teardown condition, a broadcast reaching a client whose drainer has just gone, with nothing left to deliver and nothing gone wrong, so it is deliberately not recorded; recording it would make a normal event look like a failure.

### Why `.onNext` carries its own catch

Nothing sits above the drain thread. The `Runnable` is handed to a bare `Executors/newSingleThreadExecutor` via `.execute`, so a throw escaping it goes to the JVM's default uncaught-exception handler: a trace on stderr that no part of Ooloi ever sees, and a dead drain thread. Everything still queued behind the failed event is abandoned. The `finally` does release the mutex, so the feed itself resumes on the next `send-*-event` or `onReady` kick — the loss is the backlog, not the connection, and it is silent.

Recording the throwable through `tp/log-error` puts it on the one boundary that exists ([ADR-0017 §Surfacing Unexpected Runtime Failures](../ADRs/0017-System-Architecture.md)), and the `recur` keeps the queue moving past it. One event is lost; the feed is not. This is the *record* arm of ADR-0017 §What a Deliberate `catch` May Do — stepping aside is not available here, because stepping aside means letting the throwable reach a boundary that watches this thread, and no such boundary exists above a bare single-thread executor.

Note what is deliberately *absent*: no eviction of the failing event, no retry, no termination of the stream. Terminating would mean a server-initiated `.onCompleted`, which races the client's own close — the self-closing race documented under Graceful Disconnect below. And there is nothing to reconcile afterwards: [ADR-0031 §Connection Management](../ADRs/0031-Frontend-Event-Driven-Architecture.md) rules out queue replay and reconciliation outright, on the grounds that "clients never need to 'catch up' on missed events — a new connection fetches current state when it needs it". A dropped event is therefore a recorded fact, not a repair job.

The `setOnCancelHandler` delegates to a named, identity-aware function. Extracting this body matters: gRPC fires the cancel handler asynchronously when it observes the stream's cancellation, and there is no guarantee that the entry in the registry at fire time is still *this* stream's entry. A graceful Disconnect + re-register cycle (the canonical `switch-to!` flow) can install a fresh entry under the same client-id between the cancellation and the handler running; a non-identity-aware dissoc-by-client-id would wipe that fresh entry.

Both disconnect paths send their broadcast through one function, `broadcast-client-disconnected!`, shown first; `handle-stream-cancel` follows. Both are in `server.clj`:

```clojure
(defn- broadcast-client-disconnected!
  "Single source of truth for the :server-client-disconnected broadcast.
   Emitted only when the server transport is :network; the always-on in-process
   backend suppresses collaboration broadcasts (see body, ADR-0036 / ADR-0040).

   Called by both disconnect paths — handle-stream-cancel (involuntary,
   channel close backstop) and handle-disconnect (voluntary Disconnect RPC).
   Producing the broadcast from one place guarantees the downstream consumer
   (notably the host's collaboration notification subscriber) sees a
   byte-identical event regardless of which path the disconnect took. The
   identifying metadata is read from the entry that was just removed (`old`
   in the swap-vals! result) so it remains available even after the dissoc.

   The departing client's :client-id is intentionally NOT broadcast — it is
   the sole authentication credential used by the api-authentication
   interceptor (CLIENT_ID_HEADER); leaking it to other clients would enable
   impersonation. :client-ip, :client-port, and :display-name are non-credential
   identifiers safe to share in-session (the network address is observable on
   the wire anyway, and :display-name is user-supplied for visibility)."
  [server-component _client-id old-entry new-registry]
  ;; Collaboration broadcasts are network-transport-only. The always-on local
  ;; in-process backend (ADR-0040) serves a single local client, so a
  ;; :server-client-disconnected broadcast there carries no collaboration
  ;; meaning, and its sole consumer — the host notification subscriber — filters
  ;; to network origin anyway. Suppressing at the source keeps the in-process
  ;; event stream free of collaboration noise.
  (when (= :network (:transport server-component))
    (let [meta (:metadata old-entry)]
      (send-server-event server-component
                         {:type :server-client-disconnected
                          :client-count (count new-registry)
                          :transport (:transport server-component)
                          :client-ip (:client-ip meta)
                          :client-port (:client-port meta)
                          :display-name (:display-name meta)}))))

(defn- handle-stream-cancel
  "Per-stream cancellation cleanup, identity-aware.

   The drainer executor is shut down unconditionally — its lifecycle is
   per-stream regardless of whether the registry entry has been replaced.
   The registry mutation and the :server-client-disconnected broadcast
   run only if this cancellation is for the CURRENT entry, witnessed by
   `identical?` on :server-obs. Protects re-registered entries from
   stale cancellation handlers fired after a graceful Disconnect+
   re-register cycle."
  [server-component client-id server-obs raw-exec]
  (try
    (.shutdownNow raw-exec)
    (finally
      (let [registry  (:connection-registry server-component)
            [old new] (swap-vals! registry
                                  (fn [reg]
                                    (if (identical? server-obs
                                                    (:server-obs (get reg client-id)))
                                      (dissoc reg client-id)
                                      reg)))]
        (when (not= old new)
          (let [entry (get old client-id)]
            ;; The entry was still ours when the stream ended, which means no Disconnect
            ;; RPC arrived — that removes the registry entry first and leaves this handler
            ;; a no-op. So reaching here IS an involuntary disconnection, in the sense
            ;; ADR-0018 and ADR-0024 give the word, witnessed atomically by the swap-vals!
            ;; above. A timeout is a subset of it and needs no special case: the deadline
            ;; expiring produces a cancellation that arrives here like any other, while the
            ;; interceptor separately counts it as a timeout.
            ;;
            ;; It is counted here rather than in the RegisterClient interceptor, where
            ;; :clients-disconnected-total and the timeout arm live: the interceptor knows
            ;; neither the client-id nor whether the registry entry survived, and its
            ;; ordering against this handler is undocumented. A server halt terminates its
            ;; clients through onComplete rather than here, so it is excluded for free.
            (stats/inc-server-stat! server-component :clients-disconnected-involuntary)
            ;; A dropped connection releases the client's pieces too (close-on-last-release), so an
            ;; involuntary drop closes an orphaned piece exactly as a clean unsubscribe does.
            (psl/release-client-pieces! entry new (:undo-manager-component server-component))
            (broadcast-client-disconnected! server-component client-id entry new)))))))
```

The closure captures `server-obs` (this stream's `ServerCallStreamObserver`). At cancel time, `swap-vals!` reads the current registry, checks whether `(:server-obs (get reg client-id))` is `identical?` to the captured one, and only dissocs on match. Everything after it is gated on `(not= old new)`, so a no-op (entry already replaced or absent) counts nothing, releases nothing and emits no spurious `:server-client-disconnected` event.

When the removal does take effect, the handler does three things. It counts the disconnection in `:clients-disconnected-involuntary`, because an entry still present when its stream ends means no `Disconnect` RPC arrived. It releases the departing client's pieces. And it calls `broadcast-client-disconnected!`, which sends `:server-client-disconnected` only from a **network** server, carrying `:client-count`, `:transport`, `:client-ip`, `:client-port` and `:display-name`. The departing client's `:client-id` is never broadcast: it is the credential the authentication interceptor checks, and sharing it would let another client impersonate it. The in-process server sends no such broadcast at all, since the single local client it serves has no collaboration peers to inform ([ADR-0040](../ADRs/0040-Single-Authority-State-Model.md)).

---

## Graceful Disconnect (RPC)

The unary `Disconnect` RPC is the *primary* path for client-initiated session termination, as a complementary peer of `RegisterClient` (see [ADR-0018 §Service Architecture](../ADRs/0018-API-gRPC-Interface-and-Events.md) and [ADR-0024 §Connection Lifecycle](../ADRs/0024-gRPC-Concurrency-and-Flow-Control-Architecture.md)). Used by `switch-to!` during transport switching and intended for any future client-side graceful-exit flows. The server-side handler, `handle-disconnect` in `server.clj`, with the comment explaining the absent `.onCompleted` condensed to `;; …` (detail 3 below gives it in full):

```clojure
(defn- handle-disconnect
  "Server-side handler for the Disconnect RPC.
   Client-id is read from the gRPC context (CLIENT_ID_HEADER injected by
   the api-client interceptor), never from the request payload — a client
   can only disconnect itself. A missing context header (nil) is rejected
   with {:success false}: we cannot determine the caller, so we don't
   silently do nothing. swap-vals! is atomic; cleanup and broadcast are
   gated on whether the dissoc actually fired so an already-disconnected
   call doesn't tear down resources or emit a spurious event."
  [_request server-component]
  (if-let [client-id (headers/get-client-id-from-context)]
    (let [registry  (:connection-registry server-component)
          [old new] (swap-vals! registry dissoc client-id)]
      (when (not= old new)
        (let [entry (get old client-id)]
          ;; The departed client releases every piece it held — close-on-last-release, exactly as an
          ;; unsubscribe would: a piece no remaining client still holds is closed.
          (psl/release-client-pieces! entry new (:undo-manager-component server-component))
          (when-let [raw-exec (:exec entry)]
            (.shutdownNow raw-exec))
          ;; …
          (broadcast-client-disconnected! server-component client-id entry new)))
      {:success true :error ""})
    {:success false :error "no client identity in gRPC context"}))
```

Three details matter:

1. **Identity from context, not payload.** The auth interceptor (installed for both `ExecuteMethod` and `Disconnect`) extracts `CLIENT_ID_HEADER` and stores it in the gRPC `Context`. Handlers read it via `headers/get-client-id-from-context`. This closes the attack surface where a client could try to disconnect another client by sending its id in the body. A missing context (interceptor not run, or unauthenticated method invocation) returns `{:success false}` — the call fails loudly rather than silently doing nothing.

2. **Idempotency via swap-vals!.** `swap-vals!` returns `[old new]` atomically. `(not= old new)` is true only when the dissoc actually removed something. So a no-op Disconnect (client-id already absent) returns success without releasing pieces, shutting down a non-existent drainer or firing the broadcast.

3. **No `.onCompleted` on the streaming call from this handler.** The intuitive design — have the Disconnect handler gracefully complete the `RegisterClient` server-streaming call via `.onCompleted` on its response observer — creates a *self-closing race*. The canonical caller is the same client that owns the streaming call (via `switch-to!`'s use of `disconnect-from-server`). A server-side `.onCompleted` fires the client's identity-aware `handle-stream-completed` callback, which calls `handle-server-disconnected`, which closes the API pool channels — and the unary Disconnect's response is still in flight on those very channels, causing `DEADLINE_EXCEEDED` on the client. The streaming call is cleaned up via `setOnCancelHandler` (the identity-aware backstop above) when the client tears down channels *after* `disconnect-from-server` returns. Because that backstop then finds the entry already gone, it is a no-op, and the PHASE 6 broadcast comes from this handler instead — on a network server; the in-process server sends none, as `broadcast-client-disconnected!` shows above.

Beyond the threading concerns above, both this handler and the cancel-handler backstop **release the departing client's pieces** through `psl/release-client-pieces!` — unsubscribing it from every piece it held and closing any thereby left with no subscribers, the deterministic close-on-last-release of [ADR-0022 §Piece Lifetime](../ADRs/0022-Lazy-Frontend-Backend-Architecture.md).

A **server shutdown** interacts with all of this and is specified rather than left to the reader: the order its four steps run in, why the wait for termination comes last, why registry removal must not precede observer completion if the cancel handler is to account for the disconnect exactly once, and why termination is never forced — a `.shutdownNow` reaching in-process streams hangs the process. See [ADR-0024 §Server Shutdown: Graceful Only, Never Forced](../ADRs/0024-gRPC-Concurrency-and-Flow-Control-Architecture.md#server-shutdown-graceful-only-never-forced).

---

## Queue Handling

A bounded queue prevents runaway memory use. If it is full, the oldest message is dropped before the new one is enqueued, and every drop is counted in the client's statistics. Both functions are in `server.clj`:

```clojure
(defn- track-dropped-event
  "Records statistics for a dropped event during queue overflow.

   Called when an old event is removed from queue to make room for new event."
  [server-component client-id event-type]
  (stats/increment-event-stats server-component client-id
                               {:event-type (keyword event-type)
                                :offer-success false
                                :event-bytes 0}))

(defn- offer-with-drop-oldest
  "Adds event to queue with drop-oldest backpressure handling.

   If queue is full, removes oldest events until new event can be added.
   Handles race conditions by looping until successful.

   Uses drop-oldest overflow handling to maintain queue capacity limits.
   Tracks every dropped event in statistics system.

   Returns: true when event is successfully added"
  [event-queue event-message server-component client-id event-type]
  (loop []
    (if (.offer event-queue event-message)
      ;; Successfully added without overflow
      true
      ;; Queue is full - drop oldest and retry
      (do
        (.poll event-queue)  ; Remove oldest event
        (track-dropped-event server-component client-id event-type)  ; Track the drop
        (recur)))))  ; Retry until successful
```

This way:

* Clients that fall behind don't block others.
* Queues stay within fixed limits.
* New events are always preferred over stale ones.

---

## Client Registration

The connection registry is a single atom, not an STM ref. Every change to it is one `swap!` or `swap-vals!`, so each registration, cancellation and disconnect updates it atomically; `swap-vals!` additionally hands the cancel and Disconnect paths the before-and-after pair from which they tell whether their own removal took effect. `setup-client-registry-and-drainer` adds the entry, stamped with the server's `:server-id`, and starts the drainer.

`register-client` in `server.clj`, with the comment on the connect broadcast condensed to `;; …`:

```clojure
(defn register-client
  "Handles client registration for real-time event streaming.
   
   Registers client with per-client backpressure-aware drainer:
   - Bounded queues (1000 events) with drop-oldest backpressure
   - Client ID validation prevents duplicate connections
   - Piece subscription tracking for targeted events
   - Uses ServerCallStreamObserver readiness + single-writer drainer
   
   Parameters:
   - server-component: Map containing :connection-registry and :server-statistics
   - request: RegisterClientRequest with client-id
   - response-observer: StreamObserver for real-time event delivery
   
   Returns: client-id string on successful registration"
  [server-component request response-observer]
  ;; Keep connection open - don't call onCompleted here
  (when request
    ;; Extract client-id from OoloiValue client_info
    (let [client-info (extract-client-info request)
          client-id (:client-id client-info)
          registry (:connection-registry server-component)]
      (when (validate-client-connection server-component client-info response-observer)

        ;; Set up registry entry and drainer infrastructure
        (let [drain! (setup-client-registry-and-drainer client-id server-component response-observer)
              registry-entry (get @registry client-id)
              event-queue (:event-queue registry-entry)]

          ;; Increment :clients-connected-total, now that we know we have a successful one
          (stats/inc-server-stat! server-component :clients-connected-total)

          ;; Record on the call's Context that this stream became a connection, so the
          ;; interceptor's terminal arms count its ending as a disconnection. A stream
          ;; refused by validate-client-connection never reaches here, and so is never
          ;; counted as having disconnected.
          (when-let [registered (.get REGISTERED_CONTEXT_KEY)]
            (.set ^AtomicBoolean registered true))

          ;; Send confirmation event through drainer (explicit in main flow)
          (send-confirmation-event server-component client-id event-queue response-observer drain!)

          ;; …
          (when (= :network (:transport server-component))
            (let [meta (get-in @registry [client-id :metadata])]
              (send-server-event server-component
                                 {:type :server-client-connected
                                  :client-count (count @registry)
                                  :transport (:transport server-component)
                                  :client-ip (:client-ip meta)
                                  :client-port (:client-port meta)
                                  :display-name (:display-name meta)})))

          ;; Return client-id on successful registration  
          client-id)))))
```

Once validation has passed, the function counts the connection in `:clients-connected-total` and sets the `REGISTERED_CONTEXT_KEY` flag on the call's `Context`, so that the event-connection interceptor counts this stream's eventual ending as a disconnection; a refused registration never sets it and is never counted as having disconnected. The connect broadcast follows the same rule as the disconnect broadcast: only a network server sends `:server-client-connected`, with the same non-credential fields and never the joining client's `:client-id`.

The confirmation is the first event on the new stream and travels through the drainer like every other. Its `:server-timestamp` is in epoch microseconds, and the client subtracts its own clock from it to compute its clock offset ([ADR-0018 §Event Consistency Requirements](../ADRs/0018-API-gRPC-Interface-and-Events.md#event-consistency-requirements)):

```clojure
(defn- send-confirmation-event
  "Sends client registration confirmation event through the drainer.
   
   Creates and queues the confirmation event with proper validation and
   immediately triggers the drainer for delivery. This ensures consistent
   threading model for all events including the initial confirmation.
   
   Parameters:
   - client-id: The client identifier for the confirmation  
   - event-queue: ArrayBlockingQueue for the client's events
   - response-observer: StreamObserver for the client connection
   - drain!: The drain function returned from start-stream-drainer!
   
   Returns: true if event was queued successfully, false otherwise"
  [server-component client-id event-queue response-observer drain!]
  (let [confirmation-event {:type :client-registration-confirmed
                            :client-id client-id
                            :message "Registration successful"
                            :server-timestamp (time/epoch-usec)}
        timestamp (time/epoch-usec)]
    ;; Validate confirmation event structure
    (validate-event-structure confirmation-event "client-registration")
    ;; Derive event-type string from validated :type keyword field
    (let [event-type (name (:type confirmation-event))]
      ;; Queue confirmation event and kick drainer for immediate delivery
      (if (create-and-queue-event server-component client-id event-queue response-observer confirmation-event event-type timestamp "register-client")
        (do
          ;; Use drain function for immediate delivery
          (drain!)
          true)
        false))))
```

---

## Event Publishing

Sending an event means enqueueing it for each recipient and kicking that recipient's drainer. The timestamp is taken once per event, in epoch microseconds, and every recipient gets the same one. `deliver-to-client!` wraps each recipient's delivery in its own `catch`, so a failure against one client is recorded and the broadcast continues to the rest. `send-piece-event` has the same shape, filtered to the clients subscribed to the event's `:piece-id`. All three are in `server.clj`:

```clojure
(defn- deliver-to-client!
  "Queues one event for one recipient and kicks its drainer, isolating that recipient
   from the rest of the broadcast.

   A broadcast must not be abandoned because one recipient fails. Each client's delivery
   is independent of every other's — that independence is the whole point of per-client
   queues — so a failure against one is not a reason to deny the others their event. And
   the failure has nowhere useful to go up the stack: a broadcast is triggered by a cancel
   handler or a mutation, neither of which can act on one recipient's delivery problem.
   Recording it on the boundary (ADR-0017) and continuing is the same treatment the drain
   gives a bad event, and for the same reason — see GRPC_STREAMING_THREADING_GUIDE
   §\"Why `.onNext` carries its own catch\"."
  [server-component client-id event-queue observer drain event-data event-type timestamp fn-name]
  (try
    (create-and-queue-event server-component client-id event-queue observer
                            event-data event-type timestamp fn-name)
    ;; Kick the drainer (no-op if not ready / not installed yet)
    (when drain (drain))
    (catch Throwable t
      (tp/log-error t))))

(defn send-server-event
  "Sends event to all connected clients with complete EventMessage metadata.

   Validates event structure once before broadcasting to all clients.
   Automatically derives protobuf event-type string from validated :type keyword field.
   Automatically supplies epoch-microsecond timestamp.
   Uses queue-based async delivery with drop-oldest overflow handling.
   After enqueue, triggers the client's drainer if present."
  [server-component event-data]
  ;; Validate once
  (validate-event-structure event-data "server-event")
  (let [event-type (name (:type event-data))
        timestamp (time/epoch-usec)
        registry  (:connection-registry server-component)]
    (doseq [[client-id {:keys [event-queue observer drain]}] @registry]
      (deliver-to-client! server-component client-id event-queue observer drain
                          event-data event-type timestamp "send-server-event"))))
```

`create-and-queue-event` builds the `EventMessage`, offers it with drop-oldest handling and records the delivery statistics. It never touches the transport: a cancelled stream is counted as a failed offer, and an unexpected throwable is recorded through `tp/log-error` and counted in `:internal-errors-total`. The `timestamp` it receives is the epoch-microsecond value taken by the sender:

```clojure
(defn create-and-queue-event
  "Creates EventMessage and enqueues with drop-oldest handling.

   IMPORTANT: Queueing is an internal concern and must not tear down the transport.
   This function never calls `.onError` on the observer. It returns true if the event
   was placed in the queue (possibly after dropping the oldest), false otherwise
   (which should be rare). Upstream callers can decide how to record drops.

   Parameters:
   - event-queue: ArrayBlockingQueue for the client
   - observer: StreamObserver for connection state checking
   - event-data: Clojure map event payload
   - event-type: String
   - timestamp: long (epoch microseconds)
   - function-name: String caller name (for debug logging only)

   Returns: boolean (true on success, false on failure)"
  [server-component client-id event-queue observer event-data event-type timestamp function-name]
  (try
    (if (not (.isCancelled observer))
      ;; Connection alive - queue the event
      (let [event-message (build-event-message event-data event-type timestamp)
            event-bytes (serialized-size event-message)
            offer-success (offer-with-drop-oldest event-queue event-message server-component client-id event-type)]
        ;; Collect event delivery statistics
        (stats/increment-event-stats server-component client-id
                                     {:event-type (keyword event-type)
                                      :offer-success offer-success
                                      :event-bytes event-bytes})
        offer-success)
      ;; Connection dead - collect stats for cancelled connection
      (do
        (stats/increment-event-stats server-component client-id
                                     {:event-type (keyword event-type)
                                      :offer-success false
                                      :event-bytes 0})
        false))
    (catch Throwable t
      ;; Internal server error during event queueing - collect error statistics.
      ;; These are unexpected failures in core event delivery infrastructure, which is
      ;; exactly why the counter alone is not enough: :internal-errors-total announces
      ;; that a bug happened (ADR-0025) and nothing else here can say which one.
      ;; Throwable rather than Exception is deliberate — ADR-0025's :internal category
      ;; covers resource exhaustion and system errors, not only Exceptions.
      (tp/log-error t)
      (stats/inc-server-stat! server-component :internal-errors-total)
      ;; Only increment client stat if client exists (defensive handling)
      (when-let [registry (:connection-registry server-component)]
        (when (get @registry client-id)
          (stats/inc-client-stat! server-component client-id :internal-errors)))
      false)))
```

---

## Key Points

1. Test streaming with **network transport**, not just in-process.
2. Give each stream exactly one writer: every `.onNext` goes through that client's single-thread drainer, because a `StreamObserver` is not thread-safe.
3. Respect `.isReady` and the `onReady` callback for backpressure.
4. Use bounded queues and drop-oldest to prevent memory issues.
5. Keep registry updates atomic: one `swap!` or `swap-vals!` per change, on a single atom.
6. Clean up executors and registry entries on cancel, identity-aware.
7. Wrap `.onNext` in its own `catch` — nothing watches the drain thread, so an escaping throw kills it and silently abandons the backlog.
8. Catch `RejectedExecutionException` at the submit — a pre-check of `.isShutdown` cannot close the race with teardown.

---

The result is a pattern for server streaming in Clojure that holds over network transport: queue → drainer → observer, with one writer per stream and every write gated on readiness.
