# App-Aware 3D Navigation Stack

## Context
This document captures the architecture plan discussed for extending the dual-sensor trackball firmware into a cross-platform, app-aware 3D navigation system while preserving regular mouse and scroll behavior.

Repository: `thomastthai/jfedor2-twist-to-scroll-trackball`

## Goals
- Preserve default mouse + wheel behavior with no host software required.
- Add optional advanced 3D navigation behavior for major 3D apps across Windows, macOS, and Linux.
- Use app-aware behavior driven by active application and modifier keys.
- Avoid naming tied to third-party branded products.

## Core Design

### 1) Firmware layer (device)
- Keep existing HID mouse report path as fallback.
- Add a vendor-defined raw motion input report that streams per-sensor deltas and button state.
- Add config flags:
  - `raw_stream_enabled`
  - `suppress_mouse_when_daemon_active`
- Keep backward compatibility by defaulting to standard mouse behavior.

Suggested raw report payload:

```c
struct __attribute__((packed)) raw_report_t {
    int16_t s1x, s1y;
    int16_t s2x, s2y;
    uint8_t buttons;
};
```

### 2) OS daemon layer (host)
- Read raw HID stream from device.
- Read keyboard modifier state (Ctrl, Shift, etc.).
- Detect active foreground application.
- Apply gesture classification and filtering (deadzone, smoothing, gain, axis lock).
- Route normalized navigation events to app plugins when available.
- Fallback to synthetic mouse/keyboard events where plugins are unavailable.

### 3) Application plugin layer
- Per-app plugin/add-on receives normalized nav events.
- Plugin applies transforms through native app APIs (camera/object orbit/pan/zoom).
- Optional context feedback (pivot, selection, camera mode) to daemon.

### 4) Profile/config layer
- Per-process profile mapping.
- Modifier-driven mode mapping (example: Ctrl=Rotate, Shift=Pan/Zoom).
- Axis inversion and sensitivity curves.

## Why plugin support is required
Input emulation alone is broad but imprecise. Plugins provide:
- Better camera/object context.
- More predictable behavior.
- Better compatibility with app-native navigation models.

Recommended strategy: plugin-first, emulation fallback.

## Cross-platform considerations

### Windows
- HID input: Raw Input / hidapi
- Active app: foreground window + process
- Injection fallback: `SendInput`

### macOS
- HID input: IOHIDManager
- Active app: frontmost application APIs
- Injection fallback: CGEvent
- Requires Input Monitoring / Accessibility permissions for full functionality.

### Linux
- HID input: hidraw/hidapi
- Active app + injection differ by X11 vs Wayland
- Wayland may require compositor-specific integration for app-awareness.

## Major app strategy (initial)
Start with plugin support where APIs are practical, then expand:
- Blender
- FreeCAD
- Rhino
- Maya
- 3ds Max
- Cinema 4D
- Houdini
- Unreal Editor
- Unity Editor
- Fusion 360
- SolidWorks

## Proposed implementation phases

### Phase 1: Firmware foundation
- Add raw report descriptor and report sending.
- Add config flags for raw stream + mouse suppression.
- Keep current behavior as default.

### Phase 2: Daemon MVP
- Cross-platform daemon skeleton.
- Raw ingest + modifier detection + app detection.
- Basic routing and fallback injection.

### Phase 3: First plugin integration
- Implement Blender plugin as reference.
- Define and stabilize plugin IPC contract.

### Phase 4: Broader plugin rollout
- Add additional DCC/CAD plugins iteratively.
- Add profile manager UX and preset library.

### Phase 5: Packaging and distribution
- Installers, code signing, auto-start integration.
- Permission onboarding and diagnostics.

## Suggested repository layout

```text
firmware/
daemon/
plugins/
  blender/
  maya/
  freecad/
  ...
docs/
  app-aware-3d-navigation-stack.md
```

## Notes
- The current upstream codebase already includes robust configurable mapping for cursor/scroll behavior and can remain the baseline fallback path.
- This design intentionally uses neutral terminology: **App-Aware 3D Navigation Stack**.
