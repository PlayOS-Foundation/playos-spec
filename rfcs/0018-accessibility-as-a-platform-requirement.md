---
rfc: 0018
title: Accessibility as a Platform Requirement
status: Draft
authors: []
created: 2026-07-22
---

# RFC-0018: Accessibility as a Platform Requirement

> **Status:** Draft

## Summary

This RFC formalises accessibility as a platform-level requirement for
PlayOS.  It defines the mandatory and recommended accessibility
features, the certification requirements for PlayOS Certified
Hardware, the accessibility API surface, and how applications
integrate with system accessibility preferences.  It builds on the
[Accessibility][a11y] chapter in the Shell and UX part.

[a11y]: ../book/src/09-shell-and-ux/14-accessibility.md

## Motivation

Accessibility is not an optional feature for a console platform — it
is a certification requirement (RFC-0010).  Without a formal
specification:

- Each device would implement accessibility differently, fragmenting
  the experience.
- Developers would not know what accessibility features to support.
- Players with disabilities would have inconsistent experiences across
  devices.
- Certification would lack measurable accessibility criteria.

This RFC defines what "accessible" means for PlayOS, making it
consistent, testable, and enforceable.

## Guide-Level Explanation

PlayOS accessibility is designed around the controller — the primary
input device for the platform.  Every accessibility feature works
with a gamepad.

**For players:** Screen reader reads the UI aloud.  You can remap any
button.  High contrast and large text make everything readable.
Subtitles are system-wide — set your preferences once, and every game
respects them.

**For developers:** Query the system accessibility preferences through
the Platform API.  Respect subtitle preferences, reduce-motion
settings, and color blind modes.  The platform handles the rest.

**For hardware vendors:** Screen reader, button remapping, high
contrast, large text, and subtitle support are required for
certification.  Color blind modes and mono audio are recommended.

## Reference-Level Explanation

### 1. Mandatory Features (Level 3 Certification)

| Feature | Requirement | Verification |
|---|---|---|
| Screen reader | Controller-operated; reads all Shell UI | Manual testing + automated element enumeration |
| Button remapping | System-wide; per-app profiles | Manual testing |
| High contrast mode | Contrast ratio ≥ 7:1 | Automated contrast measurement |
| Large text mode | Minimum 36 px; 1.5× scaling | Automated text size measurement |
| Subtitle support | System-wide preferences (size, color, bg) | API query test |
| Stick dead zone adjustment | Per-stick customization | Manual testing |
| Vibration control | Reduce/disable haptic feedback | Manual testing |

### 2. Recommended Features

| Feature | Requirement |
|---|---|
| Color blind modes | Protanopia, deuteranopia, tritanopia filters |
| Mono audio | Combine stereo channels |
| Reduce motion | Disable animations and transitions |
| Audio descriptions | Secondary audio track describing visual content |
| Switch controller support | External adaptive controllers |
| Eye tracking | Gaze-based navigation (future hardware) |

### 3. Screen Reader

The screen reader is the primary accessibility tool.  It MUST:

- Be operable entirely with a gamepad.
- Read element title, type, and state ("Settings button, selected").
- Provide a full-screen summary on demand.
- Support configurable speech rate (slow/normal/fast).
- Support configurable verbosity (brief/detailed).

#### Controller mapping

| Button | Action |
|---|---|
| D-Pad | Move focus.  Each element read aloud on focus. |
| A | Activate focused element. |
| B | Go back / close. |
| Y | Re-read current element. |
| X | Read full screen summary. |
| View/Back | Open accessibility quick menu. |
| Right Stick | Adjust speech rate (up=faster, down=slower). |

#### What the screen reader reads

```text
Focus moves to "Settings":
  → "Settings. Button. Press A to open."

Focus moves to Wi-Fi toggle:
  → "Wi-Fi. Toggle switch. On. Press A to toggle."

Screen summary (X button):
  → "Settings screen. Seven items. Wi-Fi on. Bluetooth off.
     Brightness 75 percent. Volume 50 percent."
```

### 4. Button Remapping

Button remapping is system-wide, with optional per-application
overrides:

```
┌──────────────────────────────────────────┐
│  ← Button Remapping                      │
├──────────────────────────────────────────┤
│  Profile: [Global ▾]                     │
│                                          │
│  A Button    → [A Button       ▾]        │
│  B Button    → [B Button       ▾]        │
│  X Button    → [X Button       ▾]        │
│  Y Button    → [Y Button       ▾]        │
│  D-Pad Up    → [D-Pad Up       ▾]        │
│  D-Pad Down  → [D-Pad Down     ▾]        │
│  Left Stick  → [Left Stick     ▾]        │
│  Right Stick → [Right Stick    ▾]        │
│  ...                                     │
│                                          │
│  [+ New per-app profile]                 │
│  [Reset to defaults]                     │
└──────────────────────────────────────────┘
```

#### Requirements

- Every physical button MUST be remappable to any logical button.
- Per-app profiles override global settings when that app is running.
- Remapping applies to the Shell UI and all applications.
- The home button (`Button::Home`) MUST always function as Home; it
  cannot be remapped away.
- Remapping state is persisted in the device profile and survives
  reboots.

### 5. Subtitle System

Subtitles are configured system-wide and queried by applications:

```cpp
// Application queries system subtitle preferences
auto prefs = Accessibility::GetSubtitlePrefs();
// prefs.enabled
// prefs.fontSize       // small, medium, large, extra-large
// prefs.fontColor      // white, yellow, custom
// prefs.backgroundColor // none, semi-transparent, solid black
// prefs.showSpeakerName // true/false
```

#### Settings UI

```
┌──────────────────────────────────────────┐
│  ← Subtitles                             │
├──────────────────────────────────────────┤
│  Enable Subtitles             [●]        │
│                                          │
│  Font Size                               │
│  ○ Small  ● Medium  ○ Large  ○ XL        │
│                                          │
│  Font Color                              │
│  ● White  ○ Yellow  ○ Custom             │
│                                          │
│  Background                              │
│  ○ None  ● Semi-transparent  ○ Solid     │
│                                          │
│  Show Speaker Name            [●]        │
└──────────────────────────────────────────┘
```

Applications SHOULD respect these preferences in their own subtitle
rendering.  A game that ignores system subtitle preferences and uses
its own settings MUST still provide equivalent accessibility.

### 6. High Contrast and Large Text

#### High contrast mode

- Increases contrast ratio to ≥ 7:1 for all Shell UI elements.
- May invert or simplify colors.
- Applies to Shell, overlay, and system dialogs.
- Applications can query the high contrast state:

```cpp
if (Accessibility::IsHighContrastEnabled()) {
    // Use high-contrast color palette
}
```

#### Large text mode

- Scales all Shell text by 1.5×.
- Minimum text size: 36 px (24 px base × 1.5).
- Does not affect application content (apps query and adapt).

### 7. Accessibility API

The Platform API exposes accessibility state:

```cpp
namespace PlayOS::Accessibility {

// System-wide preferences
bool IsScreenReaderEnabled();
bool IsHighContrastEnabled();
bool IsLargeTextEnabled();
bool IsReduceMotionEnabled();
ColorBlindMode GetColorBlindMode();
SubtitlePrefs GetSubtitlePrefs();

// Per-device capabilities
float GetSpeechRate();         // 0.5–2.0
Verbosity GetVerbosity();      // brief, normal, detailed
bool IsMonoAudioEnabled();

} // namespace PlayOS::Accessibility
```

### 8. Certification Testing

Accessibility certification testing includes:

| Test | Method |
|---|---|
| Screen reader covers all Shell screens | Automated element tree dump + manual review |
| All elements have accessible labels | Automated check for null/empty labels |
| High contrast mode delivers ≥ 7:1 ratio | Automated contrast measurement tool |
| Large text renders at ≥ 36 px | Automated text size measurement |
| Button remapping works for all buttons | Manual test matrix |
| Subtitle preferences are queryable | API conformance test |
| Screen reader works with gamepad only | Manual test (no touch/keyboard) |

### 9. Developer Guidelines

Applications SHOULD:

- Query and respect `Accessibility::GetSubtitlePrefs()`.
- Support high contrast mode when `IsHighContrastEnabled()`.
- Support large text when `IsLargeTextEnabled()`.
- Respect `IsReduceMotionEnabled()` (disable parallax, screen shake).
- Provide text alternatives for non-text content (icons, images).
- Ensure all UI is navigable by D-Pad (no pointer-only interactions).

Applications MUST:

- Not override system accessibility settings.
- Not disable the screen reader or interfere with its operation.
- Handle the focus model correctly (the platform moves focus; the app
  responds).

## Drawbacks

- Screen reader development is complex and requires ongoing
  maintenance as the Shell UI evolves.
- Requiring high contrast and large text constrains Shell UI design.
- Per-app button remapping profiles add complexity to the Shell
  settings system.
- Accessibility certification testing requires specialized expertise
  that the Foundation may not have in-house.

## Rationale and Alternatives

- **Accessibility as optional (not a certification requirement):**
  Simpler but excludes players with disabilities.  Rejected — a
  console platform must be accessible.
- **Platform-only accessibility (apps handle their own):** Shifts
  burden to developers.  PlayOS provides system-wide preferences that
  apps query, reducing per-app work while allowing app-specific
  customization.
- **Full WCAG compliance vs. console-specific guidelines:** WCAG is
  web-focused; console accessibility has different constraints
  (gamepad navigation, no pointer, TV viewing distance).  PlayOS
  defines console-specific guidelines.

## Prior Art

- **Xbox Accessibility Guidelines** (XAG): Microsoft's console
  accessibility standard, including controller remapping (Xbox
  Accessories app), screen reader (Narrator), and subtitle
  preferences.
- **PlayStation 5 accessibility:** Screen reader, button remapping,
  zoom, high contrast, mono audio — similar feature set to what
  PlayOS targets.
- **Nintendo Switch:** Limited built-in accessibility (no screen
  reader, no system-wide remapping).  PlayOS aims higher.
- **Game Accessibility Guidelines** (gameaccessibilityguidelines.com):
  Industry best practices for game accessibility.  PlayOS adopts the
  platform-level subset.
- **Android Accessibility** (TalkBack, Switch Access): Reference for
  how a screen reader works with non-touch input.

## Unresolved Questions

- Should the screen reader be available in all languages at launch, or
  phased?  The reference implementation targets English first; the API
  supports localisation.
- How are third-party accessibility controllers (Xbox Adaptive
  Controller, Hori Flex) supported?  They should appear as standard
  HID gamepads and work with button remapping.
- Should there be an accessibility certification mark separate from
  PlayOS Certified Hardware?  "PlayOS Accessible" could recognize
  devices that exceed minimum requirements.

## Future Possibilities

- **Voice control:** Navigate the Shell with voice commands
  ("Open Settings", "Launch Space Invaders").
- **Eye tracking navigation:** Gaze-based focus for players who
  cannot use a gamepad, using compatible eye-tracking hardware.
- **Haptic feedback for UI:** Vibration patterns that communicate
  focus changes and state transitions for players with visual
  impairments.
- **AI-generated audio descriptions:** Real-time audio description of
  game content using on-device ML.
- **Community accessibility profiles:** Share and download button
  remapping and accessibility profiles created by the community.
