# gRPC Server Streaming: Threading and Backpressure in Clojure

## Abstract

gRPC server streaming often works fine with **in-process transport** but fails over **network transport**, producing `RST_STREAM` cancellations and `CANCELLED` errors. The difference is in threading: in-process mode ignores gRPC's constraints, while network transport enforces strict HTTP/2 flow control and Netty event loop affinity.

This guide shows a working Clojure solution using a drainer pattern that respects gRPC's threading contracts, enforces backpressure, and isolates clients with queue-based delivery.

---

## The Problem

### A naïve background-thread approach

```clojure
(defn broken-streaming-approach
  [event-queue response-observer]
  (let [consumer-thread 
        (Thread. 
         (fn []
           (try
             (while (not (.isInterrupted (Thread/currentThread)))
               (let [event (.take event-queue)]
                 (.onNext response-observer event))) ; WRONG THREAD
             (catch InterruptedException _
               (.onCompleted response-observer))
             (catch Exception e
               (.onError response-observer e)))))]
    (.setDaemon consumer-thread true)
    (.start consumer-thread)))
```

* **With in-process transport**: works fine.
* **With network transport**: fails immediately with `RST_STREAM` cancellations, clients drop, connections degrade.

### Why it fails

* In network transport, `.onNext` must be called on the correct Netty event-loop thread.
* Background threads calling `.onNext` break the HTTP/2 stream context.
* `Context/wrap` doesn't fix this — it only propagates request-scoped data, not transport thread affinity.

---

## The Clojure Drainer Pattern

The fix is to ensure `.onNext` runs inside the right gRPC context, while still letting application code enqueue events asynchronously. Each client gets:

* A bounded queue for its events
* A single-thread executor tied to `Context/currentContextExecutor`
* A drainer that respects `.isReady` and `.isCancelled`

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
                 ;; DEFENSIVE CHECK 1: Don't submit work to shut-down executor
                 ;; This prevents RejectedExecutionException when component is halted
                 ;; Efficiency: Avoids creating work that would immediately exit
                 (when-not (.isShutdown raw-exec)
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
                                             (.onNext server-obs ev)
                                             (recur))
                                           ;; queue empty while ready
                                           nil)))
                                     (finally
                                       ;; MUTEX CLEANUP: Always release draining lock
                                       (.set draining false)))))))))]

    ;; when HTTP/2 window refills, gRPC calls this → try sending again
    (.setOnReadyHandler server-obs
                        (reify Runnable
                          (run [_] (drain!))))

    ;; on client cancel, tear down and remove from registry, then tell everyone remaining
    (.setOnCancelHandler server-obs
                         (reify Runnable
                           (run [_]
                             (try
                               (.shutdownNow raw-exec)
                               (finally
                                 (swap! registry dissoc client-id)
                                 (send-server-event server-component {:type :server-client-disconnected :client-count (count @registry)}))))))

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

---

## Queue Handling

A bounded queue prevents runaway memory use. If it's full, drop the oldest message before enqueuing the new one:

```clojure
(defn- offer-with-drop-oldest
  "Adds event to queue with drop-oldest backpressure handling.
   
   If queue is full, removes oldest events until new event can be added.
   Handles race conditions by looping until successful.
   
   Uses drop-oldest overflow handling to maintain queue capacity limits.
   
   Returns: true when event is successfully added"
  [event-queue event-message]
  (loop []
    (if (.offer event-queue event-message)
      ;; Successfully added without overflow
      true
      ;; Queue is full - drop oldest and retry
      (do
        (.poll event-queue)  ; Remove oldest event
        (recur)))))  ; Retry until successful
```

This way:

* Clients that fall behind don't block others.
* Queues stay within fixed limits.
* New events are always preferred over stale ones.

---

## Client Registration with STM

Using STM keeps client registry updates consistent:

```clojure
(defn register-client
  "Handles client registration for real-time event streaming.
   
   Registers client with per-client backpressure-aware drainer:
   - Bounded queues (1000 events) with drop-oldest backpressure
   - Client ID validation prevents duplicate connections and enforces naming rules
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
      ;; Validate client-id: format (^[a-zA-Z0-9_-]{3,64}$), uniqueness, send gRPC errors
      (when (validate-client-connection server-component client-info response-observer)
        
        ;; Set up registry entry and drainer infrastructure
        (let [drain! (setup-client-registry-and-drainer client-id server-component response-observer)
              registry-entry (get @registry client-id)
              event-queue (:event-queue registry-entry)]
          
          ;; Send confirmation event through drainer (explicit in main flow)
          (send-confirmation-event client-id event-queue response-observer drain!)
          
          ;; Broadcast client connect event to all clients (including newly connected one)
          (send-server-event server-component {:type :server-client-connected :client-count (count @registry)})
          
          ;; Return client-id on successful registration  
          client-id)))))
```

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
  [client-id event-queue response-observer drain!]
  (let [confirmation-event {:type :client-registration-confirmed
                            :client-id client-id
                            :message "Registration successful"}
        timestamp (System/nanoTime)]
    ;; Validate confirmation event structure
    (validate-event-structure confirmation-event "client-registration")
    ;; Derive event-type string from validated :type keyword field
    (let [event-type (name (:type confirmation-event))]
      ;; Queue confirmation event and kick drainer for immediate delivery
      (if (create-and-queue-event event-queue response-observer confirmation-event event-type timestamp "register-client")
        (do
          ;; Use drain function for immediate delivery
          (drain!)
          true)
        false))))
```

---

## Event Publishing

Sending events just means enqueueing them and nudging the drainer:

```clojure
(defn send-server-event
  "Sends event to all connected clients with complete EventMessage metadata.

   Validates event structure once before broadcasting to all clients.
   Automatically derives protobuf event-type string from validated :type keyword field.
   Automatically supplies nanosecond precision timestamp.
   Uses queue-based async delivery with drop-oldest overflow handling.
   After enqueue, triggers the client's drainer if present."
  [server-component event-data]
  ;; Validate once
  (validate-event-structure event-data "server-event")
  (let [event-type (name (:type event-data))
        timestamp (System/nanoTime)
        registry  (:connection-registry server-component)]
    (doseq [[client-id {:keys [event-queue observer drain]}] @registry]
      ;; Create EventMessage and queue with per-client error propagation
      (create-and-queue-event event-queue observer event-data event-type timestamp "send-server-event")
      ;; Kick the drainer (no-op if not ready / not installed yet)  
      (when drain (drain)))))
```

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
   - timestamp: long (ns)
   - function-name: String caller name (for debug logging only)

   Returns: boolean (true on success, false on failure)"
  [event-queue observer event-data event-type timestamp function-name]
  (try
    (if (not (.isCancelled observer))
      ;; Connection alive - queue the event
      (let [event-message (build-event-message event-data event-type timestamp)]
        (offer-with-drop-oldest event-queue event-message)
        true)
      ;; Connection dead - log and collect stats
      (do
        (println "DEBUG: Skipping event queue for cancelled connection - will add stat increment here")
        false))
    (catch Throwable t
      (println "SERVER WARN: create-and-queue-event failed in" function-name ":" (.getMessage t))
      false)))
```

---

## Key Points

1. Test streaming with **network transport**, not just in-process.
2. Only call `.onNext` inside the right gRPC executor (`Context/currentContextExecutor`).
3. Respect `.isReady` for backpressure.
4. Use bounded queues and drop-oldest to prevent memory issues.
5. Keep registry updates consistent with STM.
6. Clean up executors and registry entries on cancel.

---

This gives you a stable, production-viable pattern for server streaming in Clojure: queue → drainer → observer, all under gRPC's threading rules.
