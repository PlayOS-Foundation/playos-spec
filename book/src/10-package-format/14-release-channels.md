# Release Channels

Release channels let developers distribute different versions of an
application to different audiences: stable releases for all players, beta
builds for testers, and nightly builds for early feedback.

## Channels

| Channel | Audience | Typical use |
|---|---|---|
| `stable` | All players | Public release, reviewed and tested |
| `beta` | Opt-in testers | Pre-release testing, feedback |
| `nightly` | Developers and early adopters | Latest commits, may be unstable |

## Channel selection

The player selects a channel per application in the shell or store:

1. Navigate to the application in the library or store.
2. Open "Release Channel" settings.
3. Select `stable`, `beta`, or `nightly`.

The default is `stable`. Changing the channel triggers a download of the
latest version in the selected channel.

## Manifest

The manifest may specify the channel:

```json
{
  "releaseChannel": "beta"
}
```

This is informational — the marketplace manages channel assignment and
overrides manifest values.

## Versioning across channels

Versions are compared across channels: a `stable` version `1.1.0` is
newer than a `beta` version `1.0.0`. The runtime always offers the
highest version available in the selected channel.

## Requirements

- The marketplace MUST support at least `stable` and `beta` channels.
- Switching channels MUST NOT delete save data.
- The runtime MUST show which channel is active for each application
  in the library.

## Related

- [Updates and Patches](13-updates-and-patches.md)
- [Marketplace Catalog](../11-cloud-and-marketplace/07-marketplace-catalog.md)
