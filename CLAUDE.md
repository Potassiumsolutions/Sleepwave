# SleepWave 6 — Claude Onboarding

## Project Overview
A Progressive Web App (PWA) for sleep improvement and gentle waking, built for the Build with Gemini xPrize competition. Synthesizes binaural beats + ambient noise, provides a smart alarm with audio crossfade, and logs sleep sessions.

**Single-file architecture** — the entire app lives in `index.html` (CSS + JS embedded). No build step, no framework. Open directly in a browser or install as a PWA.

---

## File Map

| File | Purpose |
|---|---|
| `index.html` | Complete app — all CSS, HTML, and JS in one file (~1350 lines) |
| `manifest.json` | PWA manifest (name, icons, start URL, display mode) |
| `sw.js` | Service Worker — cache-first offline support |
| `icon-192.png` / `icon-512.png` | PWA icons |
| `archive/` | Older versions — do not touch |

---

## App Architecture

### Tabs
1. **Sleep** — Binaural beat engine + ambient mixer + master volume + Start/Stop button
2. **Wake** — Wake audio source selection (microphone recording / file upload / AI script)
3. **Alarm** — Alarm time input, arm/disarm, countdown, test wake sequence
4. **History** — Sleep session log rendered from localStorage

### Key Global State Variables (top of `<script>` block)
```
sleeping       — Boolean, true while sleep audio is running
armed          — Boolean, true while alarm is armed
sleepStartTime — Date object, set when sleep audio starts
alarmFireTime  — Date object, set when alarm fires
audioCtx       — Web Audio API context
masterGain, binauralGain, ambientMaster — gain nodes
brownGain, whiteGain, rainGain — per-noise gain nodes
leftOsc, rightOsc — binaural oscillators
```

### Key Functions
| Function | Location | What it does |
|---|---|---|
| `toggleSleep()` | ~line 708 | Start/Stop sleep audio; sets `sleepStartTime` on start |
| `stopAllAudio()` | ~line 752 | Fades master gain to 0, closes AudioContext |
| `initAudio()` | ~line 630 | Builds entire Web Audio graph from scratch |
| `armAlarm()` | ~line 827 | Arms alarm, starts 20s interval check |
| `checkAlarm()` | ~line 855 | Compares current time to alarm time (90s window) |
| `fireAlarm()` | ~line 873 | Triggers crossfade, shows wake overlay |
| `dismissAlarm()` | ~line 950 | Hides overlay, logs session to history, clears timestamps |
| `logSession(obj)` | ~line 1000 | Pushes session object to `sw_history` in localStorage (max 30) |
| `renderHistory()` | ~line 1020 | Reads `sw_history` and renders cards in History tab |
| `switchTab(name)` | ~line 580 | Tab navigation |

### Audio Graph
```
leftOsc  ─┐
           ├─ binauralGain ─┐
rightOsc ─┘                 │
                             ├─ masterGain ─► destination
brownSrc ─┐                 │
whiteSrc ──┼─ ambientMaster ─┘
rainSrc  ─┘
```

---

## Data Persistence (localStorage)

| Key | Type | Purpose |
|---|---|---|
| `sw_onboarded` | `"1"` | First-run onboarding shown flag |
| `sw_mic` | `"granted"/"denied"/"skipped"` | Microphone permission state |
| `sw_history` | JSON array | Sleep session log, max 30 entries |
| `sw_settings` | JSON object | *(to be added)* Saved slider/settings values |

### Session Object Shape
```javascript
{
  date: "5/26/2026",
  startTime: "10:45 PM",
  alarmTime: "11:12 PM",   // null if no alarm used
  dismissTime: "11:15 PM", // null if manually stopped
  totalMs: 1800000
}
```

---

## Current Logging Behavior
- Sessions are logged **only** when the alarm fires and user dismisses it (`dismissAlarm()` calls `logSession()`).
- If a user starts/stops sleep audio manually without using the alarm, **nothing is logged** (this is the gap we are filling next).

---

## iOS Compatibility Notes
- Web Audio API must be unlocked via a user gesture — `initAudio()` is always triggered from a click handler.
- Wake audio element is pre-loaded during `armAlarm()` (user gesture) so `play()` works later from a timer.
- `startIOSKeepAlive()` / `stopIOSKeepAlive()` play a silent looping base64 MP3 to prevent app suspension.

---

## Design System
- **Colors:** Dark blue bg `#080b14`, accent purple/blue `#6c8eff` / `#a78bfa`
- **Fonts:** DM Serif Display (headings), DM Sans (body)
- **Mobile-first**, max-width 480px, rounded cards (14px radius)
- Status indicators: green-glowing `.dot` elements (add `.on` class to light them up)

---

## Next Changes (Backlog)

### 1. Manual-stop sleep log prompt
When the user clicks **Stop Sleep Audio** without having used the alarm, show a dialog asking "Save this session to your sleep log?" with Save / Discard options. On Save, call `logSession()` with start time, no alarm time, and manual stop time.

**Touch point:** `toggleSleep()` — the `else` branch (line ~739). After `stopAllAudio()`, check if `sleepStartTime` is set and no alarm was used, then show prompt.

### 2. Alarm-dismiss sleep log (verify & improve)
Alarm-based logging already exists in `dismissAlarm()`. Verify the session object is complete and correct. No prompt needed here — alarm dismiss is an explicit user action.

### 3. Save Settings button
Persist the current state of all sliders and toggles to `sw_settings` in localStorage so they restore on next app load. Button should live on the Sleep tab (near the other controls). On load, read `sw_settings` and apply values before rendering.

**Settings to persist:** beat type, carrier frequency, binaural volume, brown/white/rain noise levels, master volume, wake volume ramp duration, alarm volume.

---

## Running / Testing
No build step — open `index.html` directly in Chrome or install as PWA. For quick iteration, just refresh the browser tab. Use DevTools → Application → Local Storage to inspect `sw_history` and `sw_settings`.
