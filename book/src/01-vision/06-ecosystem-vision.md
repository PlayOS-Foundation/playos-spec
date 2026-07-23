# Ecosystem Vision

PlayOS is not just a platform — it is an **ecosystem** of developers,
players, hardware makers, store operators, and tool builders. This
chapter describes the ecosystem the platform enables.

## The ecosystem map

```text
                    ┌──────────────┐
                    │   Players    │
                    └──────┬───────┘
                           │ plays on
                    ┌──────▼───────┐
                    │   Devices    │ ← OEMs build these
                    └──────┬───────┘
                           │ runs
              ┌────────────┼────────────┐
              │            │            │
       ┌──────▼──┐  ┌──────▼──┐  ┌─────▼──────┐
       │ Runtime │  │  Shell  │  │  Platform  │
       └──────┬──┘  └─────────┘  │    API     │
              │                   └─────┬──────┘
              │                         │ calls
       ┌──────▼──────────┐     ┌───────▼───────┐
       │ Cloud Services  │     │  Applications │ ← Developers build these
       └──────┬──────────┘     └───────┬───────┘
              │                        │ distributed via
       ┌──────▼──────────┐     ┌───────▼───────┐
       │  Marketplaces   │◄────│   Packages    │
       └─────────────────┘     └───────────────┘
              ▲
              │ operated by
    ┌─────────┼─────────┐
    │         │         │
┌───▼──┐ ┌───▼───┐ ┌───▼──┐
│Official││Community││ OEM  │
│ Store │ │ Store  │ │Store │
└───────┘ └───────┘ └──────┘
```

## Roles in the ecosystem

### Players

Players buy or build PlayOS devices, install games from stores of their
choice, and play. They own their games and saves. They can side-load,
mod, and tinker.

### Developers

Developers write applications against the Platform API using any engine.
They package as `.gpk`, sign, and publish to one or more stores. They
set their own price or release for free.

### OEMs and device makers

OEMs build hardware — handhelds, set-top boxes, arcade cabinets,
dedicated consoles. They port the Runtime, pass certification, and
optionally bundle their own store.

### Store operators

Anyone can run a PlayOS-compatible store: the official marketplace,
a community store for indie games, an OEM store preloaded on a device,
a private store for a school or LAN party.

### Tool builders

The PlayOS SDK, CLI, and validation tools are open source. Third parties
can build additional tooling: level editors, asset pipelines, CI/CD
integrations, analytics dashboards.

## Related

- [Mission](02-mission.md)
- [Platform Promise](05-platform-promise.md)
- [Commercial Ecosystem](../16-governance-and-process/12-commercial-ecosystem.md)
