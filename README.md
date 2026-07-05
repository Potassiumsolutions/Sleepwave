# SleepWave ✦

A Progressive Web App (PWA) for deeper sleep and gentle waking — binaural beats, healing tones, personal affirmations, and a soft crossfade alarm. Built for the **Build with Gemini xPrize** competition.

**▶️ Live app: https://potassiumsolutions.github.io/Sleepwave/**

## What it does

SleepWave walks you from lying down to waking up, all on your own device with no subscriptions, no wearables, and no internet after install. It's organized into seven tabs:

- **Guide** — a step-by-step walkthrough of the whole flow.
- **Affirm** — record or upload a personal affirmation/hypnosis track. It plays first as you drift off (optionally repeating), then gently crossfades into your background sound.
- **Sleep** — the core engine: browser-synthesized **binaural beats** (Delta/Theta/Alpha/Gamma) mixed with an ambient bed of brown noise, white noise, or rain. Or play your own custom file.
- **Tones** — a grid of **healing-frequency presets** (432 Hz, 528 Hz, 639 Hz, 741 Hz, 888 Hz, 963 Hz and more), each with its own bespoke synthesis. Tap to play; long sessions can be logged.
- **Wake** — choose how to be woken: record your own message, upload an audio file, or pick one of **6 built-in alarm tones** (beep, siren, buzzer, chirp, foghorn, reveille). A wake-message generator offers three personas — Gentle, Energetic, Mindful.
- **Alarm** — set your wake time and a ramp duration; the sleep sound fades out as the wake sound fades in.
- **History** — a log of your recent sessions (sleep start, alarm time, dismiss time, total duration).

## Highlights

- **Crossfade wake** — background audio ramps down as your wake audio ramps up over a duration you choose, so you surface gradually instead of being jolted awake.
- **Runs all night** — a silent-audio keepalive plus screen Wake Lock keep audio alive through the night on Android.
- **Fully offline & private** — recordings never leave your device; everything works after the first load.
- **Installable** — adds to your home screen and runs full-screen like a native app.

## Install (phone)

1. Open the live URL in your phone's browser.
2. **Android/Chrome:** tap the *Install* / *Add to Home Screen* banner or use the ⋮ menu.
3. **iOS/Safari:** tap Share → *Add to Home Screen*.

Grant microphone access on first launch if you want to record affirmations or wake messages.

## Files

All files are served together from the same directory over HTTPS:

| File | Purpose |
|---|---|
| `index.html` | The entire app (HTML + CSS + JS) |
| `manifest.json` | PWA install manifest |
| `sw.js` | Service worker (offline support) |
| `icon-192.png` / `icon-512.png` | Home-screen and splash icons |

## Tech stack

- Vanilla HTML / CSS / JavaScript, single file, no build step
- Web Audio API (binaural synthesis, ambient and healing-tone generation, crossfades)
- MediaRecorder API (voice recording)
- localStorage (preferences and sleep history)
- PWA (manifest + service worker), Wake Lock API

## xPrize context

SleepWave is an entry in the **Build with Gemini xPrize — Health & Human Potential** category. Sleep dysregulation is one of the most documented and least-solved health challenges globally. SleepWave addresses it with a zero-cost, device-native approach requiring no subscriptions, no wearables, and no connection after install.

---

*Built by Paul — HH Molds Inc.*
