# Package Goals

The package format is designed around a small set of goals that every design
decision must serve.

## Goals

### 1. Self-contained

A `.gpk` contains everything needed to run the application: executable,
assets, manifest, and metadata. No network access is required at install
time beyond downloading the package itself.

### 2. Portable

The same `.gpk` must work on every PlayOS Runtime Device, regardless of the
underlying operating system. Architecture-specific executables live in
`bin/<arch>/`; the runtime selects the correct one at launch time.

### 3. Verifiable

The runtime must be able to verify the integrity and authenticity of a
package before installation. Signed packages provide a chain of trust from
developer to player.

### 4. Console-style lifecycle

Install, launch, update, and uninstall follow a predictable lifecycle that
the runtime manages end-to-end. The player never extracts a `.gpk` manually
or manages files.

### 5. Permission-aware

The manifest declares permissions. The runtime enforces them. The player
consents at install time. No application accesses resources without
declaration and consent.

### 6. Inspectable

A `.gpk` is a standard ZIP archive. Developers and power users can inspect
contents with any ZIP tool. Tooling validates packages against the schema
before upload.

## Non-goals

- **Streaming install** — the full package must be downloaded before
  installation. Streaming/partial install is deferred.
- **DRM** — the package format does not include DRM. Trust is based on
  signing and the marketplace entitlement model.
- **Delta updates** — updates replace the full bundle. Binary patching
  (delta updates) is deferred.

## Related

- [Package Anatomy](03-package-anatomy.md)
- [Package Validation](16-package-validation.md)
- RFC-0005 (Package Format)
