# Achievements

**Achievements** let applications define goals for players to unlock.
They provide engagement, bragging rights, and replay value.

## Achievement definition

Developers define achievements in the package manifest or Developer
Portal:

```toml
[[achievements]]
id = "first_win"
name = "First Victory"
description = "Win your first race."
icon = "achievements/first_win.png"
isSecret = false
points = 10

[[achievements]]
id = "collector"
name = "Collector"
description = "Collect all 50 artifacts."
icon = "achievements/collector.png"
isSecret = false
points = 50
maxProgress = 50  # Progress-based achievement
```

## Unlocking

Applications unlock achievements through the Platform API:

```cpp
// Simple unlock
Achievements::Unlock("first_win");

// Progress-based
Achievements::SetProgress("collector", 12);  // 12 out of 50
```

The cloud service records the unlock and can notify friends.

## Progress display

The overlay can display an achievement toast when one is unlocked:

```
        ┌───────────────────────────────┐
        │  🏆 Achievement Unlocked!     │
        │  First Victory                │
        │  Win your first race.         │
        │                    +10 points │
        └───────────────────────────────┘
```

Applications control whether toasts are shown through the API:

```cpp
Achievements::Unlock("first_win", AchievementDisplay::ShowToast);
```

## Gamerscore

Each achievement has a `points` value. The sum of all unlocked
achievement points is the player's **Gamerscore** for that
application. Gamerscore is aggregated across all applications on the
player's profile.

## Offline achievements

If a device is offline, achievements unlock locally and sync when
connectivity returns. The timestamp is set to the local unlock time.

## Self-hosting

The Achievements service uses PostgreSQL. Self-hosted deployments
include achievement storage, progress tracking, and Gamerscore
aggregation.

## Capability

Achievements are gated by `cloud.achievements`.

## Related

- [Achievements API](../06-platform-api/18-achievements-api.md)
- [Leaderboards](12-leaderboards.md)
- [User Profile API](../06-platform-api/20-user-profile-api.md)
- [Cloud Architecture](03-cloud-architecture.md)
