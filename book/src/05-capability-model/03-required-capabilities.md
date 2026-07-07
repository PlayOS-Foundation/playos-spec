# Required Capabilities

**Required capabilities** are those every conformant PlayOS target must
provide. They form the baseline an application can always rely on without a
capability query.

## The baseline set

At the "Platform API Compatible" level, a target MUST provide:

| Capability | Meaning |
|---|---|
| `input.basic` | Digital buttons and at least one analog stick, via logical PlayOS buttons |
| `storage.local` | Read/write access to save and config paths |
| `display.info` | Query display size and safe area |
| `lifecycle.basic` | Initialization, update loop, and shutdown |

These are the capabilities the [core (portable) subset](../04-target-model/03-sdk-targets.md)
of the Platform API depends on, so they are required on both SDK targets and
runtime devices.

## Rules

- A conformant target MUST provide every required capability.
- A target that cannot provide a required capability is non-conformant and
  MUST NOT claim the corresponding certification level.
- Because these are guaranteed, applications MAY use them without calling
  `Capabilities::Has(...)` first — though doing so is still harmless.

## Not required (examples)

Touch, battery, cloud saves, system overlay, marketplace, and suspend/resume
are **optional** or runtime-only — they are discovered, not assumed. See
[Optional Capabilities](04-optional-capabilities.md).

## Vertical slice note

The proof-of-concept relies only on `input.basic` and `lifecycle.basic` (and
`display.info` for layout). Everything else the slice touches — launching a
game, returning via Home — is provided by the runtime device, expressed as
runtime capabilities.

See: [Overview](01-overview.md),
[Runtime Capability Discovery](05-runtime-capability-discovery.md),
and RFC-0003.
