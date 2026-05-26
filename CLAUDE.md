# SleepWave 6 — Claude Onboarding

## Project Overview
A Progressive Web App (PWA) for sleep improvement and gentle waking, built for the Build with Gemini xPrize competition. Synthesizes binaural beats + ambient noise, provides a smart alarm with audio crossfade, logs sleep sessions, and supports custom sleep audio files.

**Single-file architecture** — the entire app lives in `index.html` (CSS + JS embedded). No build step, no framework. Open directly in a browser or install as a PWA.

---

## Deployment

| URL | Purpose |
|---|---|
| `https://github.com/Potassiumsolutions/Sleepwave` | GitHub repo (code) |
| `https://potassiumsolutions.github.io/Sleepwave/` | Live installable PWA |

**To push an update:**
```powershell
cd "C:\Claude\Sleepwave 6"
git add index.html sw.js   # add any other changed files
git commit -m "describe what changed"
git push
```
Users already running the app will see a purple **"✦ New version available — Update"** banner at the bottom the next time they open it. Tapping Update reloads to the new version — no reinstall needed.

---

## File Map

| File | Purpose |
|---|---|
| `index.html` | Complete app — all CSS, HTML, JS (~1650 lines) |
| `manifest.json` | PWA manifest (name, icons, start URL, display mode) |
| `sw.js` | Service Worker — cache-first, version `sleepwave-v2`, shows update banner |
| `CLAUDE.md` | This file — onboarding context for Claude sessions |
| `icon-192.png` / `icon-512.png` | PWA icons |
| `archive/` | Older versions — do not touch |

---

## App Architecture

### Tabs
1. **Sleep** — Sleep Audio Source toggle (Synth / Custom File) → Binaural engine + ambient mixer (Synth mode) or file picker (Custom File mode) → Master Volume → Start/Stop button → Audio Status → Settings Presets
2. **Wake** — Wake audio source: microphone recording / file upload / AI script
3. **Alarm** — Alarm time input, arm/disarm, countdown, test wake sequence, alarm volume
4. **History** — Sleep session log rendered from localStorage, clear button

### Key Global State Variables
```
sleeping            — Boolean, true while sleep audio is running
armed               — Boolean, true while alarm is armed
currentSleepMode    — 'synth' | 'file', controls which audio path is used
sleepStartTime      — Date, set when sleep audio starts
alarmFireTime       — Date, set when alarm fires
pendingSessionStart — Date, captured at manual stop for the save-prompt flow
pendingSessionStop  — Date, captured at manual stop for the save-prompt flow
audioCtx            — Web Audio API context
masterGain          — top-level gain (controls master vol; sole gain in file mode)
binauralGain        — gain for binaural oscillators (synth mode only)
ambientMaster       — gain for all noise sources (synth mode only)
brownGain, whiteGain, rainGain — per-noise gains (synth mode only)
leftOsc, rightOsc   — binaural oscillators (synth mode only)
sleepFileBlob       — File object for custom sleep audio (file mode)
sleepFileAudio      — Audio element playing the custom file
sleepFileSource     — MediaElementSourceNode routing file through Web Audio
```

### Key Functions
| Function | What it does |
|---|---|
| `toggleSleep()` | Start/Stop sleep audio; forks on `currentSleepMode` |
| `initAudio()` | Builds full binaural+ambient Web Audio graph (synth mode) |
| `initFileAudio()` | Creates AudioContext + routes file blob through masterGain (file mode) |
| `stopAllAudio()` | Stops file audio OR fades masterGain to 0 and closes AudioContext |
| `setSleepMode(mode, el)` | Toggles Synth/Custom File UI; blocked while sleeping |
| `handleSleepFile(input)` | Stores file blob, shows filename + preview player |
| `saveManualSession()` | Logs pending session to history and hides save prompt |
| `discardManualSession()` | Dismisses save prompt without logging |
| `savePreset()` | Saves all current slider/toggle values under a named preset |
| `loadPreset()` | Applies the selected preset from the dropdown |
| `deletePreset()` | Removes a preset after confirmation |
| `renderPresetSelect(selected)` | Rebuilds the preset dropdown from localStorage |
| `applySettings(s)` | Applies a settings object to all UI controls |
| `loadSettings()` | On page load: migrates legacy key, populates dropdown, auto-loads last preset |
| `armAlarm()` | Arms alarm, starts 20s interval check, pre-loads wake audio |
| `checkAlarm()` | Compares current time to alarm time (90s window) |
| `fireAlarm()` | Sets alarmFireTime, shows wake overlay, starts crossfade |
| `dismissAlarm()` | Hides overlay, restores gains, logs session to history |
| `startWakeCrossfade()` | Fades out sleep audio (synth gains OR masterGain in file mode) |
| `logHistory(start, alarmTime, dismiss, totalMs)` | Pushes session to sw_history (max 30); alarmTime may be null |
| `renderHistory()` | Renders session cards; hides Alarm Fired row when alarmTime is null |
| `applyUpdate()` | Sends SKIP_WAITING to new SW and reloads page |
| `switchTab(name)` | Tab navigation |

### Audio Graph — Synth Mode
```
leftOsc  ─┐
           ├─ binauralGain ─┐
rightOsc ─┘                 ├─ masterGain ─► destination
brownSrc ─┐                 │
whiteSrc ──┼─ ambientMaster ─┘
rainSrc  ─┘
```

### Audio Graph — Custom File Mode
```
sleepFileBlob ─► Audio element ─► MediaElementSource ─► masterGain ─► destination
```

---

## Data Persistence (localStorage)

| Key | Type | Purpose |
|---|---|---|
| `sw_onboarded` | `"1"` | First-run onboarding shown flag |
| `sw_mic` | `"granted"/"denied"/"skipped"` | Microphone permission state |
| `sw_history` | JSON array | Sleep session log, max 30 entries |
| `sw_presets` | JSON object | Named settings presets `{ "Name": { ...values } }` |
| `sw_last_preset` | string | Name of last saved/loaded preset (auto-restored on load) |

### Session Object Shape
```javascript
{
  date: "5/26/2026",
  startTime: "10:45 PM",
  alarmTime: "11:12 PM",   // null when no alarm used (manual stop)
  dismissTime: "11:15 PM", // labeled "Stopped" in UI when alarmTime is null
  totalMs: 1800000
}
```

### Preset Object Shape
```javascript
{
  beat: "delta",       // delta | theta | alpha | gamma
  carrier: "200",      // Hz, string from slider
  binauralVol: "60",
  brownVol: "40",
  whiteVol: "0",
  rainVol: "30",
  masterVol: "60",
  rampMins: "8",
  alarmVol: "80"
}
```

---

## Sleep Logging Behavior
- **Alarm path:** `dismissAlarm()` always logs — no prompt, dismiss is the explicit action. `alarmTime` is set, label reads "Dismissed".
- **Manual stop path:** `toggleSleep()` else-branch captures `stopTime`, shows `#sleepLogPrompt` overlay. User taps Save → `saveManualSession()` calls `logHistory()` with `null` alarmTime. Label reads "Stopped" in history.
- Both paths share `logHistory()` which null-safely formats all timestamps.

---

## Settings Presets UI (Sleep Tab)
- **Name input + Save** — type a name, tap Save. Saves all 9 settings values.
- **Dropdown + Load** — lists all presets alphabetically. Pick and tap Load.
- **Delete** — removes selected preset after confirm dialog.
- On page load, the last used preset is auto-restored.
- Legacy `sw_settings` key (single preset) is auto-migrated to a preset named "Default" on first load.

---

## Custom File Sleep Mode
- Toggle at top of Sleep Tab: **Synth** (default) | **Custom File**
- Switching blocked while audio is playing.
- In Custom File mode: Binaural Engine + Ambient Mixer cards are hidden. A file picker card appears accepting any audio format (MP3, WAV, M4A, OGG).
- File loops via `Audio` element routed through Web Audio so Master Volume slider still works.
- Alarm crossfade works in file mode — `startWakeCrossfade()` fades `masterGain` directly.
- After alarm dismiss, `masterGain` is restored to the Master Volume slider value.

---

## Update Banner System
- `sw.js` version `sleepwave-v2` — new SW installs but waits (no `skipWaiting` in install).
- When browser detects `sw.js` changed, new SW enters "waiting" state.
- `index.html` registration code detects `waiting` → shows purple banner at bottom of screen.
- User taps **Update** → `applyUpdate()` sends `SKIP_WAITING` message → new SW activates → page reloads.
- **To trigger an update for all users:** just push any change to `index.html` or `sw.js`. The banner appears automatically on next app open.

---

## iOS Compatibility Notes
- Web Audio API must be unlocked via a user gesture — `initAudio()` and `initFileAudio()` are always triggered from click handlers.
- Wake audio element is pre-loaded during `armAlarm()` (user gesture) so `play()` works later from a timer.
- `startIOSKeepAlive()` / `stopIOSKeepAlive()` play a silent looping base64 MP3 to prevent app suspension.

---

## Design System
- **Colors:** Dark blue bg `#080b14`, accent purple/blue `#6c8eff` / `#a78bfa`, danger red `#f87171`
- **Fonts:** DM Serif Display (headings), DM Sans (body)
- **Mobile-first**, max-width 480px, rounded cards (14px radius)
- Status dots: `.dot` elements, add `.on` class to light them green
- Button classes: `.btn-primary` (gradient), `.btn-secondary` (surface), `.btn-danger` (red tint), `.btn-sm` (auto width)
- Overlays: `position:fixed;inset:0;z-index:100+` with `.show` class toggling `display:flex`

---

## Running / Testing
No build step — open `index.html` directly in Chrome or install as PWA. For quick iteration, just refresh the browser tab.

**DevTools inspection:**
- Application → Local Storage → inspect `sw_history`, `sw_presets`, `sw_last_preset`
- Application → Service Workers → check SW state, force update
- Console → SW logs prefixed with `[SleepWave SW]`
