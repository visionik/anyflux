# AnyFlux API Reference

Complete function signatures, parameter descriptions, return values, and struct field details.
For conceptual overview, architecture, and examples see [api.md](./api.md).

---

## Structs and Types

### `AFStatus` — Error Code

```c
typedef int32_t AFStatus;
```

Returned by all functions. `AF_OK (0)` on success; negative on error. See error code table in
[api.md §2](./api.md).

---

### `AFFlags` — Bitmask

```c
typedef uint32_t AFFlags;
```

Used for capability flags (`aFlux_runtimeSetCapabilities`).

---

### `AFPortDir` — Port Direction Enum

```c
typedef enum AFPortDir { AF_PORT_IN = 0, AF_PORT_OUT = 1 } AFPortDir;
```

---

### `AFGraphState` — Graph Run State Enum

```c
typedef enum AFGraphState {
    AF_GRAPH_STOPPED = 0,
    AF_GRAPH_RUNNING = 1,
    AF_GRAPH_PAUSED  = 2,
    AF_GRAPH_ERROR   = 3
} AFGraphState;
```

---

### `AFPortMeta` — Port Metadata

Describes one port for use in `AFComponentDesc` and `component:list` FBP Protocol responses.

```c
typedef struct AFPortMeta {
    const char* name;         /* port name, e.g. "in", "out", "err"        */
    const char* description;  /* human-readable description, may be NULL   */
    const char* type_hint;    /* e.g. "double", "string", "any"; may NULL  */
    bool        required;     /* true = graph is invalid if unconnected     */
} AFPortMeta;
```

All string fields are **borrowed** — AnyFlux copies them internally during
`aFlux_componentCreate`.

---

### `AFComponentDesc` — Component Descriptor

Defines the type, ports, callbacks, and initial state of a component. All fields are
**borrowed** at creation time.

```c
typedef struct AFComponentDesc {
    const char*        type_name;      /* e.g. "core/Adder"; copied internally    */
    const char*        description;    /* human-readable; may be NULL             */

    uint32_t           inport_count;   /* number of entries in inports[]          */
    const AFPortMeta*  inports;        /* array of inport metadata                */

    uint32_t           outport_count;  /* number of entries in outports[]         */
    const AFPortMeta*  outports;       /* array of outport metadata               */

    AFSetupFn          setup;          /* called once before graph starts; NULL ok */
    AFProcessFn        process;        /* called when packet arrives on any inport */
    AFTeardownFn       teardown;       /* called once after graph stops; NULL ok   */

    void*              userdata;       /* passed as-is to all callbacks            */
} AFComponentDesc;
```

---

### `AFSchedulerVtable` — Custom Scheduler Interface

```c
typedef struct AFSchedulerVtable {
    AFStatus (*start)  (AFScheduler sched, AFGraph g);
    AFStatus (*stop)   (AFScheduler sched);
    AFStatus (*step)   (AFScheduler sched);   /* single step; required for cooperative */
    void     (*destroy)(AFScheduler sched);   /* called by aFlux_schedulerDestroy      */
} AFSchedulerVtable;
```

---

### `AFTransportVtable` — FBP Protocol Transport Interface

```c
typedef struct AFTransportVtable {
    AFStatus (*send) (void* ctx, const char* json, size_t len);
    void     (*close)(void* ctx);
} AFTransportVtable;
```

---

### Callback Typedefs

```c
/* Called once before the graph starts. Return AF_OK or an error to abort start. */
typedef AFStatus (*AFSetupFn)(AFComponent comp, void* userdata);

/* Called when a packet arrives on any inport.
 * inport   — name of the inport that received the packet
 * packet   — borrowed; valid only for the duration of this call
 * Return AF_OK on success; any error code triggers the error observer. */
typedef AFStatus (*AFProcessFn)(AFComponent comp, const char* inport,
                                 const AVar* packet, void* userdata);

/* Called once after the graph stops. */
typedef void (*AFTeardownFn)(AFComponent comp, void* userdata);

/* Packet observer — fired for every packet emitted on any outport.
 * packet is borrowed; copy it with aVar_copy() if you need to retain it. */
typedef void (*AFPacketObserverFn)(const char* comp_id, const char* port,
                                    const AVar* packet, void* userdata);

/* Error observer — fired when a component process callback returns non-AF_OK. */
typedef void (*AFErrorObserverFn)(const char* comp_id, AFStatus err,
                                   const char* message, void* userdata);

/* State observer — fired on every graph state transition. */
typedef void (*AFStateObserverFn)(AFGraph g, AFGraphState state, void* userdata);
```

---

## Function Reference

### Graph API

#### `aFlux_graphCreate`
```c
AFGraph aFlux_graphCreate(void);
```
Creates an empty unnamed graph. Returns NULL on OOM.

#### `aFlux_graphCreateNamed`
```c
AFGraph aFlux_graphCreateNamed(const char* name);
```
| Param | Description |
|---|---|
| `name` | Borrowed string; copied internally. Used in serialization and FBP Protocol. |

Returns NULL on OOM.

#### `aFlux_graphDestroy`
```c
void aFlux_graphDestroy(AFGraph g);
```
Destroys the graph. Recursively destroys any subgraphs added as components. All component
handles added to the graph are also destroyed. Safe to call with NULL.

#### `aFlux_graphAddComponent`
```c
AFStatus aFlux_graphAddComponent(AFGraph g, const char* id, AFComponent comp);
```
| Param | Description |
|---|---|
| `g` | Target graph |
| `id` | Unique string id within this graph (e.g. `"filter"`). Borrowed; copied. |
| `comp` | Component handle — graph takes ownership |

Returns `AF_ERR_DUPLICATE` if `id` is already used. Returns `AF_ERR_RUNNING` if the graph is
running (runtime modification — support is implementation-defined).

#### `aFlux_graphRemoveComponent`
```c
AFStatus aFlux_graphRemoveComponent(AFGraph g, const char* id);
```
Removes and destroys the component. Also removes all connections touching its ports.
Returns `AF_ERR_NOT_FOUND` if id is unknown.

#### `aFlux_graphGetComponent`
```c
AFComponent aFlux_graphGetComponent(AFGraph g, const char* id);
```
Returns the component handle or NULL if not found. Caller does **not** own the returned handle.

#### `aFlux_graphConnect`
```c
AFStatus aFlux_graphConnect(AFGraph g,
                              const char* src_id, const char* src_port,
                              const char* dst_id, const char* dst_port);
```
Connects `src_id.src_port → dst_id.dst_port` with the default buffer capacity.
Returns `AF_ERR_NOT_FOUND` if either component or port is unknown.
Returns `AF_ERR_DUPLICATE` if the connection already exists.

#### `aFlux_graphConnectBuffered`
```c
AFStatus aFlux_graphConnectBuffered(AFGraph g,
                                     const char* src_id, const char* src_port,
                                     const char* dst_id, const char* dst_port,
                                     uint32_t capacity);
```
Same as `aFlux_graphConnect` with an explicit `capacity` (number of packets the buffer holds
before back-pressure blocks the sender).

#### `aFlux_graphDisconnect`
```c
AFStatus aFlux_graphDisconnect(AFGraph g,
                                 const char* src_id, const char* src_port,
                                 const char* dst_id, const char* dst_port);
```
Removes the specified connection. Returns `AF_ERR_NOT_FOUND` if it does not exist.

#### `aFlux_graphSetIIP`
```c
AFStatus aFlux_graphSetIIP(AFGraph g,
                             const char* dst_id, const char* dst_port,
                             const AVar* value);
```
Sets an Initial Information Packet. The value is deep-copied internally. The IIP is delivered
to the port as the first packet when the graph starts. Replaces any existing IIP on that port.

#### `aFlux_graphClearIIP`
```c
AFStatus aFlux_graphClearIIP(AFGraph g, const char* dst_id, const char* dst_port);
```
Removes the IIP from a port. Returns `AF_ERR_NOT_FOUND` if no IIP was set.

#### `aFlux_graphExposeInport`
```c
AFStatus aFlux_graphExposeInport(AFGraph g, const char* name,
                                  const char* comp_id, const char* port);
```
Declares that external port `name` on this graph (when used as a subgraph) maps to
`comp_id.port` internally. The mapping is one-to-one.

#### `aFlux_graphExposeOutport`
```c
AFStatus aFlux_graphExposeOutport(AFGraph g, const char* name,
                                   const char* comp_id, const char* port);
```
Same as `aFlux_graphExposeInport` for outports.

#### `aFlux_graphState`
```c
AFGraphState aFlux_graphState(AFGraph g);
```
Returns current state: `AF_GRAPH_STOPPED`, `AF_GRAPH_RUNNING`, `AF_GRAPH_PAUSED`,
or `AF_GRAPH_ERROR`.

---

### Component API

#### `aFlux_componentCreate`
```c
AFComponent aFlux_componentCreate(const AFComponentDesc* desc);
```
Allocates and initialises a component from `desc`. All string fields and port arrays are
copied. Returns NULL on OOM.

#### `aFlux_componentDestroy`
```c
void aFlux_componentDestroy(AFComponent comp);
```
Destroys the component. Do not call if the component has been added to a graph (the graph owns
it). Safe to call with NULL.

#### `aFlux_componentGetPort`
```c
AFPort aFlux_componentGetPort(AFComponent comp, const char* name, AFPortDir dir);
```
Returns a port handle valid for the lifetime of the component, or NULL if not found.

#### `aFlux_componentEmit`
```c
AFStatus aFlux_componentEmit(AFComponent comp, const char* outport, const AVar* packet);
```
Emits `packet` on the named outport. Intended to be called from within `AFProcessFn`.
The packet is **borrowed** — AnyFlux deep-copies it into the connection buffer.
Returns `AF_OK` if the port is unconnected (silent no-op). Returns `AF_ERR_FULL` if
back-pressure is active and the buffer is at capacity.

#### `aFlux_componentUserdata`
```c
void* aFlux_componentUserdata(AFComponent comp);
```
Returns the `userdata` pointer set in the descriptor (or via `aFlux_componentSetUserdata`).

#### `aFlux_componentSetUserdata`
```c
AFStatus aFlux_componentSetUserdata(AFComponent comp, void* userdata);
```
Replaces the userdata pointer. Thread-safe only if the scheduler is not currently invoking
`process` on this component.

#### `aFlux_registerComponent`
```c
AFStatus aFlux_registerComponent(const char* type_name, const AFComponentDesc* desc);
```
Registers a component type in the global registry under `type_name`. Subsequent calls to
`aFlux_createComponentByType(type_name)` use this descriptor as the template.
Returns `AF_ERR_DUPLICATE` if the name is already registered.

#### `aFlux_unregisterComponent`
```c
AFStatus aFlux_unregisterComponent(const char* type_name);
```
Removes a type from the registry. Does not affect already-created instances.

#### `aFlux_createComponentByType`
```c
AFComponent aFlux_createComponentByType(const char* type_name);
```
Creates a new instance of a registered type. Returns NULL if the type is not found or on OOM.

---

### Port API

#### `aFlux_portName`
```c
const char* aFlux_portName(AFPort port);
```
Returns the port's name string. Valid for the lifetime of the owning component.

#### `aFlux_portIsConnected`
```c
bool aFlux_portIsConnected(AFPort port);
```
Returns true if the port has at least one active connection.

#### `aFlux_portSend`
```c
AFStatus aFlux_portSend(AFPort port, const AVar* packet);
```
Pushes a packet into a port's connection buffer. Primarily for use in push-model schedulers.
Prefer `aFlux_componentEmit` from within process callbacks. Returns `AF_ERR_FULL` under
back-pressure.

#### `aFlux_portReceive`
```c
AFStatus aFlux_portReceive(AFPort port, AVar* out_packet);
```
Pulls the next packet from an inport buffer into `*out_packet`. The caller owns the returned
`AVar` and must call `aVar_clear` on it. Returns `AF_ERR_EMPTY` if no data is available.
Intended for cooperative / pull-model schedulers.

#### `aFlux_portHasData`
```c
bool aFlux_portHasData(AFPort port);
```
Non-blocking check. Returns true if at least one packet is queued.

#### `aFlux_portBufferLen`
```c
AFStatus aFlux_portBufferLen(AFPort port, uint32_t* out_len);
```
Writes current queue depth into `*out_len`.

#### `aFlux_portBufferCapacity`
```c
AFStatus aFlux_portBufferCapacity(AFPort port, uint32_t* out_capacity);
```
Writes maximum queue capacity into `*out_capacity`.

---

### Scheduler / Execution API

#### `aFlux_schedulerCreateThreadPool`
```c
AFScheduler aFlux_schedulerCreateThreadPool(uint32_t thread_count);
```
Creates a thread-pool scheduler. `thread_count = 0` uses a hardware-concurrency default.
Returns NULL on OOM or if threading is disabled (`AF_NO_THREADS`).

#### `aFlux_schedulerCreateSingleThread`
```c
AFScheduler aFlux_schedulerCreateSingleThread(void);
```
Single-threaded scheduler. Suitable for testing and simple sequential pipelines.

#### `aFlux_schedulerCreateCooperative`
```c
AFScheduler aFlux_schedulerCreateCooperative(void);
```
Cooperative scheduler for embedded / RTOS use. Processes one pending packet per `aFlux_step`
call. Does not create threads.

#### `aFlux_schedulerCreateCustom`
```c
AFScheduler aFlux_schedulerCreateCustom(const AFSchedulerVtable* vtable, void* userdata);
```
Wraps a custom implementation. `vtable` is borrowed; must remain valid for the scheduler's
lifetime.

#### `aFlux_schedulerDestroy`
```c
void aFlux_schedulerDestroy(AFScheduler sched);
```
Calls `vtable->destroy` and frees the scheduler handle. Safe to call with NULL.

#### `aFlux_run`
```c
AFStatus aFlux_run(AFGraph g, AFScheduler sched);
```
Runs the full lifecycle: `setup` all components → deliver IIPs → start scheduler → block until
`AF_GRAPH_STOPPED` → `teardown` all components. Returns the first non-`AF_OK` status
encountered, or `AF_OK`.

#### `aFlux_start`
```c
AFStatus aFlux_start(AFGraph g, AFScheduler sched);
```
Calls `setup`, delivers IIPs, then starts the scheduler non-blocking. Returns immediately.
Use `aFlux_wait` to block until completion.

#### `aFlux_stop`
```c
AFStatus aFlux_stop(AFGraph g, AFScheduler sched);
```
Signals the scheduler to stop. Drains in-flight packets then calls `teardown` on all
components. May block briefly while draining.

#### `aFlux_step`
```c
AFStatus aFlux_step(AFGraph g, AFScheduler sched);
```
Advances the cooperative scheduler by one packet. No-op for other scheduler types.
Returns `AF_ERR_EMPTY` when no packets are pending (graph has quiesced).

#### `aFlux_wait`
```c
AFStatus aFlux_wait(AFGraph g);
```
Blocks the calling thread until `aFlux_graphState(g) == AF_GRAPH_STOPPED`. Returns
immediately if already stopped.

---

### Serialization API

#### `aFlux_graphToJSON`
```c
AFStatus aFlux_graphToJSON(AFGraph g, char* buf, size_t* len);
```
Serializes the graph to the [FBP JSON format](https://github.com/flowbased/fbp-manifest).
Pass `buf = NULL` and `*len = 0` on a first call to query the required buffer size; then
allocate and call again. On success `*len` is set to the number of bytes written (including
null terminator).

#### `aFlux_graphFromJSON`
```c
AFStatus aFlux_graphFromJSON(const char* json, size_t len, AFGraph* out_graph);
```
Deserializes a graph from FBP JSON. All component type names referenced in the JSON must be
registered in the global registry before calling. Sets `*out_graph` on success; caller owns
the returned graph.

#### `aFlux_graphToCBOR`
```c
AFStatus aFlux_graphToCBOR(AFGraph g, uint8_t* buf, size_t* len);
```
Same size-query pattern as `aFlux_graphToJSON`. Encodes graph structure as CBOR.

#### `aFlux_graphFromCBOR`
```c
AFStatus aFlux_graphFromCBOR(const uint8_t* buf, size_t len, AFGraph* out_graph);
```
Deserializes a graph from CBOR. Same registry precondition as `aFlux_graphFromJSON`.

---

### FBP Protocol / Runtime API

#### `aFlux_runtimeCreate`
```c
AFRuntime aFlux_runtimeCreate(const char* runtime_type, const char* version);
```
| Param | Description |
|---|---|
| `runtime_type` | Reported to clients in `runtime:runtime` response, e.g. `"anyflux"` |
| `version` | Runtime version string, e.g. `"0.1.0"` |

#### `aFlux_runtimeDestroy`
```c
void aFlux_runtimeDestroy(AFRuntime rt);
```
Destroys the runtime and calls `close` on any bound transport.

#### `aFlux_runtimeBindTransport`
```c
AFStatus aFlux_runtimeBindTransport(AFRuntime rt,
                                     const AFTransportVtable* vtable, void* ctx);
```
Attaches a transport implementation. `vtable` is borrowed. Only one transport can be bound
at a time; rebinding replaces the previous one.

#### `aFlux_runtimeHandleMessage`
```c
AFStatus aFlux_runtimeHandleMessage(AFRuntime rt, const char* json, size_t len);
```
Feeds one incoming FBP Protocol JSON message (received from the transport) to the runtime
for dispatch. The runtime calls the bound transport's `send` to respond.

#### `aFlux_runtimeBindGraph`
```c
AFStatus aFlux_runtimeBindGraph(AFRuntime rt, AFGraph g, AFScheduler sched);
```
Binds a graph and scheduler to the runtime. Protocol `network:start` / `network:stop` commands
will call `aFlux_start` / `aFlux_stop` on these.

#### `aFlux_runtimeSetCapabilities`
```c
AFStatus aFlux_runtimeSetCapabilities(AFRuntime rt, AFFlags caps);
```
Declares which sub-protocols are supported. Use the `AF_CAP_*` flag constants.

---

### Observability API

#### `aFlux_setPacketObserver`
```c
AFStatus aFlux_setPacketObserver(AFGraph g, AFPacketObserverFn fn, void* userdata);
```
Installs a callback invoked for every packet emitted on any outport in the graph (including
within subgraphs). The `packet` argument is borrowed — copy with `aVar_copy()` to retain.
Replaces any existing observer. Pass `fn = NULL` to clear.

#### `aFlux_setErrorObserver`
```c
AFStatus aFlux_setErrorObserver(AFGraph g, AFErrorObserverFn fn, void* userdata);
```
Installs a callback invoked whenever a component's `process` returns a non-`AF_OK` status.

#### `aFlux_setStateObserver`
```c
AFStatus aFlux_setStateObserver(AFGraph g, AFStateObserverFn fn, void* userdata);
```
Installs a callback invoked on every graph state transition (stopped → running, running →
stopped, etc.).

---

### Version API

#### `aFlux_version`
```c
const char* aFlux_version(void);
```
Returns the library version string, e.g. `"0.1.0"`. The returned pointer is valid for the
process lifetime.

#### `aFlux_versionMajor` / `aFlux_versionMinor` / `aFlux_versionPatch`
```c
uint32_t aFlux_versionMajor(void);
uint32_t aFlux_versionMinor(void);
uint32_t aFlux_versionPatch(void);
```
Returns the individual version number components as integers.

---

### Capability Flags

```c
#define AF_CAP_PROTOCOL_GRAPH     (1u << 0)  /* graph:* commands            */
#define AF_CAP_PROTOCOL_NETWORK   (1u << 1)  /* network:start/stop/status   */
#define AF_CAP_PROTOCOL_COMPONENT (1u << 2)  /* component:list              */
#define AF_CAP_NETWORK_DATA       (1u << 3)  /* live network:data events    */
#define AF_CAP_NETWORK_CONTROL    (1u << 4)  /* step / pause control        */
```
