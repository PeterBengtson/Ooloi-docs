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
(defn start-stream-drainer!
  [client-id ^java.util.concurrent.ArrayBlockingQueue event-queue
   ^io.grpc.stub.StreamObserver response-observer registry]
  (let [^io.grpc.stub.ServerCallStreamObserver server-obs
        (cast io.grpc.stub.ServerCallStreamObserver response-observer)

        raw-exec (java.util.concurrent.Executors/newSingleThreadExecutor
                  (reify java.util.concurrent.ThreadFactory
                    (newThread [_ r]
                      (doto (Thread. r (str "event-drain-" client-id))
                        (.setDaemon true)))))

        exec (io.grpc.Context/currentContextExecutor raw-exec)
        draining (java.util.concurrent.atomic.AtomicBoolean. false)

        drain! (fn []
                 (.execute exec
                           (reify Runnable
                             (run [_]
                               (when (.compareAndSet draining false true)
                                 (try
                                   (loop []
                                     (when (and (.isReady server-obs)
                                                (not (.isCancelled server-obs)))
                                       (if-some [ev (.poll event-queue)]
                                         (do
                                           (.onNext server-obs ev)
                                           (recur)))))
                                   (finally
                                     (.set draining false))))))))]

    (.setOnReadyHandler server-obs (reify Runnable (run [_] (drain!))))
    (.setOnCancelHandler server-obs
                         (reify Runnable
                           (run [_]
                             (try (.shutdownNow raw-exec)
                                  (finally (swap! registry dissoc client-id))))))

    (swap! registry update client-id assoc
           :server-obs server-obs
           :drain drain!
           :exec raw-exec)

    (drain!)))
```

---

## Queue Handling

A bounded queue prevents runaway memory use. If it's full, drop the oldest message before enqueuing the new one:

```clojure
(defn offer-with-drop-oldest
  [event-queue event-message]
  (loop []
    (if (.offer event-queue event-message)
      true
      (do
        (.poll event-queue)  ; remove oldest
        (recur)))))
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
  [registry request response-observer]
  (let [client-id (:client-id (conv/proto->clj (.getClientInfo request)))]
    (when (validate-client-connection registry client-id response-observer)
      (dosync
        (let [registry-entry (create-registry-entry response-observer)
              event-queue (:event-queue registry-entry)]
          (alter registry assoc client-id registry-entry)
          (start-stream-drainer! client-id event-queue response-observer registry)
          client-id)))))
```

---

## Event Publishing

Sending events just means enqueueing them and nudging the drainer:

```clojure
(defn send-server-event
  [server-component event-data]
  (validate-event-structure event-data "server-event")
  (let [registry (:connection-registry server-component)
        timestamp (System/nanoTime)]
    (doseq [[_ {:keys [event-queue drain]}] @registry]
      (offer-with-drop-oldest event-queue
                              (make-event-message event-data timestamp))
      (when drain (drain)))))
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
