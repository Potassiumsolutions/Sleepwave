# SleepWave — Claude Onboarding

## Project Overview
A Progressive Web App (PWA) for sleep improvement and gentle waking, built for the Build with Gemini xPrize competition. Synthesizes binaural beats + ambient noise, provides a smart alarm with audio crossfade, logs sleep sessions, supports custom sleep audio files, and plays healing-frequency tone presets.

**Single-file architecture** — the entire app lives in `index.html` (CSS + JS embedded). No build step, no framework. Open directly in a browser or install as a PWA.

---

## Deployment

| URL | Purpose |
|---|---|
| `https://github.com/Potassiumsolutions/Sleepwave` | GitHub repo (code) |
| `https://potassiumsolutions.github.io/Sleepwave/` | Live installable PWA |

**To push an update:**
```powershell
cd "D:\Claude\Sleepwave"
git add index.html sw.js   # add any other changed files
git commit -m "describe what changed"
git push
```
Users already running the app will see a purple **"✦ New version available — Update"** banner at the bottom the next time they open it. Tapping Update reloads to the new version — no reinstall needed.

---

## File Map

| File | Purpose |
|---|---|
| `index.html` | Complete app — all CSS, HTML, JS (~2050 lines) |
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
4. **Tones** — Icon grid of healing-frequency presets; tap to play/stop; volume slider; fades out with alarm crossfade
5. **History** — Sleep session log rendered from localStorage, clear button

### Key Global State Variables
```
sleeping            — Boolean, true while sleep audio is running
armed               — Boolean, true while alarm is armed
currentSleepMode    — 'synth' | 'file', controls which audio path is used
sleepStartTime      — Date, set when sleep audio starts
alarmFireTime       — Date, set when alarm fires
pendingSessionStart — Date, captured at manual stop for the save-prompt flow
pendingSessionStop  — Date, captured at manual stop for the save-prompt flow
audioCtx            — Web Audio API context (sleep engine)
masterGain          — top-level gain (controls master vol; sole gain in file mode)
binauralGain        — gain for binaural oscillators (synth mode only)
ambientMaster       — gain for all noise sources (synth mode only)
brownGain, whiteGain, rainGain — per-noise gains (synth mode only)
leftOsc, rightOsc   — binaural oscillators (synth mode only)
sleepFileBlob       — File object for custom sleep audio (file mode)
sleepFileAudio      — Audio element playing the custom file
sleepFileSource     — MediaElementSourceNode routing file through Web Audio
freqAudioCtx        — Separate AudioContext for Tones tab
freqMasterGain      — Master gain for active tone preset
freqOscNodes        — Array of oscillators in active tone preset (for cleanup)
freqNoiseSrc        — Noise buffer source in active tone preset
freqSparkleTimer    — setTimeout handle for sparkle scheduler
freqMelodyTimer     — setTimeout handle for melody scheduler
freqPlayingId       — ID string of currently playing preset, or null
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
| `applySettings(s)` | Applies a settings object to all UI controls AND live audio nodes |
| `loadSettings()` | On page load: migrates legacy key, populates dropdown, auto-loads last preset |
| `armAlarm()` | Arms alarm, starts 20s interval check, pre-loads wake audio |
| `checkAlarm()` | Compares current time to alarm time (90s window) |
| `fireAlarm()` | Sets alarmFireTime, shows wake overlay, starts crossfade |
| `dismissAlarm()` | Hides overlay, restores gains, stops tone preset, logs session |
| `startWakeCrossfade()` | Fades out sleep audio AND active tone preset over ramp time |
| `logHistory(start, alarmTime, dismiss, totalMs)` | Pushes session to sw_history (max 30); alarmTime may be null |
| `renderHistory()` | Renders session cards; hides Alarm Fired row when alarmTime is null |
| `applyUpdate()` | Sends SKIP_WAITING to new SW and reloads page |
| `switchTab(name)` | Tab navigation |
| `startFreq(preset)` | Starts a tone preset — builds synthesis graph, fades in |
| `stopFreq()` | Stops active tone preset — cancels timers, fades out, closes context |
| `renderFreqPresets()` | Re-renders Tones icon grid with playing state |
| `updateFreqVol(el)` | Updates freqMasterGain from the Tones volume slider |
| `freqMakeNoise(ctx, type)` | Generates brown or white noise buffer |

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

### Audio Graph — Tones Tab (separate AudioContext)
```
[oscillators / noise] ─► [variant-specific FX chain] ─► freqMasterGain ─► destination
```
Alarm crossfade fades `freqMasterGain` to 0 over the same ramp time as the sleep audio fade.

---

## Tones Presets (FREQ_PRESETS)

| ID | Hz | Symbol | Label | Variant |
|---|---|---|---|---|
| hz888 | 888 | ∞ | Abundance & Flow | (default chorus) |
| hz999_golden | 999 | ☀ | Golden Wave | golden |
| hz432_butterfly | 432 | 🦋 | Butterfly Effect | butterfly |
| hz528 | 528 | ✦ | Transformation | (default chorus) |
| hz432 | 432 | ✿ | Natural Tuning | (default chorus) |
| hz639 | 639 | ♡ | Heart Connection | (default chorus) |
| hz741 | 741 | ◉ | Intuition | (default chorus) |
| hz963 | 963 | ✸ | Crown Frequency | (default chorus) |
| hz432_drone | 432 | ◎ | Analog Drone | drone432 |

Layout: 2-column icon grid. 9th card (odd count) is centered via `grid-column:1/-1`.

---

## Data Persistence (localStorage)

| Key | Type | Purpose |
|---|---|---|
| `sw_onboarded` | `"1"` | First-run onboarding shown flag |
| `sw_mic` | `"granted"/"denied"/"skipped"` | Microphone permission state |
| `sw_history` | JSON array | Sleep session log, max 30 entries |
| `sw_presets` | JSON object | Named settings presets `{ "Name": { ...values } }` |
| `sw_last_preset` | string | Name of last saved/loaded preset (auto-restored on load) |

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

## Android / iOS Compatibility

- **Wake lock:** `navigator.wakeLock` requested on sleep start; auto-re-acquired on release if still sleeping
- **Silent keepalive:** A silent looping base64 MP3 (`iosKeepAlive`) plays on ALL platforms to prevent audio pipeline suspension overnight — not iOS-only
- **Mic stream:** Stopped immediately after recording finishes to avoid Android entering voice-call audio mode. `getUserMedia` uses `{echoCancellation:false, noiseSuppression:false, autoGainControl:false}`
- **Web Audio unlock:** All `AudioContext` creation is gated behind user gestures

---

## File Editing Notes

- **Edit tool:** Works for small changes (< ~50 lines). Unreliable for multi-KB replacements.
- **Large replacements:** Use Node.js `fs.readFileSync`/`writeFileSync` with `{encoding:'utf8'}`, find splice points by `indexOf`, and concatenate new content.
- **PowerShell file writes:** Always use explicit UTF-8 encoding — `[System.IO.File]::ReadAllText(path, [System.Text.Encoding]::UTF8)` and `WriteAllText(path, text, [System.Text.UTF8Encoding]::new($false))`. Default encoding is Windows-1252 and will double-encode non-ASCII characters.
- **PowerShell 5.1 limits:** No `&&` pipeline chaining; no `utf8NoBOM` encoding name.

---

## Update Banner System
- `sw.js` version `sleepwave-v2` — new SW installs but waits (no `skipWaiting` in install).
- When browser detects `sw.js` changed, new SW enters "waiting" state.
- `index.html` registration code detects `waiting` → shows purple banner at bottom of screen.
- User taps **Update** → `applyUpdate()` sends `SKIP_WAITING` message → new SW activates → page reloads.
- **To trigger an update for all users:** just push any change to `index.html` or `sw.js`. The banner appears automatically on next app open.

---

## Design System
- **Colors:** Dark blue bg `#080b14`, accent purple/blue `#6c8eff` / `#a78bfa`, danger red `#f87171`
- **Fonts:** DM Serif Display (headings), DM Sans (body)
- **Mobile-first**, max-width 480px, rounded cards (14px radius)
- Status dots: `.dot` elements, add `.on` class to light them green
- Button classes: `.btn-primary` (gradient), `.btn-secondary` (surface), `.btn-danger` (red tint), `.btn-sm` (auto width)
- Overlays: `position:fixed;inset:0;z-index:100+` with `.show` class toggling `display:flex`
- Tones grid: `.freq-grid` (2-col), `.freq-card`, `.freq-card.playing` (glows with `--fcol`)

---

## Running / Testing
No build step — open `index.html` directly in Chrome or install as PWA. For quick iteration, just refresh the browser tab.

**DevTools inspection:**
- Application → Local Storage → inspect `sw_history`, `sw_presets`, `sw_last_preset`
- Application → Service Workers → check SW state, force update
- Console → SW logs prefixed with `[SleepWave SW]`
