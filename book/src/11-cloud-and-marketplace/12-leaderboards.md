# Leaderboards

**Leaderboards** let applications submit scores and display rankings.
Players compete with friends and the global community.

## Leaderboard types

| Type | Description |
|---|---|
| **Global** | All players, ranked by score. |
| **Friends** | Only the player's friends. |
| **Daily** | Resets every 24 hours. |
| **Weekly** | Resets every 7 days. |
| **Developer-defined** | Custom time windows or filters. |

## Submission

Applications submit scores through the Platform API:

```cpp
Leaderboards::SubmitScore("level1_highscore", 12500);

// With metadata (e.g., character used, time played)
Leaderboards::SubmitScore("speedrun", 184200, 
    R"({"character": "mage", "seed": 42})");
```

The cloud service validates the submission and updates rankings.

## Rankings

Applications query rankings:

```cpp
// Top 10 global
auto global = Leaderboards::GetRankings("level1_highscore", 
    LeaderboardScope::Global, 0, 10);

// Player's friends, centered on the player
auto friends = Leaderboards::GetRankings("speedrun",
    LeaderboardScope::Friends, 0, 5);
```

Each entry contains:

```cpp
struct LeaderboardEntry {
    uint64_t rank;
    char playerName[64];     // Display name
    int64_t score;
    uint64_t timestamp;      // Unix timestamp
    char metadata[256];      // Developer-defined JSON
};
```

## Anti-cheat

Leaderboards implement basic anti-cheat:

- Scores must be submitted from a valid application session.
- Impossible scores are rejected (configurable thresholds).
- Rapid-fire submissions are rate-limited.
- Reports from players trigger manual review.

For competitive games, developers should implement their own
server-side validation.

## Offline scores

If a device is offline, scores are queued locally and submitted when
connectivity returns. The player sees a "pending sync" indicator.
Offline scores are timestamped with the local time — the server
accepts timestamps within a reasonable window (±24 hours).

## Self-hosting

The Leaderboard service uses PostgreSQL. The self-hosted deployment
includes the same ranking queries and anti-cheat rules.

## Capability

Leaderboards are gated by `cloud.leaderboards`. Applications should
query this capability before showing leaderboard UI.

## Related

- [Leaderboard API](../06-platform-api/17-leaderboard-api.md)
- [Achievements](13-achievements.md)
- [Cloud Architecture](03-cloud-architecture.md)
