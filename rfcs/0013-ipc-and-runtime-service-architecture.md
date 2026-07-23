---
rfc: 0013
title: IPC and Runtime Service Architecture
status: Draft
authors: []
created: 2026-07-22
---

# RFC-0013: IPC and Runtime Service Architecture

> **Status:** Draft

## Summary

This RFC defines the Inter-Process Communication (IPC) protocol and
Runtime Service architecture that form the internal backbone of the
PlayOS Runtime.  It specifies the Cap'n Proto message schema, service
registration and authentication, message routing, service lifecycle
(including failure recovery), and the Security Monitor's role as the
IPC gatekeeper.  It is the normative foundation for the [Runtime
Services][rt-svc] and [Runtime IPC][rt-ipc] chapters.

[rt-svc]: ../book/src/08-runtime-architecture/08-runtime-services.md
[rt-ipc]: ../book/src/08-runtime-architecture/09-runtime-ipc.md

## Motivation

The Runtime is not a monolith — it is composed of independent services
(Application Manager, Input Router, Cloud Sync, etc.) that must
communicate reliably and securely.  Without a formal IPC specification:

- Services would use ad-hoc communication (pipes, shared memory,
  sockets) with no common message format.
- Service discovery would be implicit and fragile.
- Authenticating services to each other would be inconsistent.
- Failure recovery would be undefined — a crashed service could take
  down the entire Runtime.

A formal IPC RFC lets each service be implemented independently while
guaranteeing interoperability.

## Guide-Level Explanation

Think of the Runtime as a small cluster of programs running on one
device.  They talk to each other over a local message bus — no network,
no cloud, just on-device communication.

Each service (Application Manager, Input Router, Cloud Sync, etc.)
connects to the bus at startup, proves its identity to the Security
Monitor, and then sends and receives messages.  Applications never use
the bus directly — they use the Platform API, which internally
translates to bus messages when needed.

If a service crashes, the Security Monitor restarts it automatically.
The Shell keeps running; the player never notices.

## Reference-Level Explanation

### 1. Transport Layer

The IPC transport is an implementation detail, but the reference
implementation uses:

| Platform | Transport |
|---|---|
| Linux | Unix domain sockets (`SOCK_SEQPACKET`) |
| Windows | Named pipes (`PIPE_TYPE_MESSAGE`) |

The transport MUST guarantee:

1. **Message boundaries** — each send corresponds to exactly one
   receive; no message fragmentation or coalescing.
2. **In-order delivery** — messages from sender A to recipient B are
   delivered in the order they were sent.
3. **Reliability** — messages are not lost (transport is
   connection-oriented).
4. **Locality** — the bus MUST NOT be reachable over the network.  All
   IPC is local to the device.

### 2. Message Schema (Cap'n Proto)

All messages are serialized with [Cap'n Proto](https://capnproto.org/).

#### Envelope

Every message is wrapped in an envelope:

```capnp
@0xc1a7f0e8d3b59624;  # PlayOS Runtime IPC protocol ID

struct Envelope {
  sender    @0 :ServiceId;
  recipient @1 :ServiceId;
  messageId @2 :UInt64;
  timestamp @3 :Int64;       # nanoseconds since Runtime epoch
  union {
    request  @4 :Request;
    response @5 :Response;
    event    @6 :Event;
  }
}
```

#### ServiceId

```capnp
struct ServiceId {
  name    @0 :Text;     # e.g., "app-manager", "input-router"
  version @1 :Text;     # semantic version, e.g., "1.0.0"
}
```

#### Request / Response

```capnp
struct Request {
  method @0 :Text;       # e.g., "LaunchApp", "GetInstalledPackages"
  params @1 :Data;       # serialized parameters (opaque to bus)
}

struct Response {
  requestId @2 :UInt64;  # matches the request's messageId
  status   @3 :Status;
  result   @4 :Data;     # serialized result (opaque to bus)
  
  enum Status {
    ok        @0;
    error     @1;
    timeout   @2;
    denied    @3;        # permission denied
    notFound  @4;        # method not found
    invalid   @5;        # invalid parameters
  }
}
```

#### Event

```capnp
struct Event {
  type   @0 :Text;       # e.g., "app.launched", "network.changed"
  payload @1 :Data;      # serialized event data
}
```

#### Opaque payloads

The bus does not interpret `params`, `result`, or `payload`.  These
are service-specific Cap'n Proto messages serialized to bytes.  Each
service publishes its own schema for the methods it handles.

### 3. Service Registration

At startup, each service connects to the IPC socket and registers with
the Security Monitor:

```text
1. Service connects to IPC socket.
2. Service sends RegisterService request.
3. Security Monitor verifies the service's code signature
   (Ed25519, same PKI as package signing — RFC-0009).
4. Security Monitor adds the service to the routing table.
5. Service receives a RegisterService response with its assigned
   routing ID.
6. Service is now live on the bus.
```

#### RegisterService request

```capnp
struct RegisterServiceRequest {
  serviceId @0 :ServiceId;
  methods   @1 :List(Text);   # list of method names this service handles
  events    @2 :List(Text);   # list of event types this service emits
}
```

The Security Monitor MUST reject registration if:
- The service's code signature is invalid or missing.
- The `serviceId.name` collides with an already-registered service.
- The service declares methods or events that conflict with another
  service's registration (method/event namespace collision).

### 4. Message Routing

The bus routes messages based on the `recipient` field:

| recipient | Delivery |
|---|---|
| Specific `ServiceId` | Delivered to that service only |
| `ServiceId{name = "*"}` | Broadcast to all registered services |

The bus MUST NOT deliver a message where:
- The `sender` field does not match the authenticated sender.
- The `recipient` service is not registered.

#### Request-response matching

Requests and responses are matched by `messageId`.  The bus tracks
outstanding requests and enforces:

- A response MUST have the same `messageId` as the request.
- A response MUST come from the service that received the request.
- A response MUST be delivered within a configurable timeout (default:
  30 seconds).  Timed-out requests receive a synthetic `timeout`
  response.

### 5. Security Monitor

The Security Monitor is the first service to start and the gatekeeper
for all IPC:

#### Responsibilities

1. **Service authentication** — verify code signatures at registration.
2. **Routing table** — maintain the mapping of `ServiceId` → socket.
3. **Sender verification** — every message's `sender` field is verified
   against the authenticated identity of the sending socket.
4. **Service lifecycle** — monitor service health, restart failed
   services.
5. **Permission enforcement** — some methods require specific
   permissions; the Security Monitor checks the caller's permissions
   before forwarding the request.

#### Permission model for IPC

Services run with ambient authority (they are part of the Runtime), but
some methods are restricted:

```capnp
struct MethodPermissions {
  method    @0 :Text;
  permission @1 :Text;     # e.g., "system.package-install"
}
```

The Security Monitor consults the permission table before forwarding a
request.  If the caller lacks the required permission, a `denied`
response is returned.

### 6. Service Lifecycle

#### Start-up order

Services MUST start in this order:

1. **Security Monitor** — establishes the IPC socket.
2. **Logging Service** — captures all subsequent output.
3. **Input Router** — ready for input immediately (critical path).
4. **Audio Service** — enumerate devices.
5. **Network Manager** — enumerate interfaces.
6. **Package Installer** — scan installed packages.
7. **Lifecycle Manager** — ready to launch apps.
8. **Application Manager** — ready to manage app lifecycle.
9. **Shell UI** — displayed once everything is ready.
10. **Cloud Sync, Update Service, Overlay Service** — background,
    non-blocking.

The critical path (services 1–9) MUST complete before the Shell is
displayed.  Services 10+ are asynchronous and MUST NOT block the
Shell.

#### Service dependencies

A service MAY declare dependencies on other services:

```capnp
struct ServiceDependency {
  serviceId @0 :ServiceId;
  required  @1 :Bool;      # if true, this service cannot start without it
}
```

If a required dependency is not available, the service fails to start.
The Security Monitor retries after a backoff (1s, 2s, 4s, 8s, max 30s).

#### Failure recovery

If a service crashes:

1. The Security Monitor detects the disconnection (socket close).
2. The Security Monitor removes the service from the routing table.
3. Pending requests to the crashed service receive `timeout` responses.
4. The Security Monitor restarts the service process.
5. The service re-registers.
6. Other services detect the restart via the `service.registered` event.

The Shell and running applications MUST remain responsive during
service restarts.  Application state MUST be preserved.

### 7. Service Contract

Each Runtime Service MUST publish a Cap'n Proto schema defining:

- The methods it handles (request/response pairs).
- The events it emits.
- The permissions required for each method (if any).
- Its dependencies on other services.

Example for the Application Manager:

```capnp
# app-manager.capnp

interface ApplicationManager {
  launchApp      @0 (appId :Text) -> (pid :UInt32);
  terminateApp   @1 (pid :UInt32) -> ();
  listRunning    @2 () -> (apps :List(RunningApp));
  switchToApp    @3 (appId :Text) -> ();
  
  # Events emitted
  # event app.launched  { appId, pid }
  # event app.exited    { appId, pid, exitCode }
  # event app.crashed   { appId, pid, signal }
}

struct RunningApp {
  appId @0 :Text;
  pid   @1 :UInt32;
  state @2 :AppState;
  
  enum AppState {
    launching @0;
    running   @1;
    suspended @2;
    exiting   @3;
  }
}
```

### 8. Application Visibility

Applications MUST NOT have direct access to the IPC bus.  All
application-to-Runtime communication goes through the Platform API,
which internally translates to IPC messages.

However, for debugging, the Platform API MAY expose a diagnostic
interface:

```cpp
// Developer mode only — not available in production
namespace PlayOS::Diagnostics {
    std::vector<ServiceInfo> ListRuntimeServices();
    ServiceStatus GetServiceStatus(const std::string& serviceName);
}
```

This is gated by developer mode and the `system.diagnostics`
capability.

## Drawbacks

- Cap'n Proto adds a build dependency and learning curve.
- The strict service start-up order creates a serial dependency chain
  on the critical path; a slow-starting service delays the Shell.
- Service-per-process architecture increases memory overhead compared
  to a monolithic Runtime.
- The IPC bus is a single point of failure — if the Security Monitor
  crashes, all IPC stops.

## Rationale and Alternatives

- **Cap'n Proto vs. FlatBuffers vs. Protobuf:** Cap'n Proto is chosen
  for zero-copy deserialization (important for input events at 60+ FPS),
  no code generation step at build time (schema is read at startup),
  and Cap'n Proto's RPC capability protocol maps well to the
  request/response model.
- **D-Bus vs. custom IPC:** D-Bus is the standard Linux IPC but ties
  the Runtime to a Linux-specific transport.  Custom IPC with pluggable
  transports supports Windows and future platforms.
- **In-process services (threads) vs. separate processes:** Separate
  processes provide stronger isolation — a crash in Cloud Sync does not
  take down the Input Router.  The memory overhead is acceptable on
  modern hardware (each service is lightweight).
- **Service mesh (e.g., linkerd) vs. simple bus:** Overkill for a
  single-device Runtime.  The Security Monitor provides service
  discovery, authentication, and health checking without the complexity
  of a service mesh.

## Prior Art

- **Android Binder** (`/dev/binder`, AIDL): the closest analogue.
  Android's IPC uses a kernel driver for zero-copy; PlayOS uses
  userspace Cap'n Proto for portability.
- **Chromium Mojo:** Chromium's IPC system uses Cap'n Proto-like
  message passing with interface definitions.  PlayOS adopts a similar
  model with Cap'n Proto as the serialization layer.
- **systemd / D-Bus:** systemd manages service lifecycles; D-Bus
  provides the message bus.  PlayOS combines both roles in the Security
  Monitor.
- **Wayland protocol:** Wayland uses a socket-based protocol with
  interface definitions (XML).  PlayOS IPC is a service-to-service
  protocol, not an application-to-compositor protocol, but follows the
  same architectural pattern.

## Unresolved Questions

- Should the IPC bus support zero-copy shared memory for large payloads
  (e.g., save data blobs)?  Cap'n Proto supports this natively but
  requires OS-specific shared memory setup.
- How are service schemas versioned?  Cap'n Proto's schema evolution
  handles additive changes; breaking changes require a new `ServiceId`
  with a different name.
- Should there be a maximum message size?  Large messages (e.g., game
  assets transferred between services) could block the bus.  A 64 KB
  default limit with an explicit streaming mechanism is proposed.
- How does the bus handle backpressure when a service is slow to
  consume messages?

## Future Possibilities

- **Streaming messages:** For large data transfers (cloud save sync,
  package download progress), a streaming protocol on top of the IPC
  bus with flow control.
- **Service mesh for multi-device:** Extend the IPC bus to support
  inter-device communication (e.g., phone as second screen, local
  multiplayer).
- **Hot-reloading services:** Update a Runtime Service without
  restarting the whole Runtime — swap the service process while
  preserving its state.
- **Introspection and debugging:** A `playos-ipc-monitor` tool that
  connects to the bus and displays live message flow for debugging.
- **Sandboxed third-party services:** Allow device vendors to add
  custom Runtime Services (e.g., RGB lighting control) that run with
  reduced privileges.
