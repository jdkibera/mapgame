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

## 2026-04-19 — HUD sketched borders: repeated failure (DO NOT retry the same approach)

I spent **six or seven commits** trying to give the HUD cards (control strip + input + title + score) the same hand-drawn wobble as the cartouche plaque, using a CSS pseudo-element with `filter: url(#hudSketch)`. Every attempt visibly broke the bottom edge of the short control strip card (~40px tall). The user had to repeat "just draw the goddamn border" multiple times across many images before I stopped. I made the same underlying mistake at least four times, tuning one knob at a time (filter region, scale, frequency, border width, box-shadow) instead of fixing the root cause or backing out of the approach.

**Root cause (understood, repeatedly):** `feDisplacementMap` resamples the SourceGraphic by ±scale px. A CSS-bordered `::before` has a thin stroke with transparent interior. When displacement points perpendicular to the stroke and magnitude exceeds (border-width / 2), the sample lands in transparent interior → output is transparent → **hole in the border**. On short cards the noise pattern produces long correlated runs along the bottom edge, so holes stack into a missing-side.

**What I kept retrying and why each try failed:**
- `border: 1.3px + scale 2.2–3.0`: guaranteed holes, scale ≫ border/2.
- `border: 1.3px + tighter filter region`: filter region only constrains *where output can appear*; it doesn't fix the transparent-interior sampling. Cosmetic change, same holes.
- Thicker pseudo border + unchanged scale: still holes whenever scale > border/2.
- Removing the downward-offset `box-shadow`: that was a separate bug (shadow bleeding into the 22px gap between cards). Worth fixing, but didn't fix the bottom-edge holes.
- Matching the cartouche's `baseFrequency 0.085, scale 1.6` on a thin CSS border: the cartouche works because every SVG shape in `.decorations` is filtered *together*, so nearby shapes overlap and fill in displacement gaps. A lone thin CSS border has nothing to fall back on.

**What I should have done from turn 1:**
- Recognise that a CSS pseudo-border + displacement filter cannot reliably produce a closed thin-line rectangle; either
  - ship a plain `border: 1.5px solid` and call it done, or
  - render the border as an actual SVG `<rect>` (real stroke, continuous path) inside each card and filter *that* element. SVG strokes have no transparent interior to sample from — the cartouche uses this.
- When a user says the same thing three times in a row, **stop tuning and change architecture**. Every "quick tweak" in this thread cost a full turn and pushed a broken commit live.

**If you ever need to re-add sketched HUD borders:** do it with an inline `<svg>` child per card (or a JS pass that adds one) drawing a stroked `<rect x="1" y="1" width="calc(100%-2)" height="calc(100%-2)" fill="none" stroke="currentColor" stroke-width="1.5" filter="url(#hudSketch)"/>`. Do NOT revive `.hud::before { border; filter }`.

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

## Root cause of the "Chrome: music plays, greeting silent" bug

Turned out to be **two separate issues stacked on top of each other**. Takes a restart to shake one of them loose.

### Layer 1 — picker preferred a flaky network voice (fixed in code)

`pickBritishVoice` explicitly listed `Google UK English Female` in its preferred-name regex. Chrome exposes Google's full network-voice catalogue (`localService: false`) and the picker always chose that one first. Network TTS voices silently drop utterances when their fetch stalls — no `onstart`, no `onerror`, just nothing.

**Fix** (commit `bd44a55`): pure deletion — drop `Google UK English Female`, `Google US English`, and `Google Australian English Female` from the preferred-name regexes. Picker falls through to Samantha (local) on macOS.

Related: never set `utter.lang = "en-GB"` as a fallback when no voice is picked — Chrome silently picks the Google voice anyway via the lang fallback.

### Layer 2 — Chrome's TTS engine can wedge per-process

Even with the picker returning Samantha correctly, Chrome can sit in a state where `speechSynthesis.speak()` accepts the utterance but fires neither `onstart` nor `onerror` and produces no audio. Web Audio (Tone.js music) keeps working — it's a separate pipeline. No amount of code change fixes this; the engine is stuck inside Chrome.

**Fix**: restart Chrome. On the user's machine, the greeting worked perfectly on first try after a cold restart.

Next time the symptom recurs ("picker returns a local voice, log shows no lifecycle events fire, music still plays"), the fastest path is a Chrome restart before touching any code.

## How to test the voice flow without a live browser

`/tmp/mapgame-test/` has a Puppeteer harness that:
- Instruments `SpeechSynthesis.prototype.speak` to log lifecycle events
- Simulates the user's 199-voice Chrome (Samantha, Daniel, Karen + Google voices) via a `getVoices()` stub
- Runs the actual Start-click flow end-to-end

Useful because Puppeteer's bundled Chrome has only local voices — it won't reproduce the bug on its own, but the stub lets you verify the picker returns a local voice regardless of what Chrome hands you. Re-run after any change to `pickBritishVoice` or `speakBritish`.

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
8cf9144 Fix PLAYER_HUES rgba spacing — colours weren't reaching the CSS
bd44a55 Drop Google network voices from pickBritishVoice regexes   ← the actual fix
6d2b4ad Revert voice subsystem to 1b97bcf baseline; restore search suggest
1b97bcf Clean rebuild of welcome → map voice flow
83451c1 Restore original Start-handler + speakBritish shape verbatim
c2fbd91 Rip out the stacked speech/mic workarounds
c551afa Voice mode default on, British greeting on Start   ← the original
```
