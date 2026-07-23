# User Profile API

The User Profile API provides access to the currently signed-in player's
profile information. It is a **runtime-only** feature available on devices
with cloud connectivity.

## Purpose

- **Display player name** — show the player's display name in-game.
- **Avatar access** — retrieve the player's avatar image.
- **Presence** — basic online/offline/game status.

## API concepts

```cpp
namespace PlayOS {
namespace UserProfile {

// Returns the display name of the signed-in player,
// or "Player" if not signed in.
const char* DisplayName();

// Returns a path to the player's avatar image (PNG, square),
// or nullptr if no player is signed in.
const char* AvatarPath();

// Returns true if a player is signed in to PlayOS Cloud.
bool IsSignedIn();

} // namespace UserProfile
} // namespace PlayOS
```

## Anonymous play

PlayOS supports anonymous play: no sign-in required to launch and play
games. When no player is signed in:
- `DisplayName()` returns `"Player"` or a localized equivalent.
- `AvatarPath()` returns `nullptr`.
- Cloud features (saves, leaderboards, achievements) are unavailable.
- Cloud features silently no-op rather than prompting for sign-in.

The shell's Home screen prompts for sign-in at the system level; individual
games do not need to implement sign-in flows.

## Capability

The User Profile API does not require a separate capability. `IsSignedIn()`
returns `false` when cloud services are unavailable, and `DisplayName()`
always returns a usable string.

## Requirements

- `DisplayName()` MUST return a non-null, non-empty string at all times.
- `AvatarPath()` MUST return a valid PNG file path or `nullptr`.
- The platform MUST NOT prompt for sign-in from within game code —
  sign-in is a shell-level concern.

## Related

- [Accounts and Identity](../11-cloud-and-marketplace/04-accounts-and-identity.md)
- [Shell and UX](../09-shell-and-ux/01-overview.md)
- [Cloud Architecture](../11-cloud-and-marketplace/03-cloud-architecture.md)
