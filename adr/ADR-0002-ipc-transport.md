# ADR-0002 — Unix Socket IPC Transport

**Date:** Sprint 1  
**Status:** Accepted  
**Deciders:** PlayOS core team

---

## Context

`playos-init` needs a way to receive commands from trusted clients (shell, overlay) and send events back. Options considered: D-Bus, Varlink, gRPC, plain Unix sockets, netlink.

## Decision

Use Unix domain sockets with a simple length-prefixed binary frame and JSON-encoded message bodies. See [runtime-ipc.md](../runtime-ipc.md) for the full protocol.

## Rationale

- **No daemon dependency:** No D-Bus daemon, no session bus, no activation — `playos-init` is the only IPC server needed
- **Minimal dependencies:** A Unix socket needs only the kernel — no extra libraries required in the initramfs
- **Access control:** UNIX group permissions (`playos-trusted`) enforce that only trusted clients can connect — no capability negotiation needed
- **Simplicity:** Length-prefix + JSON is readable during debugging, easy to implement in C, and easy to test with `socat` or a simple Python script
- **Sufficient performance:** IPC volume is low (a few messages per user action) — protocol overhead is irrelevant

## Alternatives Considered

| Option | Rejected because |
|---|---|
| D-Bus | Requires `dbus-daemon`; adds systemd/activation complexity; not suitable for PID 1 |
| Varlink | Good fit but less widely known; minimal tooling advantage |
| gRPC | Heavy dependency (protobuf, HTTP/2); overkill for this use case |
| Netlink | Kernel-space complexity; not suited for user-space control commands |

## Consequences

- A custom framing protocol must be maintained
- JSON adds a parsing dependency (use a small embedded parser like `cJSON` or `yyjson`)
- Binary framing must be carefully tested for partial reads
