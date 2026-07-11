# Problem Statement

Building a console-style gaming experience today is harder than it should be.
The problems fall into three categories: **developers** are forced to build
infrastructure instead of games, **players** are locked into proprietary
ecosystems, and **hardware vendors** have no open platform to target.

## Developers build operating systems instead of games

To ship a game that feels like a console title on a Linux handheld, a
developer must:

- Read raw input from `/dev/input/event*` or integrate evdev/libinput
  directly — there is no standard "A button pressed" API.
- Handle display modes through DRM/KMS ioctls or negotiate Wayland
  protocols — there is no standard "give me 1080p at 120 Hz" call.
- Route audio through PipeWire or ALSA — there is no standard "play this
  sound at this volume" helper.
- Manage suspend/resume, brightness, battery, and thermal policy by reading
  sysfs and writing udev rules — there is no standard platform abstraction.

These are **operating-system concerns**, not game-design concerns. Every
developer who wants to target a handheld PC or a custom console reinvents
the same plumbing. Smaller teams abandon the attempt or ship broken
experiences. Larger teams waste months on infrastructure that disappears
the moment they port to a different device.

Contrast this with a traditional console: one SDK, one set of APIs, one
target. The developer calls `Controller::isPressed(A)` and it works. That
simplicity does not exist on open platforms today.

## Players are locked into closed ecosystems

On a proprietary console, the player's library, saves, achievements,
friends list, and purchase history are tied to a single vendor's cloud.
When the hardware generation ends, that library often becomes inaccessible.
Emulation and preservation communities reverse-engineer these platforms
because no open specification exists.

PC gaming offers openness but no cohesive console experience. A player who
wants a handheld gaming device must choose between:

- **A proprietary handheld** (Nintendo Switch, PlayStation Portal) with a
  polished UI but a closed ecosystem.
- **A Windows handheld** (ROG Ally, Steam Deck with Windows) with access to
  many stores but no unified controller-first interface.
- **A Linux handheld** (Steam Deck with SteamOS) with better
  controller-first design but tight coupling to a single storefront.

None of these give the player full ownership of their games, saves, and
experience across devices and across generations.

## Hardware vendors reinvent the platform

Every new handheld PC — from the AYANEO to the Lenovo Legion Go — ships
with a custom launcher, custom input mapper, custom TDP manager, and custom
overlay. These are not differentiated features; they are **table-stakes
infrastructure** that every vendor builds from scratch because no open,
shared platform exists.

Vendors who want to ship a console experience on Linux must choose between:

- Forking an existing compositor (Gamescope), coupling their product to
  Valve's roadmap.
- Building a custom Wayland compositor and input stack, requiring deep
  Linux graphics expertise.
- Shipping Windows with a custom shell overlay, accepting the overhead and
  licensing costs.

A shared, open platform would let vendors differentiate on hardware,
industrial design, and services — not on reinventing the input stack.

## The root cause: no open console platform contract

Traditional consoles solve these problems with a **platform contract** — a
stable set of APIs, a compositor, a runtime, and a certification program
that guarantees consistency across devices. That contract has never existed
in the open.

Desktop Linux provides the **pieces** (DRM/KMS, evdev, PipeWire, Wayland,
systemd) but not the **contract**. There is no specification that says "a
PlayOS device must provide input, display, storage, and lifecycle APIs in
this exact form." Without that contract, developers cannot target "Linux
handhelds" — they target a specific kernel, a specific compositor, a
specific audio server, on a specific device.

PlayOS exists to be that contract.

See also: [What Is PlayOS?](01-what-is-playos.md),
[Mission](02-mission.md), and [Non-Goals](04-non-goals.md).

