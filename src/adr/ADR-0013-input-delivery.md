# ADR-0013 — Input Delivery Architecture

**Status:** Accepted (2026-09-26)
**Related:** ADR-0007 (audio ownership), ADR-0012 (`WLR_BACKENDS`/libinput finding), Sprint 7 overlay architecture, Sprint 8 gamepad input, `security-model.md` §3 and §8, `post-mvp.md` (Input Service)
**Refines:** the Wayland-seat-only assumption in [Sprint 17](sprints/Sprint-17.md) and the Touch/OSK entry in [post-mvp.md](post-mvp.md)

## Context

Sprint 17 was written as a Wayland-native input feature: touch arriving as `wl_touch`, forwarded by the
compositor and hit-tested with `wlr_scene_node_at`; text entry via `zwp_text_input_v3`; the OSK rendered
by the overlay. Building T1 measured what the platform actually does instead:

- the compositor's `wlr_scene` holds only a full-screen background rect, and `toplevels=0` while the shell is rendering
- `playos_overlay_v1::set_surface` ignores its `surface` argument — it is a stub
- presentation reports `present zero-copy=9, copied=0`
- client input already arrives through `libplayos` → **evdev**: that is how the built-in controller works,
  and how the shell receives the Armoury button
- the touch panel itself is a healthy Linux device (`hid-multitouch`, `/dev/input/event5`, `ABS_MT_*`)

Input therefore does not reach today's clients through the seat, and cannot: there is no `wl_surface` on
that path for a seat notification to target.

## Decision

Two client classes, two input paths, chosen by what the client *is*.

1. **First-party clients** — the shell, the overlay, and every game built against the SDK — receive input
   through **`playos-platform-api`'s evdev backend**, alongside the gamepad support that already works.
   Touch joins that path (`ABS_MT_*`), and text entry becomes an explicit PlayOS API rather than a Wayland
   protocol.
2. **Foreign Wayland clients** — a browser, media clients, anything not built against the SDK — receive
   input through the **Wayland seat**: pointer/touch forwarding with scene hit-testing, and
   `zwp_text_input_v3` for text, exactly as Sprint 17 originally specified. This becomes live when such
   clients exist; Sprint 20's media/browser strategy is the likely trigger.

The **OSK is one system keyboard**, rendered by the overlay (Sprint 7 architecture) and driven over
`playos_overlay_v1` (`osk_visibility`, `osk_commit_string`) — the shape Sprint 17's T5/T6 already describe.
It does not depend on `zwp_text_input_v3`.

The seat keeps a **policy** role regardless: the reserved-button intercept in `system_button.c` is defence
in depth, and it runs for the first time only after the ADR-0012-era `WLR_BACKENDS` fix restored the
libinput backend.

## Why not a single path

- **Wayland-only** would require re-architecting trusted-client presentation (implementing `set_surface` as
  a real scene surface, plus a session) and would duplicate an input path that already works — for apps
  that do not yet exist.
- **evdev-only** would abandon standard input for the foreign clients Sprint 20 anticipates, putting
  IME, mice, tablets and accessibility out of reach.

## Consequences

- Sprint 17 is re-scoped: T1 keeps its (done and hardware-verified) kernel half, and its seat-forwarding
  half is now "for foreign Wayland clients"; T2/T4 are re-scoped the same way; T3's outcome
  (`GetTouchPosition`) is unchanged but fed from the platform API; T5–T8 stand as written.
- The platform API gains touch, and later keyboard/text, as first-class surfaces — so SDK samples and the
  shell share one path.
- The sandbox model is unchanged: no new device permissions and `libplayos` keeps masking reserved
  buttons, so games get touch without a new trust decision.
- A future `playos-input` service (post-mvp.md) is the natural home for cross-client input policy;
  nothing here precludes it.

## Open question (resolve before further seat work)

**How does the shell present its frames today?** It is not an xdg client, `set_surface` stores nothing, and
the compositor's own renderer never references a client surface. That answer decides whether first-party
clients could *become* Wayland surfaces cheaply — making path 2 universal — or are deliberately outside
Wayland, in which case path 1 is the only sane route.

---

## Direction confirmed (2026-09-26)

Product intent, stated by the maintainer: an **unparalleled console experience on handheld and
PC**; **every game and application is compiled against the PlayOS SDK**; the shell is a
**first-class UI** (LVGL, rendering through raylib) whose job is browsing and launching games
and apps and managing the **PlayOS Marketplace** — from the shell or a separate SDK app.

That resolves the open question above, and settles this ADR's choice:

- **Path 2 is not planned.** There are no foreign Wayland clients by design, so seat forwarding
  and `zwp_text_input_v3` are not on the product's path. They stay correct for a hypothetical
  future client class, and `playos-compositor/src/input.c` is kept as that implementation —
  inert today and documented as such. The seat keeps its **policy** role (reserved-button
  intercept), which is its day-job here.
- **Input belongs to the platform API for every client class**: gamepad (shipped), touch
  (handheld), and **keyboard and mouse** — because PlayOS targets PC hardware too, where a
  marketplace to browse and search makes them first-class rather than optional.
- **Text entry is a shell/overlay concern, not a protocol.** The OSK must be rendered by the
  **overlay**, since the shell stops rendering while a game is foreground (a locked shell
  invariant), and it is driven by a PlayOS API. LVGL's keyboard/textarea widgets are the
  pragmatic implementation (see [Sprint 22](sprints/Sprint-22.md)) instead of a hand-rolled
  keyboard.
- **The marketplace needs no new platform surface**: networking landed in Sprint 16, TLS should
  come with the OpenSSL that `wpa_supplicant`'s WPA3 support already pulls in (verify before
  relying on it), storage and the A/B update engine exist, and browsing/search is shell plus
  platform API.
- **Sequencing consequence:** make the LVGL decision (Sprint 22) *before* building more
  hand-drawn shell UI or a hand-rolled OSK — it supplies the widget layer and the keyboard.
