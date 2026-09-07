Built during the 2026 iQOO Hackathon(Pune) in collaboration with @zahy294 @styxoid

# KineTrak

**6-DOF Spatial Controller & On-Device AI Gesture Copilot**

KineTrak transforms a smartphone into a sub-millimeter 6-DOF (Degrees of Freedom) spatial controller and an on-device AI gesture copilot for 3D engines such as PyOpenGL, Unity, and Unreal. Built to eliminate the friction of navigating 3D space with traditional 2D mice and keyboards, it serves as an open, accessible alternative to expensive dedicated hardware like the 3Dconnexion SpaceMouse.

The entire pipeline operates under a strict 100% offline constraint ("Red Light" rule compliant) — zero remote GPUs, no cloud dependencies, no WebSockets, and no local Wi-Fi networking. Data transport between the mobile device and the workstation is achieved via an air-gapped IPC bridge over the **Vivo Office Kit Shared Clipboard**.

---

## Table of Contents

- [System Architecture](#system-architecture)
- [Core Technical Features](#core-technical-features)
- [The Payload Contract](#the-payload-contract)
- [Monorepo Layout](#monorepo-layout)
- [Setup & Getting Started](#setup--getting-started)
- [Desktop Controls & AI State Machine](#desktop-controls--ai-state-machine)

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│ iQOO SMARTPHONE (OriginOS 6) — "Red Light" Environment                  │
│                                                                         │
│  [ ARCore Tracking Loop (60Hz) — Headless Foreground Service ]          │
│   ↳ Camera + 6-Axis IMU sensor fusion (or IMU-only fallback)            │
│   ↳ OneEuroFilter3D noise suppression                                   │
│   ↳ Downsamples coordinates to 15Hz                                     │
│   ↳ EXCLUSIVE Clipboard Writer: Formats & writes 12-field payload       │
│                                                                         │
│  [ Gesture Detection & NPU Inference — Async Worker ]                   │
│   ↳ Trigger: Physical Volume Down button (or offline keyword spotter)   │
│   ↳ Rolling 45-frame motion tensor buffer                               │
│   ↳ Qualcomm SNPE Java API running INT8 quantized .dlc on Hexagon DSP   │
│   ↳ Updates atomic state variables (pendingAction, currentState)        │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │ Vivo Office Kit Shared Clipboard (15Hz)
                                     ↓
┌─────────────────────────────────────────────────────────────────────────┐
│ LAPTOP HOST (Python 3.13 / PyOpenGL) — "Green Light" Environment        │
│                                                                         │
│  [ Clipboard Listener Thread (clipboard_hook.py) — 60Hz Polling ]       │
│   ↳ Ingests OS clipboard; drops malformed/stale frames                  │
│   ↳ Rising-Edge Action Latching: Prevents multi-trigger spam            │
│   ↳ 500ms Stale Watchdog & Auto-Restart Sequence Recovery               │
│                                                                         │
│  [ Interpolation & State Estimation (smoothing_math.py) ]               │
│   ↳ Fixed-Lag Trajectory Reconstructor (Centripetal Catmull-Rom Spline) │
│   ↳ 3-Tap FIR moving average smoothing filter                           │
│   ↳ Constant-Velocity Kalman Filter (AxisKF) + Chi-Squared Gating       │
│   ↳ Shortest-geodesic Quaternion SLERP / NLERP                          │
│                                                                         │
│  [ Main Render Loop (main.py) — 60 FPS Viewport ]                       │
│   ↳ Pygame-ce + double-buffered PyOpenGL 3D viewport                    │
│   ↳ Phone-proportioned 3D cursor with local RGB axes & ground grid      │
│   ↳ Orthographic 2D AI State Machine HUD (Idle/Record/Think/Execute)    │
└─────────────────────────────────────────────────────────────────────────┘
```

## Core Technical Features

- **Sub-Millimeter 6-DOF Tracking** — Leverages Google ARCore Visual-Inertial Odometry (VIO) and 6-axis IMU sensor fusion to stream 3D position coordinates `(X, Y, Z)` and absolute unit quaternions `(Q_W, Q_X, Q_Y, Q_Z)`.
- **Air-Gapped Telemetry Pipe** — Decimates tracking data to a stable 15Hz transmission rate across the local OS clipboard to prevent sync daemon congestion.
- **Fixed-Lag Spline Reconstruction** — Reconstructs continuous 60 FPS viewport motion from 15Hz telemetry using Centripetal Catmull-Rom Splines (α = 0.5) and a 3-tap FIR smoothing filter, eliminating stepping jitter and vertical runaway drift.
- **Edge-Triggered Action Latching** — Uses rising-edge state detection (`raw_action != "NULL" and raw_action != last_raw_action`) to execute incoming discrete commands strictly once, ignoring repetitive ticks across the 500ms latch window.
- **Snapdragon Hexagon NPU Acceleration** — Classifies physical 3D gestures from a rolling 45-frame buffer using an INT8-quantized Deep Learning Container (`gesture_model_quantized.dlc`) via the Qualcomm SNPE Java API.
- **Single-Writer Concurrency** — Isolates the clipboard by routing all UI and NPU worker states through thread-safe atomics (`AtomicBoolean`, `AtomicReference`), leaving the 15Hz tracking loop as the exclusive system clipboard writer.
- **Stale Watchdog & Auto-Recovery** — Detects transmission gaps > 500ms to trigger a Zero-Order Hold (ZOH) and resets sequence tracking seamlessly upon app restarts.

## The Payload Contract

Every sync frame transmitted across the shared clipboard adheres to a strict 12-field, pipe-delimited schema:

```
KT|[SEQ]|[STATE]|[X]|[Y]|[Z]|[QW]|[QX]|[QY]|[QZ]|[GESTURE_STATE]|[ACTION]
```

| Field | Type | Description |
| --- | --- | --- |
| `KT` | — | Protocol identification header. |
| `SEQ` | int | Monotonically increasing sequence number for ordering and packet-drop monitoring. |
| `STATE` | 0 or 1 | Tracking lock status (`1` = active tracking, `0` = tracking lost). |
| `X, Y, Z` | float | Gravity-aligned world translation coordinates in meters. |
| `QW, QX, QY, QZ` | float | Unit quaternions representing absolute 3D spatial rotation. |
| `GESTURE_STATE` | enum | Current mobile state — `NULL`, `RECORDING`, `THINKING`. |
| `ACTION` | enum | Resolved discrete command, held for ~500ms — `NULL`, `ACTION:SELECT`, `ACTION:DELETE`, `ACTION:SPAWN`, `ACTION:RESET`, `ACTION:EXPLODE`. |

## Monorepo Layout

```
kinetrak-core/
├── kinetrak-desktop/                 # Desktop Python client & visualizer ("Green Light")
│   ├── main.py                       # 60 FPS Pygame-ce / PyOpenGL viewport and HUD
│   ├── clipboard_hook.py             # 60Hz clipboard reader, edge latch, & watchdog
│   ├── smoothing_math.py             # Splines, Kalman filters, SLERP, & 1€ smoothing
│   ├── benchmark_rate.py             # Telemetry bridge rate & jitter measurement tool
│   ├── test_clipboard.py             # Automated transport & latch contract test suite
│   ├── requirements.txt              # Pinned Python dependencies
│   └── assets/                       # 3D models and geometries
│
├── kinetrak-android/                 # Android native application ("Red Light")
│   └── app/
│       ├── libs/                     # snpe-release.aar (Qualcomm SNPE Java API)
│       ├── src/main/assets/          # gesture_model_quantized.dlc (quantized model)
│       ├── src/main/java/com/ggr/kinetrak/
│       │   ├── MainActivity.kt        # Hardware key hooks (Volume Down)
│       │   ├── ArCoreHeadlessEngine    # 60Hz ARCore VIO headless pipeline
│       │   ├── ClipboardBridgeService  # 15Hz clipboard serialization loop
│       │   ├── MotionBufferManager     # 45-frame rolling tensor recorder
│       │   └── math/OneEuroFilter.kt   # On-device adaptive noise filter
│       └── build.gradle.kts           # Dependencies (ARCore, SNPE aar)
│
├── KineTrak_Design_Doc_v4.2.md       # Architectural specification
└── KineTrak_Environment_Setup.md     # Development setup guide
```

## Setup & Getting Started

### Prerequisites

- **Host workstation:** Windows 10/11 with Python 3.10–3.13 and Vivo Office Kit installed.
- **Mobile device:** OriginOS / Android flagship, paired with USB or Wireless Debugging enabled.

### 1. Desktop Workstation Setup ("Green Light")

Navigate to the desktop client directory:

```bash
cd kinetrak-desktop
```

Create and activate a Python virtual environment:

```powershell
python -m venv venv
.\venv\Scripts\Activate.ps1
```

Install dependencies:

```powershell
pip install -r requirements.txt
```

Run the contract verification test suites:

```powershell
python test_clipboard.py
python smoothing_math.py
```

Launch the 3D viewport (synthetic mode available for testing without hardware):

```powershell
# Standalone synthetic simulation
python main.py --synthetic --mode 1

# Live clipboard listening mode
python main.py --gain 2.2
```

### 2. Android Device Setup ("Red Light")

1. Place `snpe-release.aar` into `kinetrak-android/app/libs/`.
2. Place the quantized `.dlc` model into `kinetrak-android/app/src/main/assets/`.
3. Ensure app-level `build.gradle.kts` includes:

   ```kotlin
   repositories {
       flatDir { dirs("libs") }
   }
   dependencies {
       implementation(name = "snpe-release", ext = "aar")
       implementation("com.google.ar:core:1.54.0")
   }
   ```

4. Build and install the debug APK to the device via ADB:

   ```powershell
   .\gradlew.bat assembleDebug
   adb install -r app/build/outputs/apk/debug/app-debug.apk
   ```

5. Configure high background power permissions in OriginOS and launch KineTrak.

## Desktop Controls & AI State Machine

### Controls

| Key | Action |
| --- | --- |
| `SPACEBAR` / `R` | Recalibrates spatial origin (zeroes out current translational offsets to snap cursor to center). |
| `T` | Toggles the internal 15Hz synthetic telemetry generator for testing. |
| `M` | Cycles synthetic transmission rate modes (15Hz, 20Hz, 30Hz). |
| `ESC` | Safe termination of background clipboard threads and OpenGL context. |

### Dynamic Telemetry HUD

The 2D orthographic overlay transitions through four distinct operational states driven by the `GESTURE_STATE` and `ACTION` telemetry fields:

| State | HUD Ring Visual | System Activity |
| --- | --- | --- |
| **IDLE** | Cyan Ring | Continuous 1:1 6-DOF spatial cursor tracking. |
| **RECORDING** | Pulsing Yellow Ring | Volume Down held; rolling 45-frame motion buffer active. |
| **THINKING** | Spinning Purple Ring | Volume Down released; Qualcomm SNPE executing on Hexagon DSP. |
| **EXECUTION** | Expanding Green Flash | Action resolved; 3D viewport entity transforms or triggers. |
