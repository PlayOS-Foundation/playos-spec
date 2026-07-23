# Accounts and Identity

The Accounts Service manages **player identity** — sign-in, profiles,
friends, and device linking.

## Identity model

A player has a **PlayOS Account** identified by a unique `playerId`
(UUID v7). The account is the root of all cloud data:

```
Player Account ─┬─ Cloud Saves
                ├─ Leaderboard Entries
                ├─ Achievement Progress
                ├─ Entitlements (purchased apps)
                ├─ Friends List
                └─ Device List
```

## Authentication

Authentication uses **OpenID Connect** (OIDC). Players can sign in
with:

- PlayOS Account (email + password or passkey)
- Third-party IdP (Google, Apple, GitHub)
- Anonymous (device-local, no cloud sync)

The Runtime obtains an OAuth 2.0 access token and attaches it to all
cloud API requests.

## Anonymous play

Players can play without an account. In anonymous mode:

- Saves are local-only (no sync).
- Leaderboards show anonymous entries.
- Achievements unlock locally but are lost on device reset.
- Purchased applications are tied to the device.

When the player signs in, anonymous data is optionally migrated to the
account.

## Device linking

A player can link multiple devices to their account. Linked devices
share:

- Cloud saves
- Achievement progress
- Entitlements (applications)
- Preferences (language, accessibility)

## Friends

Players can add friends by `playerId`. Friends enable:

- Friend leaderboards
- Presence ("Playing Game X")
- Multiplayer invites (future)

## Privacy

- `playerId` is never exposed in leaderboards (display name only).
- Players can set their profile to private.
- Friends requests require mutual consent.

## Related

- [Developer Portal](05-developer-portal.md)
- [Entitlements](09-entitlements.md)
- [Cloud Architecture](03-cloud-architecture.md)
