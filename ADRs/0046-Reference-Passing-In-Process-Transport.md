# ADR-0046: Reference-Passing In-Process Transport

## Status

Proposed

## Table of Contents

- [Context](#context)
  - [The Conversion Cost Problem](#the-conversion-cost-problem)
  - [What InProcessTransport Already Optimises](#what-inprocesstransport-already-optimises)
  - [What Remains Unoptimised](#what-remains-unoptimised)
  - [The Paintlist Transfer Imperative](#the-paintlist-transfer-imperative)
- [Decision](#decision)
  - [Reference-Passing Marshallers](#reference-passing-marshallers)
  - [Clojure Wire Marshallers](#clojure-wire-marshallers)
  - [Transport-Aware Service Definition](#transport-aware-service-definition)
  - [Transport-Aware Client Calls](#transport-aware-client-calls)
  - [Statistics Adaptation](#statistics-adaptation)
- [Architecture](#architecture)
  - [Current Data Flow (9 Conversion Steps per Round Trip)](#current-data-flow-9-conversion-steps-per-round-trip)
  - [Reference-Passing Data Flow (Zero Conversion Steps)](#reference-passing-data-flow-zero-conversion-steps)
  - [Network Data Flow (Unchanged)](#network-data-flow-unchanged)
  - [Marshaller Interface Contract](#marshaller-interface-contract)
  - [Thread Safety Guarantee](#thread-safety-guarantee)
- [Rationale](#rationale)
  - [Why Not Direct Function Dispatch](#why-not-direct-function-dispatch)
  - [Why Custom Marshallers Rather Than Custom Transport](#why-custom-marshallers-rather-than-custom-transport)
  - [Relationship to ADR-0019](#relationship-to-adr-0019)
- [Consequences](#consequences)
- [Implementation Scope](#implementation-scope)
- [Notes](#notes)
- [Related ADRs](#related-adrs)

---

## Context

### The Conversion Cost Problem

Ooloi's gRPC architecture ([ADR-0002](0002-gRPC.md)) uses a unified `OoloiValue` protobuf message ([ADR-0018](0018-API-gRPC-Interface-and-Events.md)) that can represent any Clojure data type with perfect fidelity. This requires a three-layer conversion pipeline:

1. **Context-free layer** (`clojure_conversion.clj`): Clojure values <-> internal maps
2. **Transport layer** (`protobuf_bridge.clj`): internal maps <-> Java protobuf objects (Builders, `.build()`, `.getFoo()`)
3. **Wire layer**: Java protobuf objects <-> bytes (handled by gRPC marshallers)

For every unary API call, this pipeline executes **9 conversion steps**:

**Server-side (5 steps):**
1. `protobuf-object->internal-map` on request params
2. `proto->clj` to produce native Clojure values
3. Execute API function (the actual work)
4. `clj->proto` on the result
5. `internal-map->protobuf-object` to build the response protobuf

**Client-side (4 steps):**
6. `clj->protobuf-object` to build the request protobuf
7. Construct `OoloiRequest` via Java Builder
8. Destructure `OoloiResponse` via Java accessors
9. `protobuf-object->clj` to recover native Clojure values

Each step traverses the entire data structure. For deeply nested Clojure data (defrecords containing vectors of maps of keywords), every field crosses the Clojure/Java interop boundary.

### What InProcessTransport Already Optimises

ADR-0019 established `InProcessTransport` for combined deployments, eliminating TCP/IP overhead. Investigation reveals that gRPC-Java goes further: its `ProtoInputStream` mechanism (in place since 2015) **already avoids byte serialization** for in-process protobuf messages. The `stream()` method wraps the Java protobuf object without encoding; `parse()` detects the wrapper via `instanceof` and extracts the object by reference. Layer 3 (wire) is already zero-cost.

### What Remains Unoptimised

Layers 1 and 2 — the Clojure <-> Java protobuf object conversion — execute fully on every call regardless of transport mode. For the combined deployment where client and server share a JVM and all data structures are immutable Clojure persistent values, this conversion provides no architectural value. It transforms Clojure values into protobuf objects, passes them by reference (thanks to `ProtoInputStream`), and transforms them back into the same Clojure values.

**Measured cost** (instrument library, 80 KiB equivalent payload):
- Cold start: 400ms total (JIT compilation of conversion code + gRPC framework init)
- Warm (after 1000 calls): 18ms total, of which 6.74ms is server handler time
- Client-side conversion overhead: ~11ms warm per call, proportional to data size

### The Paintlist Transfer Imperative

ADR-0038 specifies backend-authoritative rendering with paintlists containing 30,000+ objects for complex orchestral scores. Viewport paintlists are estimated at 50-150 KiB per fetch. At current conversion rates (~0.14ms/KiB warm), a 500 KiB paintlist costs ~70ms in conversion alone — consuming nearly half the 150ms p95 latency target for visible edits. During initial score loading with parallel paintlist fetches, the aggregate cost is prohibitive.

Reference passing reduces this to effectively zero.

## Decision

We introduce **transport-specific marshallers** that eliminate the Clojure <-> protobuf conversion for in-process transport while preserving the full conversion pipeline for network transport.

### Reference-Passing Marshallers

For in-process transport, a `reference-marshaller` implements `MethodDescriptor.Marshaller<Object>`:

- **`stream(value)`**: wraps the Clojure value in a `ReferenceInputStream` — a thin `InputStream` subclass that holds an object reference. No conversion occurs.
- **`parse(stream)`**: detects `ReferenceInputStream` via `instanceof` and extracts the value. No conversion occurs.

`ReferenceInputStream.read()` throws `UnsupportedOperationException`. This is never called — `InProcessTransport` passes the `InputStream` object by reference without reading bytes. If mistakenly used with a network transport, the failure is immediate and unambiguous.

This follows the identical design pattern as gRPC-Java's own `ProtoInputStream`, applied at the Clojure value level rather than the protobuf message level.

### Clojure Wire Marshallers

For network transport, a `clojure-wire-marshaller` implements `MethodDescriptor.Marshaller<Object>`:

- **`stream(value)`**: performs the full `clj->proto -> internal-map->protobuf-object -> toByteArray()` pipeline
- **`parse(stream)`**: performs the full `parseFrom(bytes) -> protobuf-object->internal-map -> proto->clj` pipeline

This encapsulates the existing three-layer conversion into the marshaller, removing conversion responsibility from handler and client code.

### Transport-Aware Service Definition

The `ServerServiceDefinition` is built manually (not via generated `OoloiServiceImplBase.bindService()`) with `MethodDescriptor<Object, Object>` instances carrying the transport-appropriate marshallers.

Service handlers for `ExecuteMethod` and `ExecuteBatch` receive and return **Clojure data**:
- `ExecuteMethod` handler receives `{:method "..." :params <clojure-value>}`, returns `{:success true :result <clojure-value>}`
- `ExecuteBatch` handler receives a stream of such maps, returns a single result map

`RegisterClient` remains on the protobuf marshaller path — event streaming is infrequent with small payloads, and the conversion cost is negligible. It can be migrated to reference passing later without architectural change; the same marshaller interface applies.

For the two payload-significant methods, the handler implementation is identical regardless of transport. Only the marshaller differs.

### Transport-Aware Client Calls

Client code uses `ClientCalls/blockingUnaryCall` (and equivalent async calls) with `MethodDescriptor<Object, Object>` instances carrying the transport-appropriate marshallers, replacing generated stub classes (`OoloiServiceGrpc/newBlockingStub`).

For `ExecuteMethod` and `ExecuteBatch`, client code sends and receives **Clojure data** in both transport modes. `RegisterClient` continues to use protobuf stubs.

### Statistics Adaptation

Byte-size statistics (`.getSerializedSize()`) are meaningful only for network transport. For in-process:
- Call count and duration continue to be tracked
- Byte-size fields report zero (there are no bytes)
- The statistics snapshot includes a transport-mode indicator

## Architecture

### Current Data Flow (9 Conversion Steps per Round Trip)

```
Client Clojure value
  → clj->proto (step 6)
  → internal-map->protobuf-object (step 7: Java Builders, .build())
  → [ProtoInputStream wraps protobuf object — no bytes]
  → [InProcessTransport passes reference]
  → [ProtoInputStream unwraps — no bytes]
  → protobuf-object->internal-map (step 1: .hasFoo(), .getFoo())
  → proto->clj (step 2)
  → API function executes (step 3)
  → clj->proto (step 4)
  → internal-map->protobuf-object (step 5: Java Builders, .build())
  → [ProtoInputStream wraps — no bytes]
  → [InProcessTransport passes reference]
  → [ProtoInputStream unwraps — no bytes]
  → protobuf-object->internal-map (step 8: .hasFoo(), .getFoo())
  → proto->clj (step 9)
Client Clojure value
```

### Reference-Passing Data Flow (Zero Conversion Steps)

```
Client Clojure value
  → ReferenceInputStream wraps Clojure value
  → InProcessTransport passes reference
  → ReferenceInputStream unwraps Clojure value
  → API function executes
  → ReferenceInputStream wraps Clojure result
  → InProcessTransport passes reference
  → ReferenceInputStream unwraps Clojure result
Client Clojure value
```

No data structure traversal. No Java interop calls. No object allocation beyond the thin wrapper.

### Network Data Flow (Unchanged)

```
Client Clojure value
  → clojure-wire-marshaller.stream(): clj->proto → protobuf → bytes
  → Network transport: bytes over TCP/TLS
  → clojure-wire-marshaller.parse(): bytes → protobuf → proto->clj
  → API function executes
  → clojure-wire-marshaller.stream(): clj->proto → protobuf → bytes
  → Network transport: bytes over TCP/TLS
  → clojure-wire-marshaller.parse(): bytes → protobuf → proto->clj
Client Clojure value
```

### Marshaller Interface Contract

Both marshallers implement `MethodDescriptor.Marshaller<Object>` with this contract:

- The `Object` is always a Clojure value (map, vector, keyword, defrecord, etc.)
- `stream()` and `parse()` are inverse operations: `parse(stream(x))` returns a value equal to `x`
- For reference marshallers: `parse(stream(x))` returns the **identical** object (`identical?` is true)
- For wire marshallers: `parse(stream(x))` returns an **equal** object (`=` is true)

### Thread Safety Guarantee

Clojure persistent data structures are immutable by construction. Passing them by reference across gRPC's executor threads is safe without synchronisation. This is the same guarantee that makes STM ([ADR-0004](0004-STM-for-concurrency.md)) and the single-authority state model ([ADR-0040](0040-Single-Authority-State-Model.md)) possible. Serialisation has never been the mechanism enforcing the rendering boundary ([ADR-0038](0038-Backend-Authoritative-Rendering-and-Terminal-Frontend-Execution.md)) — immutability and API design are.

## Rationale

### Why Not Direct Function Dispatch

Bypassing gRPC entirely for in-process would create two transport mechanisms, two interceptor chains, two exception-handling paths, and two test surfaces. The interceptors (authentication validation, connection lifecycle tracking, statistics collection) would need reimplementation outside gRPC, creating divergence risk.

The marshaller approach changes **only** the serialisation format while preserving gRPC's full infrastructure: interceptor chain, `Context` propagation, deadline handling, status codes, streaming, health checking, and error translation. This is exactly the right seam to vary.

### Why Custom Marshallers Rather Than Custom Transport

gRPC-Java's transport layer (`InProcessTransport`) is already correct — it passes `InputStream` references without byte copying. The problem is not in the transport but in the marshaller endpoints: the code that converts between application types and the `InputStream` that the transport carries. Replacing marshallers addresses the actual cost without touching the transport infrastructure.

### Relationship to ADR-0019

ADR-0019 eliminated TCP/IP overhead by switching to `InProcessTransport`. This ADR eliminates the remaining conversion overhead by switching to reference-passing marshallers. Together, they make in-process gRPC calls nearly as fast as direct function calls while preserving the full gRPC programming model.

| Layer | ADR-0019 | ADR-0046 |
|-------|----------|----------|
| TCP/IP stack | Eliminated | N/A |
| Socket buffers | Eliminated | N/A |
| Byte serialisation | Already bypassed (ProtoInputStream) | N/A |
| Protobuf Java object construction | Present | **Eliminated** |
| Clojure <-> internal map conversion | Present | **Eliminated** |
| gRPC framework (interceptors, context) | Preserved | Preserved |

## Consequences

### Positive

- **Near-zero conversion overhead** for in-process API calls — estimated reduction from ~18ms warm to ~7ms warm for the instrument library payload
- **Cold-start improvement** — eliminates JIT compilation of conversion code, reducing first-call latency significantly
- **Paintlist transfer readiness** — enables the <150ms p95 target for viewport paintlist fetches at orchestral scale
- **Architectural consistency** — same gRPC programming model, same interceptors, same error handling, same streaming
- **Handler simplification** — service handlers work exclusively with Clojure data, eliminating protobuf-specific code from handler logic
- **Test clarity** — tests can assert object identity for in-process responses, confirming zero-copy behaviour

### Negative

- **Generated stub classes no longer used** — client code uses `ClientCalls` directly instead of `OoloiServiceGrpc/newBlockingStub`. The generated class remains readable as documentation: canonical method signatures and `fullMethodName` strings are discoverable there even though the class is no longer called directly.
- **Manual `ServerServiceDefinition` construction** — the generated `bindService()` is replaced with explicit definition building
- **Two marshaller implementations to maintain** — reference and wire, though both are straightforward
- **Transport mismatch risk** — connecting a reference-marshaller client to a network server would fail. Mitigated: `InProcessChannelBuilder` and `InProcessServerBuilder` are paired by construction; cross-connection is structurally impossible

### Mitigations

- Generated code remains in the build for the network path's protobuf handling
- Integration tests exercise both transport paths
- `ReferenceInputStream.read()` fails fast if accidentally used over network transport
- The shared `transport.clj` namespace ensures client and server use matching marshallers for a given transport mode

## Implementation Scope

The optimisation targets **`ExecuteMethod` (unary) and `ExecuteBatch` (client streaming)** — the two methods that carry payload-significant data.

**`RegisterClient` (server streaming for events) is deferred.** Event streaming is infrequent and carries simple, small data structures. The conversion overhead on events is negligible. The event path can remain on the protobuf marshaller indefinitely or be migrated later without architectural change — the same marshaller interface applies.

### Implementation Phases

1. **Transport abstraction** — `ReferenceInputStream`, reference marshaller, wire marshaller, method descriptor factories. New namespace in `shared/`. Pure additions, no existing code changes.
2. **Service definition builder** — manual `ServerServiceDefinition` construction with Clojure-native handlers. New namespace in `shared/`. Parallel to existing generated service, not replacing it yet.
3. **Server integration** — wire `start-grpc-server` to use the new service definition. Transport type selects marshallers. Existing `execute-unified-method` refactored to separate handler logic from protobuf conversion.
4. **Client integration** — replace generated stubs with `ClientCalls` using transport-appropriate descriptors. `execute-method` in `api_client.clj` sends/receives Clojure data directly.
5. **Statistics adaptation** — guard `.getSerializedSize()` calls, add transport-mode indicator.
6. **Test infrastructure** — update test macros, add object-identity assertions for in-process, verify network path unchanged.

Each phase can be committed and tested independently. Phases 1-2 are pure additions with zero risk to existing functionality.

## Notes

**`.getSerializedSize()` is a correctness issue, not merely a meaningless metric.** The current code calls `.getSerializedSize()` on every request, response, and event for statistics collection. With reference-passing marshallers, the objects flowing through the handler are Clojure maps, not protobuf messages. Calling `.getSerializedSize()` on a Clojure map produces a `ClassCastException`. These calls must be guarded by transport mode or removed for the reference path. This is not optional — it is a correctness requirement.

**`ExecuteBatch` STM boundary is unchanged.** The batch handler accumulates streamed request maps and wraps execution in a `dosync` block. With reference passing, each streamed message is a Clojure map rather than a protobuf message, but the accumulation and STM wrapping logic is identical. The marshaller handles per-message conversion; the handler sees the same data shape regardless of transport.

**The generated `OoloiServiceGrpc` class remains in the build.** It continues to serve three purposes: (1) the wire marshaller uses its protobuf message classes for network transport, (2) it defines the canonical `fullMethodName` strings that method descriptors must match, and (3) existing tests that exercise the network path continue to function.

**The `.proto` file is unchanged.** The protobuf schema remains the network wire format contract. Reference passing is a transport optimisation that does not alter the schema or the network protocol. A network client connecting to a network server sees no difference.

## Related ADRs

- [ADR-0001: Frontend-Backend Separation](0001-Frontend-Backend-Separation.md) — three-deployment architecture requiring transport flexibility
- [ADR-0002: gRPC](0002-gRPC.md) — communication protocol foundation
- [ADR-0018: API-gRPC Interface and Events](0018-API-gRPC-Interface-and-Events.md) — unified `OoloiValue` message and dynamic method resolution
- [ADR-0019: In-Process gRPC Transport Optimization](0019-In-Process-gRPC-Transport-Optimization.md) — eliminated TCP/IP overhead; this ADR eliminates conversion overhead
- [ADR-0038: Backend-Authoritative Rendering](0038-Backend-Authoritative-Rendering-and-Terminal-Frontend-Execution.md) — paintlist transfer performance requirements driving this optimisation
- [ADR-0040: Single-Authority State Model](0040-Single-Authority-State-Model.md) — immutability guarantee that makes reference passing safe
