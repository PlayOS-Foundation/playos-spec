# Security Model

The PlayOS Runtime isolates applications from each other and from the
system. Every application runs in a **sandbox** and must declare the
permissions it needs.

## Principles

1. **Least privilege** — an application gets only the permissions it
   declares.
2. **Isolation** — one application cannot read another application's
   data.
3. **Verified origin** — applications are cryptographically signed.
   Unsigned applications run only in development mode.
4. **Consent** — users must grant permissions before an application
   can use them.
5. **No escalation** — an application cannot elevate its own
   privileges at runtime.

## Layers

```
┌─────────────────────────────────────┐
│   Application A    │ Application B  │  ← Separate sandboxes
├───────────────────┼─────────────────┤
│           Permissions Layer         │  ← Enforced by Security Monitor
├─────────────────────────────────────┤
│         Platform API                │  ← Permission checks here
├─────────────────────────────────────┤
│         Runtime Services            │  ← IPC authenticated
├─────────────────────────────────────┤
│         Operating System            │  ← Linux namespaces, etc.
└─────────────────────────────────────┘
```

## Sandboxing

On Linux, sandboxing uses:

- **User namespaces** — each application has its own UID.
- **Mount namespaces** — restricted filesystem view.
- **Seccomp** — syscall filtering.
- **Cgroups** — resource limits (CPU, memory, I/O).

On Windows, equivalent mechanisms (job objects, restricted tokens) are
used.

## Permission enforcement

Permissions are declared in the package manifest and enforced at
runtime:

```toml
[permissions]
network.client = true
storage.persistent = true
cloud.saves = true
```

If an application calls an API that requires a permission it has not
been granted, the API returns a permission error. The Runtime never
grants permissions silently.

## Code signing

Applications must be signed with an Ed25519 key. The Runtime verifies
the signature before installing or running an application. Development
builds can use unsigned packages with a `devMode = true` manifest flag.

## User consent

When an application is installed, the user sees the permission list
and must approve it. Individual permissions cannot be selectively
denied — it is all-or-nothing. If the user does not consent, the
application is not installed.

## Runtime service authentication

Runtime services authenticate to each other using code signing. The
Security Monitor verifies each service before allowing it to join the
IPC bus. This prevents a compromised application from impersonating a
Runtime service.

## Related

- [Permissions (Package Format)](../10-package-format/09-permissions.md)
- [Signing (Package Format)](../10-package-format/10-signing.md)
- [Permissions API](../06-platform-api/23-permissions-api.md)
- [Security and Trust](../02-platform-principles/09-security-and-trust.md)
- RFC-0001 (Platform Principles, Security section)
