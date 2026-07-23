# Target Compatibility Levels

PlayOS defines three compatibility levels that describe what a target
provides and what applications can expect.

## Levels

### Platform API Compatible

The minimum bar. The target implements all **required capabilities**
and passes the API conformance test suite.

- Applications can use the core (portable) subset of the Platform API.
- Runtime-only features (overlay, cloud saves, marketplace) report
  capabilities as absent.
- This is the level for SDK Targets (Windows, macOS, Linux, Android,
  PS Vita).

### Runtime Compatible

The target runs the full PlayOS Runtime, supports overlay, game
switching, package installation, and suspend/resume.

- All Platform API Compatible requirements, plus:
- `system.overlay`, `system.game-switching`,
  `system.suspend-resume`, `system.package-install` are present.
- Cloud capabilities (`cloud.saves`, `cloud.leaderboards`,
  `cloud.achievements`) are present when connected to a PlayOS Cloud
  instance.
- This is the minimum level for a device sold as a "PlayOS Console."

### PlayOS Certified Hardware

The highest bar. In addition to Runtime Compatible, the device passes:

- **Performance requirements** — minimum frame rates, input latency,
  boot time.
- **Accessibility requirements** — screen reader support, remappable
  controls, subtitle support.
- **Security review** — verified boot, secure storage, hardware root
  of trust.
- **Physical design** — controller layout meets ergonomic guidelines.

## Level table

| Feature | Platform API Compatible | Runtime Compatible | Certified Hardware |
|---|---|---|---|
| Required capabilities | ✅ | ✅ | ✅ |
| API conformance tests | ✅ | ✅ | ✅ |
| Runtime services | ❌ | ✅ | ✅ |
| Cloud capabilities | ❌ | ✅ | ✅ |
| Performance requirements | ❌ | ❌ | ✅ |
| Accessibility requirements | ❌ | ❌ | ✅ |
| Security review | ❌ | ❌ | ✅ |

## How applications query the level

Applications do not query the compatibility level directly. They query
capabilities. The level determines which capabilities are available:

- `Platform API Compatible` → required capabilities only.
- `Runtime Compatible` → required + runtime + cloud capabilities.
- `Certified Hardware` → everything above + vendor capabilities.

## Related

- [Certification](../14-certification/01-overview.md)
- [Runtime Devices](02-runtime-devices.md)
- [SDK Targets](03-sdk-targets.md)
- [Certified Devices](05-certified-devices.md)
