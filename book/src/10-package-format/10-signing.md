# Signing

Package signing provides integrity and authenticity verification. The
runtime verifies signatures before installation; unsigned packages are
supported for development but not for store distribution.

## Signature model

PlayOS uses detached signatures: the `signature` file at the package root
contains a cryptographic signature over the archive contents, not embedded
within individual files.

```text
MyGame.gpk
├── manifest.json
├── bin/...
├── assets/...
├── icon.png
└── signature           # detached signature
```

## Signing process

1. Developer generates an Ed25519 key pair.
2. Developer registers the public key with the marketplace (or uses a
   marketplace-managed signing service).
3. Before upload, the developer (or marketplace) signs the package archive.
4. The resulting signature is placed at `signature` in the package root.

## Verification

At install time, the runtime:

1. Reads the `signature` file.
2. Verifies it against the package contents using the developer's public
   key (obtained from the marketplace's key registry).
3. If verification fails, installation is rejected.
4. If no signature is present, the runtime treats the package as unsigned
   (allowed for development and side-loading, rejected for store installs).

## Development packages

Packages installed through Developer Mode or side-loaded via USB/network
MAY be unsigned. The runtime marks them as "unverified" in the library
and may show a warning. See [Development Packages](15-development-packages.md).

## Key management

- Developers manage their own signing keys. The marketplace does not hold
  private keys.
- Key compromise: the developer revokes the old public key and registers a
  new one. Packages signed with the revoked key are rejected on new
  installs.
- The marketplace may offer a managed signing service where it holds the
  key and signs on the developer's behalf.

## Requirements

- Signatures MUST use Ed25519.
- The `signature` file MUST be at the package root.
- The runtime MUST verify signatures before extraction.
- Signature verification failure MUST block installation with a clear
  error message.

## Related

- [Package Signature Format](../99-appendices/07-package-signature-format.md)
- [Security Model](../13-security-privacy-and-trust/02-trust-model.md)
- [Package Validation](16-package-validation.md)
- RFC-0005 (Package Format)
