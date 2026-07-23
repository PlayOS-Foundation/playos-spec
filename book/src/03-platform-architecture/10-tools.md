# Tools

The **PlayOS Tools** are developer-facing command-line and GUI tools
for building, packaging, testing, and debugging PlayOS applications.

## Tool list

| Tool | Description |
|---|---|
| `playos-cli` | Main CLI: build, package, deploy, test |
| `playos-package` | Create and validate .gpk files |
| `playos-sign` | Sign packages with Ed25519 keys |
| `playos-verify` | Verify package signatures |
| `playos-device` | Manage connected Runtime Devices |
| `playos-profile` | Manage device profiles |
| `playos-cert` | Run certification tests |
| `playos-debug` | Attach debugger to running application |

## `playos-cli`

The main entry point for developers:

```bash
# Create a new project
playos-cli init my-game

# Build for the current target
playos-cli build

# Package as .gpk
playos-cli package

# Deploy to connected device
playos-cli deploy --device rog-ally

# Run certification tests
playos-cli test --suite conformance
```

## `playos-package`

Creates and validates .gpk packages:

```bash
# Create a package from a build directory
playos-package create ./build --output my-game-1.0.0.gpk

# Validate an existing package
playos-package validate my-game-1.0.0.gpk

# Inspect a package without extracting
playos-package info my-game-1.0.0.gpk
```

## `playos-sign`

Manages signing keys:

```bash
# Generate a new signing key
playos-sign generate-key --output my-key.ed25519

# Sign a package
playos-sign sign my-game.gpk --key my-key.ed25519

# Verify a package signature
playos-sign verify my-game.gpk
```

## `playos-device`

Manages connected Runtime Devices:

```bash
# List connected devices
playos-device list

# Deploy a package to a device
playos-device deploy my-game.gpk --device rog-ally

# View device logs
playos-device logs --device rog-ally --follow
```

## IDE integration

The tools are designed to be IDE-agnostic. Editor integrations
(e.g., VS Code extension, JetBrains plugin) wrap the CLI tools.
The CLI is the canonical interface.

## Related

- [Developer Guide](../15-developer-guide/01-overview.md)
- [Package Validation](../10-package-format/16-package-validation.md)
- [Signing](../10-package-format/10-signing.md)
- [Reference Implementations](11-reference-implementations.md)
