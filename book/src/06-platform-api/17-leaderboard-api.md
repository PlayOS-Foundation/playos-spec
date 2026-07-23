# Leaderboard API

The Leaderboard API submits scores and queries rankings through PlayOS Cloud.
It is a **runtime-only** feature gated on the optional capability
`cloud.leaderboards` (enum: `Capability::Leaderboards`).

## Purpose

- **Score submission** — submit a numeric score for a named leaderboard.
- **Ranking queries** — query rankings around the player, friends, or
  globally.
- **Multiple leaderboards** — a single title can have many leaderboards
  (e.g. "Level 1", "Endless Mode", "Weekly").

## API concepts

```cpp
namespace PlayOS {
namespace Leaderboard {

// Submit a score. Higher is better by default.
SubmitResult Submit(const std::string& boardId, int64_t score);

// Query entries around the current player.
std::vector<LeaderboardEntry> GetEntries(
    const std::string& boardId,
    int rangeStart,   // 0 = top of board
    int count
);

} // namespace Leaderboard
} // namespace PlayOS
```

A `LeaderboardEntry` contains the player display name (or "Anonymous"),
rank, score, and timestamp.

## Leaderboard modes

Leaderboards MAY support ordering modes defined server-side:

| Mode | Meaning |
|---|---|
| `higher-is-better` | Highest score ranks first (default) |
| `lower-is-better` | Lowest score ranks first (time trials) |

The mode is configured in the developer portal when creating a
leaderboard, not in the API call.

## Capability

| Capability | Enum | Status |
|---|---|---|
| `cloud.leaderboards` | `Capability::Leaderboards` | Optional (runtime-only) |

## Failure behavior

When `Capability::Leaderboards` is absent:
- `Submit()` returns `SubmitResult::NotAvailable`.
- `GetEntries()` returns an empty vector.

## Requirements

- Scores MUST be associated with the authenticated player. If no player
  is signed in, the score MAY be submitted as anonymous or rejected.
- `GetEntries` MUST support querying by range (global top 100, player's
  position ±5, etc.).
- The platform MUST prevent score spoofing (server-side validation,
  signed submissions).

## Related

- [Cloud Architecture](../11-cloud-and-marketplace/03-cloud-architecture.md)
- [Leaderboards](../11-cloud-and-marketplace/12-leaderboards.md)
- RFC-0004 (Platform API Surface)
- RFC-0007 (Cloud and Marketplace Architecture)
