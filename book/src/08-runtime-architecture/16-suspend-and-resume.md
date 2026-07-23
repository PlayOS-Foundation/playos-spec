# Suspend and Resume

PlayOS Runtime Devices support **application suspension** — saving the
running application state and restoring it later without restarting the
application.

## Why suspend/resume

Handheld consoles are used in short sessions. Players expect to:

- Press the power button, put the device down, and pick it up hours
  later right where they left off.
- Switch between applications without losing progress.
- Receive a low-battery warning and have the device save state before
  power loss.

## Lifecycle states

```
      ┌─────────┐
  (1) │ Running │
      └────┬────┘
           │ Suspend
           ▼
      ┌─────────┐
  (2) │Suspended│ ──── Resume ────► Running
      └────┬────┘
           │ Low battery / timeout
           ▼
      ┌─────────┐
  (3) │ Hibernated │ ──── Restore ────► Running
      └─────────┘     (cold start + restore)
```

## How it works

### Suspend

1. The Lifecycle Manager sends `Lifecycle::RequestSuspend()` to the
   running application.
2. The application serializes its state in the `OnSuspend` callback.
3. The Runtime saves the serialized state + a memory snapshot to
   persistent storage.
4. The application process is paused or terminated.

### Resume

1. The Lifecycle Manager restores the application process from the
   memory snapshot (if available) or starts it fresh.
2. The Runtime restores the serialized state.
3. The application receives the `OnResume` callback with the saved
   state.
4. The application reconstructs its internal state.

## The `OnSuspend` callback

```cpp
// Called by the Runtime before suspending the application.
// The application must save enough state to restore later.
Lifecycle::SavedState OnSuspend();

struct SavedState {
    void* data;
    size_t size;
};
```

The `SavedState` buffer is opaque to the Runtime — its contents are
entirely up to the application. Common practice: serialize to a binary
blob using a serialization library.

## The `OnResume` callback

```cpp
// Called when the application resumes.
void OnResume(const Lifecycle::SavedState& state);
```

## Capability

Suspend/resume is gated by `lifecycle.suspend-resume`. Applications
query it:

```cpp
if (Capabilities::Has(Capability::LifecycleSuspendResume)) {
    // Runtime Device — full suspend/resume support
}
```

On SDK Targets, this capability is absent. Applications should still
implement `OnSuspend`/`OnResume` for testing.

## Constraints

- Suspend/resume **must** complete within 2 seconds on the reference
  device (ROG Ally).
- Saved state **must** be no larger than 50 MB.
- If the device loses power during hibernation, the application starts
  fresh (no state restoration).

## Related

- [Lifecycle API](../06-platform-api/07-lifecycle-api.md)
- [Runtime Services](08-runtime-services.md)")
- [Game Switching](11-game-switching.md)
