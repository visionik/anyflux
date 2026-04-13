# AnyFlux Specification

**Version:** 0.1.0 (Draft)
**Date:** April 2026
**Title:** AnyFlux — A Lightweight, Cross-Platform, Cross-Language Filter Graph / Flow-Based Programming Library

---

## 1. Introduction

AnyFlux is a **filter graph** and **flow-based programming (FBP)** library designed for maximum portability and usability.

### Goals

1. **Easy to use** — Fluent APIs, JSON graphs, optional visual editing via FBP Protocol.
2. **Easy to implement** — Minimal, consistent core concepts across languages.
3. **Fast** — Zero-copy native typed paths; hybrid push/pull execution.
4. **Skinny** — Small footprint (<100 KB core where possible), no heavy dependencies, embedded-friendly.
5. **Extendable** — Dynamic component registration, plugins, hierarchical subgraphs, custom schedulers.

### Supported Platforms

- Desktop: Windows, macOS, Linux
- Embedded: FreeRTOS, Zephyr (and other RTOS)
- Languages: Native implementations in C/C++, Python, TypeScript/JavaScript, Go, PHP, and others via FFI.

### Design Philosophy

- **Native-first**: Each language has a fully idiomatic API while sharing identical core concepts.
- **Dual data paths**: Typed ports for performance + [`AVar`](https://github.com/visionik/anyvar) ([AnyVar](https://github.com/visionik/anyvar)) for cross-language interop.
- **FBP Protocol compatible**: Optional thin layer for visual editors (Flowhub) and testing (fbp-spec).
- **Separate concerns**: Execution core vs. control/visual layer.

### Architecture Overview

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#909090', 'secondaryColor': '#808080', 'tertiaryColor': '#707070', 'primaryTextColor': '#000000', 'secondaryTextColor': '#000000', 'tertiaryTextColor': '#000000', 'noteTextColor': '#000000', 'lineColor': '#404040', 'actorLineColor': '#404040', 'signalColor': '#404040' }}}%%
graph TB
    subgraph "User Layer"
        FA[Fluent API]
        JG[JSON Graph]
        FP[FBP Protocol<br/>Flowhub / Visual Editors]
    end

    subgraph "Core Layer"
        GR[Graph / Network]
        CO[Components / Nodes]
        CN[Connections / Edges]
        SC[Scheduler / Engine]
    end

    subgraph "Data Layer"
        TP[Typed Packets<br/>zero-copy, native]
        VP[AVar Packets<br/>cross-language, dynamic]
    end

    subgraph "Platform Layer"
        DT[Desktop<br/>Thread pool + lock-free queues]
        EM[Embedded<br/>Cooperative / RTOS tasks]
    end

    FA --> GR
    JG --> GR
    FP --> GR
    GR --> CO
    GR --> CN
    CO --> SC
    CN --> SC
    SC --> TP
    SC --> VP
    SC --> DT
    SC --> EM
```

---

## 2. Core Concepts

### Concept Relationships

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#909090', 'secondaryColor': '#808080', 'tertiaryColor': '#707070', 'primaryTextColor': '#000000', 'secondaryTextColor': '#000000', 'tertiaryTextColor': '#000000', 'noteTextColor': '#000000', 'lineColor': '#404040' }}}%%
erDiagram
    GRAPH ||--o{ COMPONENT : contains
    GRAPH ||--o{ CONNECTION : contains
    COMPONENT ||--|{ INPORT : has
    COMPONENT ||--|{ OUTPORT : has
    CONNECTION }o--|| OUTPORT : "from"
    CONNECTION }o--|| INPORT : "to"
    CONNECTION ||--o| BUFFER : uses
    COMPONENT ||--o| SUBGRAPH : "can be"
    SCHEDULER ||--|| GRAPH : drives
    PACKET }o--|| CONNECTION : "flows through"
```

### 2.1 Packet / Value

The unit of data that flows through the graph.

- **Typed Packet** (`Packet<T>` or native type): For same-language, high-performance paths (zero-copy where possible).
- **AVar Packet** ([AnyVar](https://github.com/visionik/anyvar)): Cross-language / dynamic data.

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#909090', 'secondaryColor': '#808080', 'tertiaryColor': '#707070', 'primaryTextColor': '#000000', 'secondaryTextColor': '#000000', 'tertiaryTextColor': '#000000', 'noteTextColor': '#000000', 'lineColor': '#404040' }}}%%
graph LR
    subgraph "AVar Types  github.com/visionik/anyvar"
        V["AVar<br/><i>32-bit type/flag field</i>"]
        V --> N[null]
        V --> B[bool]
        V --> I["int64 / uint64"]
        V --> I2["int32 / uint32"]
        V --> D["double / float32"]
        V --> S[string]
        V --> BN[binary]
        V --> A[array]
        V --> M[map]
    end

    subgraph "Type Flags"
        F1["A_IS_BACKEND"]
        F2["A_IS_NUMERIC"]
        F3["A_IS_INTEGER"]
        F4["A_IS_BUFFER"]
        F5["A_IS_CONTAINER"]
    end

    subgraph "Backends"
        NB["Native (default)"]
        CB["CBOR"]
        JB["JSON"]
        XB["Custom ABackend"]
    end

    subgraph "Ownership"
        AH["AVarAllocator<br/>hooks"]
        MV[Move / copy]
    end

    V -.-> F1
    V -.-> F2
    V -.-> F3
    V -.-> F4
    V -.-> F5
    V -.-> NB
    V -.-> CB
    V -.-> JB
    V -.-> XB
    V -.-> AH
    V -.-> MV
```

**AnyFlux uses [AnyVar](https://github.com/visionik/anyvar) (`AVar`) as its canonical cross-language variant type.**

`AVar` is a C11 tagged-union with a 32-bit `AVarType` type/flag field. Zero-initialise and use — no setup required. Backend dispatch is an optional extension via `ABackend`; the native path has zero overhead. All operations return `AVAR_OK` or a negative `AVAR_ERR_*` code.

```c
#include "anyvar.h"

/* Scalars — zero-init, no setup */
AVar v = {0};
aVar_setI64(&v, 42);                        /* AVAR_OK or AVAR_ERR_*      */
aVar_setDouble(&v, 3.14);
aVar_setString(&v, "hello", /*copy=*/false); /* borrowed — no malloc       */
aVar_setString(&v, "hello", /*copy=*/true);  /* owned — one malloc         */
aVar_clear(&v);

/* Type inspection */
if (aVar_type(&v) == A_INT64) { ... }
if (A_IS_NUMERIC(aVar_type(&v)))  { ... }   /* int32/64, uint32/64, float  */
if (A_IS_CONTAINER(aVar_type(&v))) { ... }  /* array or map               */

/* Arrays */
AVar arr = {0};
aVar_arrayInit(&arr, /*capacity=*/4);
AVar item = {0};
aVar_setI32(&item, 7);
aVar_arrayPush(&arr, &item);
aVar_clear(&arr);

/* Backend dispatch (advanced — opt-in only) */
AVar cbor = {0};
aVar_convert(&v, &AVAR_CBOR_BACKEND, &cbor);
aVar_clear(&cbor);
```

**Supported value kinds:** `A_NULL` · `A_BOOL` · `A_INT64` · `A_UINT64` · `A_INT32` · `A_UINT32` · `A_DOUBLE` · `A_FLOAT32` · `A_STRING` · `A_BINARY` · `A_ARRAY` · `A_MAP`

**Type flag macros:** `A_IS_BACKEND` · `A_IS_NUMERIC` · `A_IS_INTEGER` · `A_IS_FLOAT` · `A_IS_BUFFER` · `A_IS_CONTAINER` · `A_IS_UNSIGNED` · `A_IS_32BIT_INT`

**Error codes:** `AVAR_OK` · `AVAR_ERR_TYPE` · `AVAR_ERR_NULL` · `AVAR_ERR_OOM` · `AVAR_ERR_BOUNDS` · `AVAR_ERR_KEY` · `AVAR_ERR_BACKEND` · `AVAR_ERR_ARGS`

> See [AnyVar](https://github.com/visionik/anyvar) (`include/anyvar.h`) for the full API, `ABackend` vtable, `AVarAllocator` hooks, and ownership rules.
> **Typed packets:** `AVar` is required only at ABI boundaries (FFI, wire protocol, observability callbacks). Within a single language/process on the hot path, components pass native types directly via shared `userdata` — zero conversion, zero overhead.

### 2.2 Ports

- **InPort** and **OutPort** — named (e.g., `"data"`, `"error"`).
- Components can have **any number** of named inports and outports.
- Support array ports for fan-in/fan-out (e.g., `out[N]`).
- Typed or `AVar` mode.
- Pull-based interface (inspired by RAMEN) for embedded/low-overhead use: `Puller<T>` / `Pullable<T>`.
- Unconnected outports are a silent no-op; optional inports simply never fire.

### 2.3 Component / Node

A black-box reusable processing unit.

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#909090', 'secondaryColor': '#808080', 'tertiaryColor': '#707070', 'primaryTextColor': '#000000', 'secondaryTextColor': '#000000', 'tertiaryTextColor': '#000000', 'noteTextColor': '#000000', 'lineColor': '#404040', 'stateLabelColor': '#000000', 'compositeBackground': '#a0a0a0' }}}%%
stateDiagram-v2
    [*] --> Created
    Created --> Setup : setup()
    Setup --> Ready : success
    Setup --> Error : failure
    Ready --> Processing : process()
    Processing --> Ready : packet handled
    Processing --> Error : failure
    Ready --> TearingDown : teardown()
    TearingDown --> [*] : cleanup complete
    Error --> TearingDown : teardown()
```

**Defined by:**

- Input ports + Output ports (with metadata: type hints, description) — any number of each.
- Processing logic (callback, method, or lambda). The callback receives the **name of the inport that fired**, enabling multi-input dispatch.
- Lifecycle: `setup()`, `process()`, `teardown()`.
- Registration: Static (compile-time) or dynamic (`register("myfilter", MyFilter)`).
- Hierarchical: A whole subgraph can act as a Component.

### 2.4 Connection / Edge

- Directed link from an `OutPort` to an `InPort`.
- Bounded buffer with back-pressure.
- Optional direct (synchronous) pass-through for low-latency paths.

### 2.5 Graph / Network

- Container for Components + Connections.
- Supports:
  - Fluent builder API.
  - JSON serialization (standard FBP graph format).
  - Hierarchical subgraphs (see §2.5.1).
  - Introspection and runtime modification (add/remove nodes/edges while running where supported).

#### 2.5.1 Subgraphs

A subgraph is a complete graph that acts as a component in a parent graph. Its internal
components and connections are invisible to the parent — it is addressed only through its
exposed ports.

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#909090', 'secondaryColor': '#808080', 'tertiaryColor': '#707070', 'primaryTextColor': '#000000', 'secondaryTextColor': '#000000', 'tertiaryTextColor': '#000000', 'noteTextColor': '#000000', 'lineColor': '#404040' }}}%%
graph LR
    subgraph Parent
        SRC[Source]
        subgraph "NormClamp — subgraph"
            NORM[Normalise] --> CLAMP[Clamp]
        end
        SINK[Sink]
        ESINK[ErrorSink]
    end
    SRC -->|"value → in_value"| NORM
    SRC -->|"scale → in_scale"| NORM
    CLAMP -->|"out → out_result"| SINK
    CLAMP -->|"over → out_overflow"| ESINK
```

- Subgraphs can have **multiple exposed inports and outports** — any number.
- `setup`/`teardown` propagate recursively.
- Back-pressure flows through exposed port boundaries as normal.
- Nesting is unlimited.

### 2.6 Scheduler / Engine

- Drives execution.
- Pluggable implementations.
- Hybrid push + pull model.
- Control methods: `start()`, `stop()`, `step()` (for testing), `run()` (blocking).

---

## 3. Data Flow

### Push vs Pull Execution

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#909090', 'secondaryColor': '#808080', 'tertiaryColor': '#707070', 'primaryTextColor': '#000000', 'secondaryTextColor': '#000000', 'tertiaryTextColor': '#000000', 'noteTextColor': '#000000', 'lineColor': '#404040', 'actorBkg': '#808080', 'actorTextColor': '#000000', 'noteBkgColor': '#909090' }}}%%
sequenceDiagram
    participant S as Source
    participant C as Connection<br/>(Bounded Buffer)
    participant F as Filter
    participant O as Output

    Note over S,O: Push Model (Desktop Default)
    S->>C: emit(packet)
    C->>F: deliver(packet)
    F->>F: process()
    F->>O: emit(result)

    Note over S,O: Pull Model (Embedded)
    O->>F: pull()
    F->>C: pull()
    C->>S: pull()
    S-->>C: packet
    C-->>F: packet
    F->>F: process()
    F-->>O: result
```

### Back-Pressure Flow

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#909090', 'secondaryColor': '#808080', 'tertiaryColor': '#707070', 'primaryTextColor': '#000000', 'secondaryTextColor': '#000000', 'tertiaryTextColor': '#000000', 'noteTextColor': '#000000', 'lineColor': '#404040', 'actorBkg': '#808080', 'actorTextColor': '#000000', 'noteBkgColor': '#909090' }}}%%
sequenceDiagram
    participant P as Producer
    participant B as Buffer<br/>(capacity: N)
    participant C as Consumer

    P->>B: emit(pkt1)
    P->>B: emit(pkt2)
    P->>B: emit(pkt3)
    Note over B: Buffer full!
    P->>B: emit(pkt4)
    B-->>P: BLOCK (back-pressure)
    C->>B: consume(pkt1)
    Note over B: Slot freed
    B-->>P: UNBLOCK
    P->>B: emit(pkt4)
```

---

## 4. Graph JSON Format

AnyFlux uses the standard FBP JSON graph format for interoperability.

```json
{
  "properties": { "name": "My Audio Pipeline" },
  "processes": {
    "source": { "component": "core/Generator" },
    "filter": { "component": "myproject/LowPass" }
  },
  "connections": [
    {
      "src": { "process": "source", "port": "out" },
      "tgt": { "process": "filter", "port": "in" }
    }
  ],
  "inports": {},
  "outports": {}
}
```

> Supports IIPs (Initial Information Packets), groups, metadata, subgraphs.

### Example Pipeline Visualization

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#909090', 'secondaryColor': '#808080', 'tertiaryColor': '#707070', 'primaryTextColor': '#000000', 'secondaryTextColor': '#000000', 'tertiaryTextColor': '#000000', 'noteTextColor': '#000000', 'lineColor': '#404040' }}}%%
graph LR
    GEN["core/Generator<br/><i>source</i>"] -->|out → in| LP["myproject/LowPass<br/><i>filter</i>"]
    LP -->|out → in| AMP["myproject/Amplifier<br/><i>gain</i>"]
    AMP -->|out → in| OUT["core/Output<br/><i>sink</i>"]
```

---

## 5. FBP Protocol Support

AnyFlux includes an optional, thin adapter for the FBP Network Protocol.

### Protocol Interaction

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#909090', 'secondaryColor': '#808080', 'tertiaryColor': '#707070', 'primaryTextColor': '#000000', 'secondaryTextColor': '#000000', 'tertiaryTextColor': '#000000', 'noteTextColor': '#000000', 'lineColor': '#404040', 'actorBkg': '#808080', 'actorTextColor': '#000000', 'noteBkgColor': '#909090' }}}%%
sequenceDiagram
    participant UI as Visual Editor<br/>(Flowhub)
    participant RT as AnyFlux Runtime
    participant GR as Graph Engine

    UI->>RT: runtime:getruntime
    RT-->>UI: capabilities, version

    UI->>RT: component:list
    RT-->>UI: available components + port metadata

    UI->>RT: graph:addnode (source, core/Generator)
    RT->>GR: add component
    GR-->>RT: ok
    RT-->>UI: ack

    UI->>RT: graph:addedge (source.out → filter.in)
    RT->>GR: connect
    GR-->>RT: ok
    RT-->>UI: ack

    UI->>RT: network:start
    RT->>GR: start()
    GR-->>RT: started
    RT-->>UI: network:started

    loop Live Monitoring
        GR->>RT: data event
        RT-->>UI: network:data
    end
```

### Minimal Required Sub-Protocols

| Sub-Protocol | Purpose |
|---|---|
| `runtime` | Capabilities, version, runtime metadata |
| `graph` | Full CRUD on nodes, edges, IIPs, groups, subgraphs |
| `component` | List available components with port metadata |
| `network` | Start/stop, status, live data events, errors |

### Transport

- WebSocket (primary for desktop/browser)
- Serial/UART, custom channels for embedded
- Pluggable

### Capabilities Example

```json
{
  "protocol": "runtime",
  "command": "runtime",
  "payload": {
    "type": "anyflux",
    "version": "0.1.0",
    "capabilities": [
      "protocol:graph",
      "protocol:network",
      "protocol:component",
      "network:data"
    ]
  }
}
```

### Required Test Components (Core Collection)

| Component | Behavior |
|---|---|
| `Repeat` | Forwards input unchanged |
| `Drop` | Discards input |
| `Output` | Sends data to runtime outport or console |

> **Implementation Note:** The protocol handler is not in the hot data path. Internal typed/Variant packet passing remains fast and zero-copy. Protocol only handles control, editing, and optional live monitoring.

---

## 6. API

The full C ABI-compatible API is documented in [`docs/api.md`](./docs/api.md) (concepts,
mermaid diagrams, and examples) and [`docs/api-reference.md`](./docs/api-reference.md)
(complete function and struct reference).

## 6a. API Sketches (Idiomatic per Language)

### C++ (Header-only friendly)

```cpp
auto g = anyflux::Graph::create();
g->add<Adder>("adder");
g->connect("source.out", "adder.in");
g->start();
```

### Python

```python
g = anyflux.Graph()
g.add("myfilters.Adder", id="adder")
g.connect("source.out", "adder.in")
g.start()
```

### TypeScript

```typescript
const g = new anyflux.Graph();
g.add("adder", "myfilters/Adder");
g.connect("source.out", "adder.in");
await g.start();
```

---

## 7. Execution Models

### Scheduler Strategy by Platform

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#909090', 'secondaryColor': '#808080', 'tertiaryColor': '#707070', 'primaryTextColor': '#000000', 'secondaryTextColor': '#000000', 'tertiaryTextColor': '#000000', 'noteTextColor': '#000000', 'lineColor': '#404040' }}}%%
graph TB
    subgraph "Desktop Scheduler"
        TP[Thread Pool]
        LF[Lock-Free Queues]
        AP[Auto-Parallelize<br/>Independent Branches]
        TP --- LF
        LF --- AP
    end

    subgraph "Embedded Scheduler"
        ST[Single-Threaded<br/>Cooperative]
        TQ[RTOS Task/Queue<br/>Mapping]
        PL[Pull-Based<br/>Execution]
        ST --- TQ
        TQ --- PL
    end

    SC[Scheduler Interface] --> TP
    SC --> ST
```

| Model | Description | Best For |
|---|---|---|
| **Push** | Data-driven (default for most desktop use) | Streaming, event processing |
| **Pull** | On-demand | Embedded, DSP, control loops |
| **Hybrid** | Push with pull-back for throttling | Mixed workloads |

- Automatic branch synchronization at merge points (inspired by DSPatch).
- Back-pressure via bounded connections.

---

## 8. Extendability Features

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#909090', 'secondaryColor': '#808080', 'tertiaryColor': '#707070', 'primaryTextColor': '#000000', 'secondaryTextColor': '#000000', 'tertiaryTextColor': '#000000', 'noteTextColor': '#000000', 'lineColor': '#404040' }}}%%
mindmap
    root((AnyFlux<br/>Extendability))
        Components
            Dynamic Registry
            Static Registration
            Plugin System
                Shared Libraries desktop
                Static Registration embedded
        Graphs
            Subgraph Composition
            Runtime Modification
            Hierarchical Nesting
        Schedulers
            Custom Implementations
            Platform-Specific
        Introspection
            Port Metadata
            Type Hints
            Live Monitoring
```

---

## 9. Embedded Considerations

- Compile-time flags to disable threads, exceptions, RTTI, heap usage.
- Static allocation pools for packets and nodes.
- Lightweight JSON parser for protocol (e.g., jsmn).
- Pull-based execution to minimize queues and context switches.
- `AVar` compile-time flags for embedded targets (defined in [AnyVar](https://github.com/visionik/anyvar)):

| CMake option | Effect |
|---|---|
| `AVAR_NO_HEAP` | Disable heap allocation paths |
| `AVAR_PACKED` | Request packed `AVar` layout |
| `AVAR_CUSTOM_ALLOC` | Enable `AVarAllocator` custom allocation hooks |
| `AVAR_NO_THREAD_SAFE` | Disable internal mutex |
| `AVAR_NO_MAP` | Disable map support (saves code size on tiny targets) |

### Desktop vs Embedded Feature Matrix

| Feature | Desktop | Embedded |
|---|---|---|
| Threading | Thread pool | Single-thread / RTOS tasks |
| Allocation | Heap (default) | Static pools |
| JSON parser | Full (nlohmann, etc.) | Minimal (jsmn) |
| Exceptions | Yes | Optional (compile flag) |
| RTTI | Yes | Optional (compile flag) |
| `AVar` ownership | Refcount | Move-only or static |
| Protocol transport | WebSocket | Serial/UART |
| Graph modification | Runtime | Compile-time preferred |

---

## 10. Implementation Roadmap

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#909090', 'secondaryColor': '#808080', 'tertiaryColor': '#707070', 'primaryTextColor': '#000000', 'secondaryTextColor': '#000000', 'tertiaryTextColor': '#000000', 'noteTextColor': '#000000', 'lineColor': '#404040' }}}%%
gantt
    title AnyFlux Implementation Phases
    dateFormat YYYY-MM
    axisFormat %b %Y

    section Phase 1: Core
    Packet/AVar + Ports             :p1a, 2026-05, 4w
    Component + Graph               :p1b, after p1a, 3w
    Single-threaded Scheduler       :p1c, after p1b, 2w

    section Phase 2: Serialization
    JSON graph load/save            :p2a, after p1c, 2w
    Fluent builder APIs             :p2b, after p1c, 2w

    section Phase 3: Execution
    Typed + AVar dual paths         :p3a, after p2a, 3w
    Back-pressure                   :p3b, after p3a, 2w

    section Phase 4: Schedulers
    Desktop thread pool             :p4a, after p3b, 3w
    Embedded cooperative            :p4b, after p3b, 3w

    section Phase 5: Protocol
    FBP Protocol adapter            :p5a, after p4a, 3w
    Test components                 :p5b, after p5a, 1w

    section Phase 6: Polish
    Live monitoring                 :p6a, after p5b, 2w
    Hierarchical graphs             :p6b, after p5b, 3w
    Dynamic registration            :p6c, after p5b, 2w

    section Phase 7: Bindings
    Language bindings               :p7a, after p6a, 6w
    Platform optimizations          :p7b, after p6a, 4w
```

### Phase Summary

| Phase | Deliverables | Dependencies |
|---|---|---|
| **1. Core** | Packet/[AVar](https://github.com/visionik/anyvar), Ports, Component, Graph, single-threaded Scheduler | None |
| **2. Serialization** | JSON graph load/save, fluent builders | Phase 1 |
| **3. Execution** | Typed + AVar execution paths, back-pressure | Phase 2 |
| **4. Schedulers** | Pluggable schedulers (desktop vs embedded) | Phase 3 |
| **5. Protocol** | Minimal FBP Protocol adapter + test components | Phase 4 |
| **6. Polish** | Live monitoring, hierarchical graphs, dynamic registration | Phase 5 |
| **7. Bindings** | Language bindings and platform-specific optimizations | Phase 6 |

---

## 11. Non-Goals (for Skinny Design)

- Full distributed computing framework (use higher-level tools on top).
- Heavy visual editor built-in (rely on Flowhub via protocol).
- Built-in persistence or complex orchestration.

---

## 12. License & Open Source

AnyFlux is intended to be open source (**MIT/Apache 2.0** recommended) to encourage community components and bindings.

---

> This specification defines the shared foundation. Each language implementation must support the core concepts while providing native, idiomatic APIs. The FBP Protocol layer is optional but strongly recommended for visual tooling and ecosystem compatibility.
>
> **Feedback and contributions welcome.** Next steps: formalize `AVar` serialization (see [AnyVar](https://github.com/visionik/anyvar)), define exact component metadata schema, and prototype the C++ core.

---

## 13. Related Projects

- **[AnyVar](https://github.com/visionik/anyvar)** — The canonical cross-language C11 variant type (`AVar`) used as AnyFlux's variant packet type. Phase 1 (C core) is implemented. `AVar` is a tagged union with a 32-bit `AVarType` field supporting 12 value kinds (null, bool, int32/64, uint32/64, double, float32, string, binary, array, map), type flag macros, optional `ABackend` dispatch, `AVarAllocator` hooks, and CMake-configurable embedded build variants.
