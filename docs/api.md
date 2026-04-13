# AnyFlux C API

**Version:** 0.1.0 (Draft)
**ABI:** C11, `extern "C"` safe, FFI-compatible across C, C++, Python, Go, TypeScript, and others.

All objects are opaque handles. All functions return `AFStatus`. Packets cross all boundaries as
`AVar` (see [AnyVar](https://github.com/visionik/anyvar)). Typed packets exist only within a
single-language implementation and never cross the ABI.

---

## 1. Conventions

### Naming

| Element | Convention | Example |
|---|---|---|
| Opaque handle types | `AF` prefix, PascalCase | `AFGraph`, `AFComponent` |
| Functions | `aFlux_` prefix, camelCase | `aFlux_graphCreate` |
| Constants / macros | `AF_` prefix, UPPER_SNAKE | `AF_OK`, `AF_ERR_OOM` |
| Callbacks | `AF*Fn` suffix | `AFProcessFn`, `AFSetupFn` |
| Descriptor structs | `AF*Desc` suffix | `AFComponentDesc` |
| Vtable structs | `AF*Vtable` suffix | `AFSchedulerVtable` |

### Error Handling

All functions return `AFStatus` (`int32_t`). Zero is success; negative values are errors.

```c
typedef int32_t AFStatus;

#define AF_OK                0
#define AF_ERR_NULL         -1   /* required pointer argument was NULL        */
#define AF_ERR_OOM          -2   /* allocation failed                         */
#define AF_ERR_TYPE         -3   /* wrong AVar type for operation             */
#define AF_ERR_INVALID      -4   /* invalid argument or state                 */
#define AF_ERR_NOT_FOUND    -5   /* component/port/key not found              */
#define AF_ERR_BOUNDS       -6   /* index out of range                        */
#define AF_ERR_RUNNING      -7   /* operation not allowed while graph runs    */
#define AF_ERR_STOPPED      -8   /* operation not allowed while graph stopped */
#define AF_ERR_FULL         -9   /* connection buffer full (back-pressure)    */
#define AF_ERR_EMPTY        -10  /* port has no data (pull model)             */
#define AF_ERR_DUPLICATE    -11  /* component id or connection already exists */
#define AF_ERR_BACKEND      -12  /* backend dispatch error                    */
```

### Ownership

- Handles returned by `*Create` functions are owned by the caller; must be released with
  the matching `*Destroy`.
- Strings passed to API functions are **borrowed** — the caller retains ownership; AnyFlux
  copies what it needs internally.
- `AVar` packets passed to emit/send functions are **borrowed** — AnyFlux copies internally
  if it needs to retain the value.
- `AVar` packets received via callbacks are **borrowed** — valid only for the duration of
  the callback.

### Header Guard

```c
#ifdef __cplusplus
extern "C" {
#endif

/* ... all declarations ... */

#ifdef __cplusplus
}
#endif
```

---

## 2. Core Types

```c
#include "anyvar.h"   /* AVar, AVarType, AVAR_OK, etc. */

typedef int32_t  AFStatus;
typedef uint32_t AFFlags;

/* Opaque handles */
typedef struct AFGraph_*       AFGraph;
typedef struct AFComponent_*   AFComponent;
typedef struct AFPort_*        AFPort;
typedef struct AFConnection_*  AFConnection;
typedef struct AFScheduler_*   AFScheduler;
typedef struct AFRuntime_*     AFRuntime;     /* FBP Protocol runtime */

/* Port direction */
typedef enum AFPortDir {
    AF_PORT_IN  = 0,
    AF_PORT_OUT = 1
} AFPortDir;

/* Graph run state */
typedef enum AFGraphState {
    AF_GRAPH_STOPPED  = 0,
    AF_GRAPH_RUNNING  = 1,
    AF_GRAPH_PAUSED   = 2,
    AF_GRAPH_ERROR    = 3
} AFGraphState;
```

---

## 3. Graph API

A graph is the top-level container for components and connections.

```c
/* Lifecycle */
AFGraph   aFlux_graphCreate(void);
AFGraph   aFlux_graphCreateNamed(const char* name);
void      aFlux_graphDestroy(AFGraph g);

/* Components */
AFStatus  aFlux_graphAddComponent(AFGraph g, const char* id, AFComponent comp);
AFStatus  aFlux_graphRemoveComponent(AFGraph g, const char* id);
AFComponent aFlux_graphGetComponent(AFGraph g, const char* id);
AFStatus  aFlux_graphComponentCount(AFGraph g, uint32_t* out_count);

/* Connections */
AFStatus  aFlux_graphConnect(AFGraph g,
                              const char* src_id, const char* src_port,
                              const char* dst_id, const char* dst_port);
AFStatus  aFlux_graphConnectBuffered(AFGraph g,
                                      const char* src_id, const char* src_port,
                                      const char* dst_id, const char* dst_port,
                                      uint32_t capacity);
AFStatus  aFlux_graphDisconnect(AFGraph g,
                                 const char* src_id, const char* src_port,
                                 const char* dst_id, const char* dst_port);
AFStatus  aFlux_graphConnectionCount(AFGraph g, uint32_t* out_count);

/* Initial Information Packets (IIPs) */
AFStatus  aFlux_graphSetIIP(AFGraph g,
                             const char* dst_id, const char* dst_port,
                             const AVar* value);
AFStatus  aFlux_graphClearIIP(AFGraph g,
                               const char* dst_id, const char* dst_port);

/* Subgraph exposed ports */
AFStatus  aFlux_graphExposeInport(AFGraph g, const char* name,
                                   const char* comp_id, const char* port);
AFStatus  aFlux_graphExposeOutport(AFGraph g, const char* name,
                                    const char* comp_id, const char* port);

/* State */
AFGraphState aFlux_graphState(AFGraph g);
AFStatus     aFlux_graphGetName(AFGraph g, const char** out_name);
```

---

## 4. Component API

### Descriptor

Components are defined via a flat descriptor struct — no vtable inheritance, no C++ required.

```c
/* Lifecycle callbacks */
typedef AFStatus (*AFSetupFn)   (AFComponent comp, void* userdata);
typedef AFStatus (*AFProcessFn) (AFComponent comp,
                                  const char* inport,
                                  const AVar* packet,
                                  void*       userdata);
typedef void     (*AFTeardownFn)(AFComponent comp, void* userdata);

/* Port metadata (used by component:list in FBP Protocol) */
typedef struct AFPortMeta {
    const char* name;
    const char* description;  /* human-readable, may be NULL */
    const char* type_hint;    /* e.g. "int64", "string", "any" — may be NULL */
    bool        required;
} AFPortMeta;

/* Component descriptor — all fields are borrowed; copy what you need */
typedef struct AFComponentDesc {
    const char*        type_name;      /* e.g. "core/Adder"                  */
    const char*        description;    /* human-readable, may be NULL         */

    uint32_t           inport_count;
    const AFPortMeta*  inports;

    uint32_t           outport_count;
    const AFPortMeta*  outports;

    AFSetupFn          setup;          /* called before graph starts, may be NULL */
    AFProcessFn        process;        /* called when packet arrives on inport    */
    AFTeardownFn       teardown;       /* called after graph stops, may be NULL   */

    void*              userdata;       /* opaque state passed to all callbacks    */
} AFComponentDesc;
```

### Instance API

```c
/* Create / destroy */
AFComponent  aFlux_componentCreate(const AFComponentDesc* desc);
void         aFlux_componentDestroy(AFComponent comp);

/* Port handles (valid for lifetime of component) */
AFPort       aFlux_componentGetPort(AFComponent comp, const char* name, AFPortDir dir);

/* Emit a packet on an outport — called from within AFProcessFn */
AFStatus     aFlux_componentEmit(AFComponent comp,
                                  const char* outport,
                                  const AVar* packet);

/* Userdata access */
void*        aFlux_componentUserdata(AFComponent comp);
AFStatus     aFlux_componentSetUserdata(AFComponent comp, void* userdata);

/* Metadata */
AFStatus     aFlux_componentGetDesc(AFComponent comp, const AFComponentDesc** out_desc);
```

### Component Registry

```c
/* Register a reusable component type by name */
AFStatus     aFlux_registerComponent(const char* type_name,
                                      const AFComponentDesc* desc);
AFStatus     aFlux_unregisterComponent(const char* type_name);

/* Create an instance from a registered type name */
AFComponent  aFlux_createComponentByType(const char* type_name);

/* Enumerate registered types */
AFStatus     aFlux_registeredTypeCount(uint32_t* out_count);
AFStatus     aFlux_registeredTypeName(uint32_t index, const char** out_name);
```

---

## 5. Port API

```c
/* Identity */
const char*  aFlux_portName(AFPort port);
AFPortDir    aFlux_portDir(AFPort port);
bool         aFlux_portIsConnected(AFPort port);

/* Push model: send from an outport (usually via aFlux_componentEmit) */
AFStatus     aFlux_portSend(AFPort port, const AVar* packet);

/* Pull model: request data from an inport (embedded / cooperative scheduler) */
AFStatus     aFlux_portReceive(AFPort port, AVar* out_packet);
bool         aFlux_portHasData(AFPort port);

/* Connection buffer state */
AFStatus     aFlux_portBufferLen(AFPort port, uint32_t* out_len);
AFStatus     aFlux_portBufferCapacity(AFPort port, uint32_t* out_capacity);
```

---

## 6. Packet / AVar Integration

AnyFlux packets are `AVar` values from [AnyVar](https://github.com/visionik/anyvar).
No wrapper type is introduced; `AVar` is the packet type at all ABI boundaries.

```c
/*
 * Typed packets — within one language/process, components may pass native
 * types directly via the process callback without boxing into AVar.
 * At any ABI boundary (FFI, FBP protocol wire, CBOR/JSON serialization),
 * values must be represented as AVar.
 *
 * Convenience: AnyFlux provides no additional packet wrapper.
 * Use AnyVar's API directly:
 *
 *   AVar pkt = {0};
 *   aVar_setI64(&pkt, 42);
 *   aFlux_componentEmit(comp, "out", &pkt);
 *   aVar_clear(&pkt);
 *
 * AnyFlux copies the AVar internally when needed; the caller always
 * clears its own local copy.
 */
```

---

## 7. Scheduler / Execution API

### Built-in Schedulers

```c
/* Thread-pool scheduler (desktop default) */
AFScheduler  aFlux_schedulerCreateThreadPool(uint32_t thread_count);

/* Single-threaded scheduler (testing / simple pipelines) */
AFScheduler  aFlux_schedulerCreateSingleThread(void);

/* Cooperative scheduler (embedded / RTOS tasks) */
AFScheduler  aFlux_schedulerCreateCooperative(void);

void         aFlux_schedulerDestroy(AFScheduler sched);
```

### Custom Scheduler Vtable

```c
typedef struct AFSchedulerVtable {
    /* Start driving the graph; non-blocking, returns immediately */
    AFStatus (*start)  (AFScheduler sched, AFGraph g);

    /* Stop all processing; blocks until in-flight work drains */
    AFStatus (*stop)   (AFScheduler sched);

    /* Advance one step — required for cooperative / test schedulers */
    AFStatus (*step)   (AFScheduler sched);

    /* Called by aFlux_schedulerDestroy */
    void     (*destroy)(AFScheduler sched);
} AFSchedulerVtable;

AFScheduler  aFlux_schedulerCreateCustom(const AFSchedulerVtable* vtable,
                                          void* userdata);
void*        aFlux_schedulerUserdata(AFScheduler sched);
```

### Execution Control

```c
/* Blocking: setup → start → run until complete or error → teardown */
AFStatus     aFlux_run(AFGraph g, AFScheduler sched);

/* Non-blocking: setup → start → return */
AFStatus     aFlux_start(AFGraph g, AFScheduler sched);

/* Signal stop; drains in-flight packets before teardown */
AFStatus     aFlux_stop(AFGraph g, AFScheduler sched);

/* Single step (cooperative / test scheduler only) */
AFStatus     aFlux_step(AFGraph g, AFScheduler sched);

/* Block until graph reaches AF_GRAPH_STOPPED */
AFStatus     aFlux_wait(AFGraph g);

/* State query */
AFGraphState aFlux_graphState(AFGraph g);
bool         aFlux_isRunning(AFGraph g);
```

---

## 8. Serialization API

### JSON Graph Format

Uses the standard [FBP graph JSON format](https://github.com/flowbased/fbp-manifest).

```c
/* Serialize graph to JSON.
 * Pass buf=NULL, *len=0 to query required buffer size. */
AFStatus     aFlux_graphToJSON(AFGraph g, char* buf, size_t* len);

/* Deserialize graph from JSON.
 * Components referenced by type name must be registered before calling. */
AFStatus     aFlux_graphFromJSON(const char* json, size_t len,
                                  AFGraph* out_graph);
```

### CBOR (via AnyVar CBOR backend)

```c
/* Serialize graph to CBOR.
 * Pass buf=NULL, *len=0 to query required buffer size. */
AFStatus     aFlux_graphToCBOR(AFGraph g, uint8_t* buf, size_t* len);
AFStatus     aFlux_graphFromCBOR(const uint8_t* buf, size_t len,
                                  AFGraph* out_graph);
```

### Packet Serialization (delegate to AnyVar)

```c
/*
 * AnyFlux does not define separate packet serialization functions.
 * Use AnyVar directly:
 *
 *   aVar_convert(&pkt, &AVAR_CBOR_BACKEND, &cbor_pkt);
 *   cbor_pkt.u.backend.vtable->encode_cbor(&cbor_pkt, buf, &len);
 */
```

---

## 9. FBP Protocol / Runtime API

Optional thin adapter implementing the
[FBP Network Protocol](https://flowbased.github.io/fbp-protocol/).
Not in the hot data path — only handles control, editing, and live monitoring.

### Transport Vtable (pluggable)

```c
typedef struct AFTransportVtable {
    /* Send a JSON message to the connected client */
    AFStatus (*send) (void* ctx, const char* json, size_t len);

    /* Called when the runtime is destroyed */
    void     (*close)(void* ctx);
} AFTransportVtable;
```

### Runtime Lifecycle

```c
AFRuntime    aFlux_runtimeCreate(const char* runtime_type,
                                  const char* version);
void         aFlux_runtimeDestroy(AFRuntime rt);

/* Bind a transport (WebSocket, serial, etc.) */
AFStatus     aFlux_runtimeBindTransport(AFRuntime rt,
                                         const AFTransportVtable* vtable,
                                         void* ctx);

/* Feed an incoming JSON message from the transport */
AFStatus     aFlux_runtimeHandleMessage(AFRuntime rt,
                                         const char* json,
                                         size_t len);

/* Bind a graph + scheduler to drive via protocol commands */
AFStatus     aFlux_runtimeBindGraph(AFRuntime rt,
                                     AFGraph g,
                                     AFScheduler sched);
```

### Capability Flags

```c
#define AF_CAP_PROTOCOL_GRAPH     (1u << 0)  /* graph CRUD                 */
#define AF_CAP_PROTOCOL_NETWORK   (1u << 1)  /* start/stop/status          */
#define AF_CAP_PROTOCOL_COMPONENT (1u << 2)  /* component:list             */
#define AF_CAP_NETWORK_DATA       (1u << 3)  /* live packet monitoring     */
#define AF_CAP_NETWORK_CONTROL    (1u << 4)  /* step/pause                 */

AFStatus     aFlux_runtimeSetCapabilities(AFRuntime rt, AFFlags caps);
AFFlags      aFlux_runtimeCapabilities(AFRuntime rt);
```

---

## 10. Observability

### Packet Observer

```c
/* Called for every packet emitted on any outport in the graph.
 * Valid only for the duration of the callback — copy if you need to retain. */
typedef void (*AFPacketObserverFn)(const char* comp_id,
                                    const char* port,
                                    const AVar* packet,
                                    void*       userdata);

AFStatus     aFlux_setPacketObserver(AFGraph g,
                                      AFPacketObserverFn fn,
                                      void* userdata);
AFStatus     aFlux_clearPacketObserver(AFGraph g);
```

### Error Observer

```c
typedef void (*AFErrorObserverFn)(const char* comp_id,
                                   AFStatus    err,
                                   const char* message,
                                   void*       userdata);

AFStatus     aFlux_setErrorObserver(AFGraph g,
                                     AFErrorObserverFn fn,
                                     void* userdata);
AFStatus     aFlux_clearErrorObserver(AFGraph g);
```

### State Change Observer

```c
typedef void (*AFStateObserverFn)(AFGraph      g,
                                   AFGraphState state,
                                   void*        userdata);

AFStatus     aFlux_setStateObserver(AFGraph g,
                                     AFStateObserverFn fn,
                                     void* userdata);
```

---

## 11. Language Bindings Support

### Version

```c
const char*  aFlux_version(void);
uint32_t     aFlux_versionMajor(void);
uint32_t     aFlux_versionMinor(void);
uint32_t     aFlux_versionPatch(void);
```

### Binding Conventions

Each language binding MUST provide idiomatic equivalents for:

| C API concept | Idiomatic target |
|---|---|
| `AFGraph` handle + `aFlux_graphCreate/Destroy` | Class with constructor/destructor or context manager |
| `AFComponentDesc` + callbacks | Class with `setup`, `process`, `teardown` methods |
| `AFProcessFn` callback + `aFlux_componentEmit` | Method that receives packet, returns/yields packets |
| `AVar` packet | Language-native dynamic type (dict/object/map), converted at boundary |
| `AFStatus` error code | Exception or Result/Either type |
| `aFlux_run` / `aFlux_start` | Blocking call / async task |

### Minimal Binding Interface (Python example shape)

```python
import anyflux

g = anyflux.Graph()
g.add("adder", "core/Adder")
g.connect("source.out", "adder.in")
g.start()          # non-blocking
g.wait()
```

### Minimal Binding Interface (TypeScript example shape)

```typescript
const g = new anyflux.Graph();
g.add("adder", "core/Adder");
g.connect("source.out", "adder.in");
await g.run();
```

### Minimal Binding Interface (Go example shape)

```go
g := anyflux.NewGraph()
g.Add("adder", "core/Adder")
g.Connect("source.out", "adder.in")
g.Run()
```

---

## 12. Built-in Core Components

Every AnyFlux implementation MUST provide:

| Type name | Inports | Outports | Behaviour |
|---|---|---|---|
| `core/Repeat` | `in` | `out` | Forwards packet unchanged |
| `core/Drop` | `in` | — | Discards packet |
| `core/Output` | `in` | — | Emits to runtime outport or stdout |
| `core/Split` | `in` | `out[N]` | Fans packet out to all connected outports |
| `core/Merge` | `in[N]` | `out` | Forwards from any connected inport |

---

## 13. Embedded / Compile-Time Options

| CMake option | Preprocessor flag | Effect |
|---|---|---|
| `AF_NO_HEAP` | `AF_NO_HEAP` | Disable heap; use static allocation pools |
| `AF_NO_THREADS` | `AF_NO_THREADS` | Disable thread-pool scheduler and mutex |
| `AF_NO_PROTOCOL` | `AF_NO_PROTOCOL` | Exclude FBP Protocol adapter |
| `AF_NO_SERIALIZATION` | `AF_NO_SERIALIZATION` | Exclude JSON/CBOR graph serialization |
| `AF_NO_MAP` | `AF_NO_MAP` | Disable map support in packets |
| `AF_STATIC_COMPONENTS` | `AF_STATIC_COMPONENTS` | Compile-time only component registry |
| `AF_MAX_COMPONENTS` | `AF_MAX_COMPONENTS=N` | Static component registry size |
| `AF_MAX_CONNECTIONS` | `AF_MAX_CONNECTIONS=N` | Static connection pool size |

---

## 14. Example: Minimal Pipeline (C)

```c
#include "anyflux.h"
#include "anyvar.h"

/* --- Generator component --- */

static AFStatus gen_process(AFComponent comp,
                             const char* inport,
                             const AVar* packet,
                             void* userdata)
{
    (void)inport; (void)packet; (void)userdata;
    AVar out = {0};
    aVar_setI64(&out, 42);
    AFStatus s = aFlux_componentEmit(comp, "out", &out);
    aVar_clear(&out);
    return s;
}

/* --- Printer component --- */

static AFStatus print_process(AFComponent comp,
                               const char* inport,
                               const AVar* packet,
                               void* userdata)
{
    (void)comp; (void)inport; (void)userdata;
    if (aVar_type(packet) == A_INT64)
        printf("received: %lld\n", (long long)aVar_asI64(packet));
    return AF_OK;
}

/* --- Wire up and run --- */

int main(void)
{
    static const AFPortMeta gen_out[]   = {{ "out",  NULL, "int64", false }};
    static const AFPortMeta print_in[]  = {{ "in",   NULL, "any",   true  }};

    AFComponentDesc gen_desc = {
        .type_name     = "example/Gen",
        .inport_count  = 0, .inports  = NULL,
        .outport_count = 1, .outports = gen_out,
        .process       = gen_process,
    };
    AFComponentDesc print_desc = {
        .type_name     = "example/Print",
        .inport_count  = 1, .inports  = print_in,
        .outport_count = 0, .outports = NULL,
        .process       = print_process,
    };

    AFComponent gen   = aFlux_componentCreate(&gen_desc);
    AFComponent print = aFlux_componentCreate(&print_desc);

    AFGraph g = aFlux_graphCreate();
    aFlux_graphAddComponent(g, "gen",   gen);
    aFlux_graphAddComponent(g, "print", print);
    aFlux_graphConnect(g, "gen", "out", "print", "in");

    AFScheduler sched = aFlux_schedulerCreateSingleThread();
    aFlux_run(g, sched);            /* blocking */

    aFlux_schedulerDestroy(sched);
    aFlux_graphDestroy(g);
    return 0;
}
```
