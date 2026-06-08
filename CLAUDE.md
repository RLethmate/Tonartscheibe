# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Improscheibe** ("Harmony Wheel") is an Android music-learning app that visualizes chord relationships on a rotating wheel and detects live chords via microphone. It consists of two independent parts:

- **`app/`** — Capacitor-wrapped Android app (single HTML/CSS/JS file at `app/www/index.html`)
- **`../chord-detector/backend/main.py`** — FastAPI WebSocket server for ML-based chord detection (separate repo directory)

## Key Commands

### Android App
All commands run from `app/`:
```bash
# After editing app/www/index.html, deploy to Android build:
npx cap copy android

# Open in Android Studio (then press Run ▶):
npx cap open android
```

### Chord Detection Backend
Run from `../chord-detector/`:
```bash
# Kill any existing instance first:
lsof -ti:8002 | xargs kill -9 2>/dev/null

# Start server (Apple Silicon — ARM Python required for TF/crema):
.venv-arm/bin/python3.11 -m uvicorn backend.main:app --host 0.0.0.0 --port 8002
```

## Architecture

### App (`app/www/index.html`)
A single self-contained file (~2200 lines). Key sections by line number:

| Section | Lines | Description |
|---|---|---|
| CSS | 14–420 | All styling incl. wheel, dashboard, tutorial overlay, settings sheet |
| HTML | 421–540 | Static markup: overlays, settings sheet, wheel container, dashboard, progression container |
| Data | 541–680 | `notes[]`, `guitarChordLibrary{}`, `progressions{}`, `songs[]` |
| Wheel init | 841–900 | `initWheel()` — builds DOM segments for inner/outer wheel |
| Notation/viz | 904–1200 | `drawNotation()`, `drawGuitarChart()`, `drawPiano()` — dashboard rendering |
| Playback | 1281–1480 | `getTensionProfile()`, `playChordFromPosition()` — Web Audio API synthesis |
| Song search | 1478–1775 | `showAllSongs()`, `filterSongs()`, `renderProgressions()` |
| Main update | 1777–1810 | `updateUI()` — redraws wheel, dashboard, progressions on state change |
| Live mode | 1808–2070 | `toggleLiveMode()`, `handleLiveMessage()`, `showLiveChordInViz()`, `applyKeyToWheelDebounced()` |
| Tutorial | 1186–1265 | `TUTORIAL_STEPS[]`, `startTutorial()`, `_showTutorialStep()` |

**State variables** (globals): `currentStep` (wheel position 0–11), `currentVisualAngle`, `tensionMode`, `isLiveMode`, `liveState` (WebSocket + audio pipeline).

**Wheel mechanics**: `outerWheel` rotates via CSS transform. `currentStep` = semitone offset of root note C from top. `setWheelRotation(angle)` sets the CSS transform. The 7 inner segments are always fixed; only the outer (chromatic) ring rotates.

**Chord colors** (used everywhere consistently):
```js
colorsMajor = { tonic: "#C1E1C1", subdom: "#FDFD96", dom: "#FFB7B2", minor: "#B5EAD7", dim: "#D3D3D3" }
```

### Backend (`../chord-detector/backend/main.py`)
Single FastAPI app with one WebSocket endpoint `/ws`.

**Audio pipeline per hop:**
1. Client sends Float32 PCM frames at 16 kHz over WebSocket (binary)
2. Server accumulates frames into a sliding window (`StreamConfig.window_sec`, default 0.55s)
3. `process_hop()` runs in a thread pool (`run_in_executor`) to avoid blocking the event loop:
   - `crema_infer_chord()` — resamples 16kHz→22050Hz, runs TensorFlow crema model
   - Falls back to `dsp_detect_chords()` if crema unavailable
4. `snap_chord_to_key()` — returns `"--"` for non-diatonic chords (key confidence ≥ 0.60)
5. `estimate_key_top2_from_events()` — key tracking with weighted chord duration + recency
6. Response JSON: `{chord, confidence, key: {tonic, mode, confidence}}`

**crema dependency quirk**: Requires `scikit-learn < 1.6` (avoids `__sklearn_tags__` AttributeError in pumpp's LabelEncoder). On Apple Silicon use `.venv-arm` with ARM Python 3.11.

### Live Mode Data Flow
```
Android mic (getUserMedia, 16kHz)
  → ScriptProcessorNode (512 samples/hop)
  → Float32Array binary → WebSocket → FastAPI /ws
  → crema TF inference (run_in_executor)
  → JSON {chord, key} → WebSocket back
  → handleLiveMessage() → showLiveChordInViz() + applyKeyToWheelDebounced()
  → wheel rotates, dashboard shows chord notation + guitar/piano fingering
```

**WS URL** is hardcoded at `app/www/index.html:1812`:
```js
const LIVE_WS_URL = "ws://192.168.2.100:8002/ws";
```
Change this for cloud deployment (use `wss://` for Play Store).

### Capacitor Config
`app/capacitor.config.json` has `"allowMixedContent": true` — required for `ws://` (non-TLS) connections in the WebView. Must be removed when switching to `wss://` for Play Store.

## Notes / Gotchas

- **`npx cap copy android`** must be run after every change to `app/www/` before testing on device. Android Studio does not watch for file changes.
- **Tutorial auto-start**: Controlled by `localStorage.getItem('impro_tutorial_done_v2')`. Clearing app data or changing the key suffix resets it.
- **H vs B**: The app uses American notation throughout (`B`, not German `H`). The notes array is `["C","C#","D","D#","E","F","F#","G","G#","A","A#","B"]`.
- **Symbol PNGs** (`haus.png`, `berg_gross.png`, etc.) exist in both root (for standalone `index.html`) and `app/www/` (for the Android app) — this duplication is intentional.
