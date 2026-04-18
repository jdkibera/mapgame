# Session Log

Ongoing record of what's been built, what's fragile, and lessons learned so the next agent (or a future us) doesn't relearn them.

## Current state

Single-file app at `index.html`. Deployed via GitHub Pages at `jdkibera.github.io/mapgame`. No build step.

### Features

- **D3 + topojson world map**, Natural Earth projection, pan + zoom.
- **12 progressive levels** (Home Waters → Here Be Dragons), 15 countries per round, region filter, flag mode, study mode.
- **Scoring**: 10/7/4 points per country; 5/3/1 capital bonus.
- **Ocean targets**: 5 major oceans are part of the pool. Names are hidden by default; the target ocean shows a pulsing copper marker at its center.
- **Auto-zoom** (toggleable) softly zooms on small countries.
- **Voice mode** (default on): Web Speech API recognition, continuous; British female TTS for a welcome greeting.
- **Multiplayer** (solo or up to 5 players): rotates turns per country; per-level + overall winners; per-player pastel hues with a halo around the input for the active player.
- **Mobile layout**: compact HUD, hidden toolbar with `⋯` menu, portrait rotate-phone hint, welcome overlay scrolls.
- **Cache-busting** meta headers so GitHub Pages updates reach clients reliably.

### The voice/mic flow (current, post-rebuild — commit 1b97bcf)

Welcome screen:
- Heading defaults to `"Welcome! Enter Your Name"`.
- Name input placeholder: `"Your name"`. No `"Tallulah"` prefill.
- Begin Voyage button is **disabled** until solo has 1 name or multi has ≥2 names.
- First non-empty name triggers `requestMicPermission()` — a single `getUserMedia({audio:true})` whose stream tracks are stopped immediately. Gets Chrome's permission dialog off the map page.
- Switching to Multiplayer carries a typed solo name into Player 1.

Start click (linear, no `await` before the speech):
1. Build `state.players` / `state.mode`
2. `speakBritish(greeting)` — synchronous, inside the user gesture (iOS/Safari requirement)
3. `state.suppressListening = true`; cleared after 5s, then `startListening()`
4. Hide welcome, `pickNext()`
5. `audioInit().then(startAmbient)` in the background

`speakBritish` is the minimal one-liner: `cancel()` → new utterance → `speak()`. No `onend`, no `resume`, no nudges.

## Lessons learned (DO NOT redo)

The voice mode / mic permission layer is where I caused the most damage. Things that sounded reasonable but made it worse:

- **Silent-prime utterance** (`speak(" ", volume:0)`): Intended to "warm up" Chrome's TTS. Actually consumed the user-gesture token; the real greeting dropped silently.
- **Unconditional `speechSynthesis.cancel()` at the top of every speak**: Same — spent the gesture token on Chrome. If you need to cancel, check `synth.speaking || synth.pending` first.
- **`speechSynthesis.resume()` on every call**: Chrome sometimes stalls the queue when you resume() on an unpaused synth.
- **10s pause/resume nudge interval for long utterances**: Our utterances are < 5s. The interval fired before speech even started and reset the engine.
- **Holding a persistent `getUserMedia` stream** to suppress Chrome's mic pill: Ended up competing with SpeechRecognition's own mic stream and delaying TTS by ~10s.
- **`onEnd` callback on speakBritish** used to gate `startListening`: Fine in principle but every timing bug manifested as the mic eating the greeting. The 5s fixed timer is simpler and reliable enough.
- **`primeWelcome()` doing both getUserMedia AND a silent-prime utterance**: Double-booked the gesture token; neither worked.
- **Moving `speakBritish` AFTER `await audioInit()`** was originally fine (c551afa did that), but once `audioInit` gets blocked by Tone.start() waiting on audio context activation, the speak never fires. Safer: speak first, audioInit in background.

Each of those was a patch on top of the previous patch. By the end, the flow had 10+ moving pieces and none of them worked. The rebuild at 1b97bcf is ~80 lines of linear code that does the same thing; that's the shape to keep.

## Known-fragile / open

- **Chrome desktop TTS delay**: if the current minimal flow still stalls on Chrome, the next thing to look at is `audioInit()` — specifically `Tone.start()` and whether it holds onto the audio context in a way that blocks speech. Possibly just speak before calling audioInit at all and never await it.
- **iOS Kate/Serena voices** aren't installed by default on macOS. `pickBritishVoice()` falls back to Samantha (US female) before Daniel (UK male). To get a real British female, the user has to download a voice via System Settings → Accessibility → Spoken Content → System Voice → Customize.
- **iOS Chrome "Microphone access allowed" pill**: with `continuous: true` on SpeechRecognition it should only show once; if it still pops each turn, the fix is tricky — avoid pre-grabbing a persistent `getUserMedia` stream (it made things worse last time).
- **Portrait-mobile map rotation**: had one before, removed because resize/rotation was unreliable. Current flow shows a small "Rotate your phone" hint instead.
- **localStorage `we_muted`**: if the user clicked Sound Off in a previous session, `Audio.muted=true` silences speech too. The Sound button mutes everything (SFX + ambient + speech). Intentional, but easy to forget while debugging.

## Conventions / repo

- Commits should use heredoc'd messages and include the `Co-Authored-By: Claude ...` trailer.
- Branch: `main`. Push directly; no PRs.
- Cache-Control meta headers are in the `<head>`; hard-refresh still needed once after an update.
- No build step, no tests. Change `index.html`, commit, push.

## Recent commit trail (most relevant to voice/mic)

```
1b97bcf Clean rebuild of welcome → map voice flow
83451c1 Restore original Start-handler + speakBritish shape verbatim
c2fbd91 Rip out the stacked speech/mic workarounds
a4a6294 Warm up speechSynthesis on welcome + stop mic from eating its own greeting
69a185b Prime mic permission on first welcome-screen interaction
23289b0 Prefer female voices + Chrome speechSynthesis resume/nudge
19c0d63 Stop consuming the user-gesture token before the welcome greeting
4c8ca5d Prime speech + hold mic stream on Start click for mobile
50bb222 Continuous speech recognition + speak before async for iOS
c551afa Voice mode default on, British greeting on Start   ← the working baseline
```
