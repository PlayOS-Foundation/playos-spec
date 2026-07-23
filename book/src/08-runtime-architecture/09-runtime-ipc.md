# Runtime IPC

Runtime services communicate over a message-passing **Inter-Process
Communication** (IPC) bus. The IPC protocol is internal to the Runtime
— applications use the Platform API, not IPC directly.

## Transport

The IPC transport is an implementation detail. The reference
implementation uses Unix domain sockets on Linux and named pipes on
Windows. Alternative transports (D-Bus, custom shared memory) may be
used as long as they satisfy the protocol requirements.

## Protocol

Messages are serialized with [Cap'n Proto](https://capnproto.org/).
The schema is versioned separately from the Runtime and follows
semantic versioning.

### Message structure

```capnp
struct RuntimeMessage {
  sender    @0 :ServiceId;
  recipient @1 :ServiceId;
  messageId @2 :UInt64;
  timestamp @3 :Int64;
  union {
    request  @4 :Request;
    response @5 :Response;
    event    @6 :Event;
  }
}
```

### Message types

- **Request/Response** — synchronous or asynchronous RPC between
  services.
- **Event** — fire-and-forget broadcast to interested services.

## Service registration

At startup, each service registers with the Security Monitor:

1. Service connects to the IPC socket.
2. Service sends a `RegisterService` message with its identity and
   capabilities.
3. Security Monitor validates the service identity (code signing).
4. Service is added to the routing table.

## Routing

The IPC bus routes messages based on `recipient` field. Broadcast
messages (events) are delivered to all registered services. The bus
does not interpret message payloads.

## Security

- Services authenticate on connection (code signing verification).
- Messages are isolated per-sender (no spoofing).
- The IPC bus is not exposed to applications.
- All IPC is local — no network IPC.

## Application communication

Applications **do not** use the Runtime IPC bus directly. They use the
Platform API, which internally translates to IPC messages when
appropriate. For example, `Cloud::SaveGame()` translates to a
Cloud Sync service request internally.

## Related

- [Runtime Services](08-runtime-services.md)
- [Security Model](17-security-model.md)
