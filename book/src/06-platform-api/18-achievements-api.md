# Achievements API

The Achievements API tracks player progress and unlocks through PlayOS Cloud.
It is a **runtime-only** feature gated on the optional capability
`cloud.achievements` (enum: `Capability::Achievements`).

## Purpose

- **Unlock achievements** — mark an achievement as completed.
- **Progress tracking** — report partial progress toward an achievement
  (e.g. "Collect 47/100 coins").
- **Query state** — check which achievements are unlocked.

## API concepts

```cpp
namespace PlayOS {
namespace Achievements {

// Unlock an achievement by its developer-defined ID.
void Unlock(const std::string& achievementId);

// Report partial progress (0.0 to 1.0). Reaching 1.0 is equivalent to Unlock.
void SetProgress(const std::string& achievementId, float progress);

// Query unlock state for all achievements.
std::vector<AchievementState> GetAll();

} // namespace Achievements
} // namespace PlayOS
```

An `AchievementState` contains the achievement ID, whether it is unlocked,
current progress (0.0–1.0), and the unlock timestamp if completed.

## Achievement definition

Achievements are defined in the developer portal, not in code. Each
achievement has:
- A stable string identifier.
- A display name and description.
- An icon.
- Whether it is visible before unlock (secret achievements are hidden).
- Optional point value for gamerscore/trophy systems.

## Capability

| Capability | Enum | Status |
|---|---|---|
| `cloud.achievements` | `Capability::Achievements` | Optional (runtime-only) |

## Failure behavior

When `Capability::Achievements` is absent:
- `Unlock()` and `SetProgress()` are no-ops.
- `GetAll()` returns an empty vector.

This allows the same code to run on SDK Targets without modification —
achievements simply don't fire.

## Requirements

- Achievements unlocked while offline MUST be queued and submitted when
  connectivity is restored.
- `SetProgress` reaching 1.0 MUST behave identically to `Unlock`.
- Unlocking an already-unlocked achievement is a no-op (not an error).
- Achievement state SHOULD be cached locally so `GetAll()` works offline.

## Related

- [Cloud Architecture](../11-cloud-and-marketplace/03-cloud-architecture.md)
- [Achievements](../11-cloud-and-marketplace/13-achievements.md)
- [Developer Portal](../11-cloud-and-marketplace/05-developer-portal.md)
- RFC-0004 (Platform API Surface)
- RFC-0007 (Cloud and Marketplace Architecture)
