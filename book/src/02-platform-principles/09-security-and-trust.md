# Security and Trust

PlayOS is built on a foundation of **security and player trust**. Every
application is sandboxed, every permission is consented to, and every
package is verifiable.

## The principle

> The player controls what applications can access. The platform enforces
> it. The application cannot bypass it.

## Security pillars

### 1. Sandboxing

Every application runs in a restricted environment. It cannot access
another application's save data, the system files, or hardware without
explicit permission. See [Sandboxing](../13-security-privacy-and-trust/05-sandboxing.md).

### 2. Permissions

Applications declare what they need in the manifest. The player grants
or denies at install time. Permissions can be revoked at any time. See
[Permissions](../13-security-privacy-and-trust/04-permissions.md).

### 3. Package signing

Packages from the store are cryptographically signed. The runtime
verifies the signature before installation. The player can see who
published the application. See [Package Signing](../13-security-privacy-and-trust/03-package-signing.md).

### 4. Secure updates

System and application updates are delivered over TLS and verified
before application. The platform cannot be downgraded to a vulnerable
version. See [Secure Updates](../13-security-privacy-and-trust/06-secure-updates.md).

### 5. Privacy by default

Telemetry is opt-in. Crash reports are anonymized. The player's save
data is encrypted at rest on cloud storage. See
[Privacy Principles](../13-security-privacy-and-trust/08-privacy-principles.md).

### 6. Child safety

Parental controls allow restricting content by rating, limiting play
time, and controlling purchases. See
[Child Safety and Parental Controls](../13-security-privacy-and-trust/10-child-safety-and-parental-controls.md).

## Trust model

The trust model is layered:

```text
Player trusts the device (hardware root of trust)
    → Device trusts the Runtime (verified boot)
        → Runtime trusts the package (signature verification)
            → Application is granted permissions (player consent)
```

No layer can bypass the layer above it.

## Related

- [Security, Privacy and Trust](../13-security-privacy-and-trust/01-overview.md)
- [Trust Model](../13-security-privacy-and-trust/02-trust-model.md)
- [Certification](../14-certification/01-overview.md)
