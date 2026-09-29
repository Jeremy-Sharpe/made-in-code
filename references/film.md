# The long-form animated explainer film

A playbook for a 15-30 minute narrated, fully animated film where every frame is drawn in code:
SVG, CSS and GSAP, rendered frame by frame. No image models, no stock illustration, no screen
captures. Distilled from a real build: a ~23 minute linocut-storybook adaptation of Gregory
Gundersen's essay "The Hierarchy of Money", told as a campfire fable.

**Install the craft skill first.** This playbook sits on top of `product-launch-motion`
(MIT, by AbubakrChan: https://github.com/AbubakrChan/product-launch-motion). It owns the
laws (word-locked sync, determinism, two-node camera, sound is arithmetic, verify the
delivered file), the HyperFrames renderer contract, the traps list and the master script.
Read its SKILL.md and references before you start. This file only covers what changes when
the film is 20+ minutes long, character-driven and illustrated entirely in code.

## 1. Calibration numbers

| Measure | The real build |
|---|---|
| Runtime | ~23.5 min picture (1,410 s), 22:52 of narration |
| Scenes | 27, each 35-70 s, 8-24 beats (15-20 typical) |
| Script | 3,221 spoken words, ~150 wpm delivered |
| Code | ~7,700 lines of shared art/film library JS, ~12,800 lines of scene HTML |
| Cast | 24 rigged puppet characters, ~40 props, ~20 scenery elements and full-frame sets |
| Agents | ~7 parallel scene builders plus toolkit, voice, music and pipeline agents |
| Wall clock | ~14 h from brief to all 27 scenes rendered (including a 2-3 h voice stall), then a polish day for the hook, fixes and thumbnails |
| Model cost | roughly US$490 API-equivalent of Opus usage for the whole session; TTS fitted in a free tier |
| Render | ~18-20 fps per scene on a 10-core laptop; 2 concurrent renders ~26 fps aggregate; mastering ~1x realtime; ~12 GB of scene MP4s at CRF 16 |

## 2. The pipeline end to end

```
brief -> DIRECTION.md -> SCRIPT.md -> toolkit (art library) -> voice per scene
      -> word timings per scene -> one composition per scene (parallel builders)
      -> render per scene -> concat -> music/fire/SFX mix -> master -> thumbnails
```

### 2.1 Brief
One paragraph from the person: source, register ("history podcast, campfire"), length,
audience goal. Flag the rights question up front (section 6).

### 2.2 DIRECTION.md (one page)
Follow the upstream skill's "three directions, kill two" rule, with the long-form test
added: **will this look still be interesting at minute 18?** Shadow puppets were killed
because silhouettes cannot carry numbers, ledgers or colour-coded currencies for 20
minutes. A bright isometric toy village was killed because every edu channel looks like
that. The winner, a firelit linocut storybook, matched the voice register (a fable is
carved and printed), looked like no one else's explainer, and gave a **semantic colour
code for free**: the only saturated colours are the "stones", one per currency, so colour
IS the argument. Write that rule down and never break it.

Write down: the one claim in two sentences; palette as hex tokens (including the night
surround); type, vendored locally; texture (paper fibre, ink grain, off-register plates,
a deterministic grade); sound (bed, ambience, foley list, where the silences go); and:
- **Motion personality**: here, puppets step "on twos" (12 drawings/s) while the camera
  glides. Entrances are "printed in" (a plate presses: scale 1.04 to 1 with an
  off-register settle) or "carved in" (a wipe along a gouge path). Nothing bounces.
- **Two signature moves** that recur and grow; long films need recurrence, not one trick.
  THE LEDGER: every financial idea resolves into one carved two-column ledger. THE CRANE:
  each new layer of money cranes the camera up a storey, and the finale pulls back to the
  whole tower. Ration them: cranes happened only at four scripted moments.

### 2.3 SCRIPT.md format
Each scene is one block. One block = one composition = one voice file.

```
## s05 · The bank

VO:
Still, most shops wouldn't lend to the rancher. They didn't know him. ...

PICTURE: Split stage: the rancher left, the widow and her jar right, a gap between them.
The entrepreneur steps into the gap; stones flow widow to him to rancher; 5 in, 3 out,
2 stay with him and glow. A carved sign is hung: BANK.

MARGIN: (optional) small on-screen footnote with real-world history. Never spoken.
```

Rules that paid off:
- Only `VO:` text is spoken (the voice script parses it). An **approved-figures list** at
  the top names the only numerals allowed on screen.
- Write for the ear, and budget words, not minutes: at ~145-150 wpm, 3,000 words is ~21
  minutes. The 3,441-word first draft lost a minor section and four scenes' fat to reach 3,221.
- **Scene ids are folder names** (`05-bank`); give the canonical list to the voice agent
  and builders before either starts.
- **Lead with the claim.** The first cold open was 47 s of campfire atmosphere. It became a
  35 s hook stating the surprising claim in sentence one ("The money in your bank account
  isn't there"), a fast visual per clause, and only then the campfire.

### 2.4 Voice
Audition before committing. Every candidate reads the same ~60-word passage; measure
median F0 (depth), F0 range in semitones (expressiveness; under ~3 semitones drones over
20 minutes), words per minute, and word error rate against the text. Claude cannot hear
the samples: **the person picks the voice by ear**; you rank on numbers.

Learnt: instruction-style TTS ("speak slowly, unhurried") barely moves pace (every
candidate ran 160-170 wpm), so you need a real rate control; and one model silently
**dropped the last sentence**, so transcribe every take and diff it against the script.
What we used: **Azure AI Speech**, the Andrew HD voice (the `en-US-AndrewMultilingualNeural`
family), with full **SSML**: slowed with `<prosody rate="-18%" pitch="-4%">`, plus exact
`<break>` control for the silences before each crane; key in an env var. Slowing in SSML
beats time-stretching in post. Azure's free tier (F0) gives 0.5 million TTS characters a
month, but only for standard neural voices: HD voices need a paid tier.

Options (checked late September 2026; confirm before you rely on any row):

| Option | Needs key? | Pace + pause control | Word timestamps? | Licence or cost | When to pick it |
|---|---|---|---|---|---|
| Azure AI Speech, HD voice (what we used) | Yes | SSML `<prosody rate>` and `<break time>`: exact | Speech SDK word-boundary events (not checked for HD voices) | Paid; F0 free tier excludes HD | Long narration where pauses must land on exact beats |
| ElevenLabs Eleven v4 (`eleven_v4`, released 28 Sep 2026; `eleven_v4_turbo` for real time) | Yes | No SSML `<break>` on v4; `speed` setting 0.7-1.2, pauses via audio tags (`[pause]`), ellipses and punctuation | Character-level, via the with-timestamps endpoint (v4 support not confirmed) | Paid per character (launch discount at time of writing); free-plan limits not verified | Most expressive read; voice cloning from ~10 s of audio |
| Kokoro-82M (local) | No | `speed` argument; pauses via punctuation or silence you splice in | No | Apache-2.0, commercial use OK; runs on CPU or Apple silicon | No key, fast, fixed built-in voices |
| Chatterbox / Chatterbox-Turbo (local, Resemble AI) | No | `exaggeration` and `cfg_weight` knobs; paralinguistic tags on Turbo; no documented rate control | No | MIT; output carries an imperceptible watermark | No key and you want to clone a voice from a reference clip |

Without exact rate control (ElevenLabs v4, the local models), generate at natural pace,
insert pauses as spliced digital silence, and stretch only as a last resort.

If the provider returns word (or character) timestamps, you can skip or cross-check the
local alignment in 2.5; the script is still ground truth.

Per-scene voice script (`voice.py sNN`): read the scene's `VO:` text; one TTS request
(key in an env var, raw take cached and gitignored; **retry dropped streams**, trap 1);
insert pauses as SSML breaks after exact script phrases, then **verify** each in the audio
and top up with digital silence if short; trim to 0.15 s before the first word and 0.30 s
after the last; normalise to -18 LUFS, true peak <= -3 dBTP; bake a **lead-in** (3.0 s for
chapter-card scenes); write `scenes/<id>/vo.wav` (48 kHz mono) and `cues.json`.

### 2.5 Word timings (all local)
- Transcribe locally with Whisper (whisper.cpp, or mlx-whisper on Apple silicon) with
  word timestamps. No hosted ASR.
- **The script is ground truth, not the transcript.** Whisper merges and mishears words
  ("gray stone" becomes "greystone"). The robust version force-aligns the script text to
  the audio (a local CTC aligner), snaps word edges to energy onsets, and uses Whisper only
  to flag differences. When two recognisers both heard "sheep" for "sheet", the take was
  re-recorded.
- `cues.json`: `{"words":[{"w","start","end"}...], "cues":{"pause1": 21.3}}`. Named cues
  survive re-voicing.

### 2.6 One composition per scene
A long film is **27 independent HyperFrames compositions**, each with its own paused GSAP
timeline and `vo.wav`, rendered separately and joined: parallel building and rendering,
cheap single-scene re-renders, no 23-minute timeline in one tab. Per-scene contract,
enforced by a staging script that fails loudly:
- `#root data-duration` on the 30 fps grid and >= VO length plus ~0.5-1 s tail.
- `<audio id="vo" src="vo.wav" data-duration="<exact wav length>">`; every audio tag
  needs an `id` or it renders silent.
- Every tween position comes from `word("phrase", n)` or `cue("name")`, which **throw**
  when the word is missing, so a bad cue fails the gate instead of firing at t=0. Use the
  occurrence number: repeated words ("much", "economy") otherwise hit the first instance.
- Continuity passes through `PLAN.md` files (positions, seeds, final camera framing) so
  scene N+1 opens on scene N's last frame. The finale reused six earlier scenes' sets.

### 2.7 Parallel builders
Build the shared toolkit **before** briefing builders and prove it in the real renderer
(a kit-test scene that renders twice to identical frames). Then fan out one builder per
chapter (2-5 scenes) on one written brief (section 4). Builders start as soon as their
scenes are voiced; placeholder timings are fine for drafts, never for the final render.
Coordination: a **2-slot render lock** (a `mkdir` lock with stale-lock recovery; more
concurrent renders gain nothing); builders commit only their own folders; builders never
edit shared `lib/` but report bugs to one library agent, followed by a re-render pass.

### 2.8 Render, concat, mix, master
- Render each scene at CRF 16 and write its audio as a WAV padded or trimmed to **exactly
  frames x 1600 samples** (48 kHz / 30 fps, frame count read from the rendered video).
  Concat copies the video streams and joins the **WAVs** sample-exactly; never concat AAC
  segments (each adds priming delay and the VO drifts).
- Music and ambience are **film-level**, never in scenes: a cue sheet maps one track per
  scene group, anchored to scene-relative cues or words (so re-renders move them), with
  3 s equal-power crossfades, phrase-matched loop points, and dips before each crane.
  Duck ~6 dB under narration keyed from `vo.wav` only, so foley never pumps it.
- Master with the upstream two-pass loudnorm script to -14 LUFS / -1 dBTP, after a
  true-peak limiter so loudnorm stays linear. Write a JSON mix report every run (VO,
  music under speech, music in pauses, crossfades, loop joins).

### 2.9 Thumbnails, drawn with the same toolkit
HTML pages importing the same art library, screenshotted headlessly at 1920x1080 and
downsampled (lanczos) to 1280x720, plus a 320x180 copy to judge legibility as a phone feed
shows it. Make five options and a contact sheet; the midpoint "thesis diagram" frame is a
strong candidate.

## 3. The code-drawn art toolkit

A shared library of plain ES modules (no build step) returning **SVG markup strings**,
everything seeded (no `Math.random`, no `Date`): `ink.js` (PRNG, noise, filter defs, gouge/hatch generators,
`carve()`, `carvedText()`); `props.js` (stones, piles, banknotes, ledger, scales, chains,
with animatable parts as `<g data-part="lid" data-pivot="x y">`, where the pivot is exactly
GSAP's `svgOrigin`); `scenery.js` (elements, plus full-frame sets returned as
`{ far, mid, near }` for parallax layers); `puppet.js` + `cast.js`; `film/` (chapter cards,
margin notes, key-word presses, the ledger, the tower/crane, transitions, counting
numerals). Every module has a review page rendered to PNG that you and the person look at
before any builder touches it.

### 3.1 Ink: rough edge, speckled voids, roller mottle
One filter on a **group** (never per path), so edges roughen in scene pixels:

```js
// Carved ink: displace the edge, then multiply alpha by voids x mottle.
const inkFilter = (id, freq = 0.05, scale = 2.6, seed = 5) => `
<filter id="${id}" x="-3%" y="-3%" width="106%" height="106%" color-interpolation-filters="sRGB">
  <feTurbulence type="fractalNoise" baseFrequency="${freq}" numOctaves="3" seed="${seed}" result="w"/>
  <feDisplacementMap in="SourceGraphic" in2="w" scale="${scale}" xChannelSelector="R" yChannelSelector="G" result="rough"/>
  <feTurbulence type="fractalNoise" baseFrequency="0.26" numOctaves="3" seed="${seed + 17}" result="n"/>
  <feColorMatrix in="n" type="matrix" values="0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 -14 0 0 0 11.1" result="voids"/>
  <feTurbulence type="fractalNoise" baseFrequency="0.012 0.05" numOctaves="2" seed="${seed + 31}" result="m"/>
  <feColorMatrix in="m" type="matrix" values="0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0.9 0 0 0.62" result="mottle"/>
  <feComposite in="voids" in2="mottle" operator="arithmetic" k1="1" k2="0" k3="0" k4="0" result="a"/>
  <feComposite in="rough" in2="a" operator="in"/>
</filter>`;
// Make s/m/l strengths, plus a near-solid "night" variant for full-frame dark masses
// (normal speckle on a whole night sky reads as snow).
```

### 3.2 Gouges and off-register plates
A **gouge** is one tapered sliver path (swelling belly, pointed ends). A **hatch** is a run
of gouges along guide polylines (parallel, concentric, comb teeth for grass, rays for
glows, flow-field streamlines) clipped to a shape, with width driven by a tone field
(0 paper to 1 ink): that is how shading is carved. One run = one `<path>`. Spot colour is
its own plate under the ink, wobbled differently and offset 1-3 px:

```js
// Seeded PRNG (mulberry32) + an off-register plate under an ink key block. Never Math.random.
const rng = s => () => { s |= 0; s = s + 0x6D2B79F5 | 0; let t = Math.imul(s ^ s >>> 15, 1 | s);
  t = t + Math.imul(t ^ t >>> 7, 61 | t) ^ t; return ((t ^ t >>> 14) >>> 0) / 4294967296; };
function carvedStone(d, colour, seed) {
  const r = rng(seed), off = () => (1 + 2 * r()) * (r() < 0.5 ? -1 : 1);
  return `<g><path d="${d}" fill="${colour}" filter="url(#plate)" transform="translate(${off()} ${off()})"/>
    <path d="${d}" fill="none" stroke="#1D1916" stroke-width="7" filter="url(#ink-m)"/></g>`;
}
```

Paper is a fixed-seed `feTurbulence` tile used as a **CSS background image**, so it
rasterises once and costs nothing under camera moves.

### 3.3 The grade: deterministic firelight
Anything that "breathes" is a pure function of the playhead:

```js
function hash(n) { const s = Math.sin(n * 127.1 + 311.7) * 43758.5453; return s - Math.floor(s); }
function vnoise(x) { const i = Math.floor(x), f = x - i, u = f * f * (3 - 2 * f); return hash(i) * (1 - u) + hash(i + 1) * u; }
function flicker(t) { // [-1, 1], same t -> same value, every render
  const v = 0.34 * Math.sin(t * 2 * Math.PI * 0.53 + 0.7) + 0.22 * Math.sin(t * 2 * Math.PI * 1.37 + 2.1)
          + 0.12 * Math.sin(t * 2 * Math.PI * 3.1 + 4.0) + 0.32 * (vnoise(t * 7) * 2 - 1);
  return Math.max(-1, Math.min(1, v));
}
```

Drive per-frame hooks from a **property setter**, not `onUpdate`: renderers and probes
`seek()` the timeline, and some seeks suppress callbacks, but property writes always run.

```js
const subs = [], clock = { _t: 0 };   // subs: functions of t
Object.defineProperty(clock, "t", { get() { return this._t; }, set(v) { this._t = v; subs.forEach(f => f(v)); } });
tl.to(clock, { t: DURATION, duration: DURATION, ease: "none" }, 0);
// subs: grade opacity from flicker(t), grain re-seeded at 12 Hz from floor(t * 12), vignette.
// Grade with plain alpha layers only: mix-blend-mode renders as a wash.
```

### 3.4 Stepping on twos
Puppets change drawing on a **global** 1/12 s grid, so every character steps on the same
beats; the camera never steps.

```js
// An ease that holds the tween's time on the global 1/12 s grid. `at` MUST equal the tween's position.
function stepEase(ease, at, dur) {
  const base = gsap.parseEase(ease || "none");
  return p => {
    if (p <= 0) return base(0); if (p >= 1) return base(1);
    const q = Math.min(dur, Math.max(0, Math.floor((at + p * dur) * 12 + 1e-6) / 12 - at));
    return base(q / dur);
  };
}
tl.to("#stone", { x: 400, duration: 1.6, ease: stepEase("power2.out", at, 1.6) }, at);
```

### 3.5 The puppet rig
3/4-profile cut-out characters, ~200 units tall with feet on y = 0, built as nested SVG
groups with pivots baked in (`hips > torso > neck > head`, `torso > shoulderR > upperArmR >
forearmR > handR`, `hips > thighR > shinR > footR`; R is the near side). Never write joint
transforms directly. Keep one **state object** in anatomical degrees
(`shR, elR, thR, knR, anR, torso, neck, bob, x, y, face, expr, handR`) with clamped
ranges so limbs cannot break, and an `apply()` that writes every joint from state. Every
action is a stepped tween of a private proxy whose setter updates state:

```js
function segment(tl, P, at, dur, fn) {           // fn(u) mutates P.state for u in 0..1
  dur = Math.max(dur, 1 / 12);
  let val = 0; const proxy = {};
  Object.defineProperty(proxy, "u", { get: () => val, set: v => { val = v; fn(v); P.apply(); } });
  tl.fromTo(proxy, { u: 0 }, { u: 1, duration: dur, immediateRender: false, lazy: false,
                               ease: stepEase("power1.inOut", at, dur) }, at);
}
// Actions built on it: pose(name|state), walk({to}), nod, shake, point/give({target}) with
// 2-bone IK, count, write, panic, expr. Each returns its end time so calls chain.
```

What made the puppets act rather than slide: a **kinematic walk** (stance foot IK-pinned
to the floor, heel-off, toe-lift, bob, counter-swing; check it with successive drawings
overlaid at true positions); named poses plus expressions on story beats; **props in hand
slots** that counter-rotate to stay upright; a **costume spec** (build, nose, chin, hair,
headwear, garments, feet, accessories) so 24 characters come from data; one ink filter
per puppet root so roughness stays in scene pixels at any scale. Review the cast as a
grid, scale-3 close-ups, every pose and a walk strip before builders use it. Add each
puppet's actions **in time order** and never overlap two actions on the same key, or
seeking becomes order-dependent.

## 4. Scene-builder brief template

Every builder agent got the same written brief. Adapt it:

```
You are building scenes <ids> of a <N> min animated film, "<title>": <one-line register>,
all art drawn in code. The aim is a film people watch to the end and share.

Read first, in order: DIRECTION.md; PIPELINE.md (build contract, helpers, determinism
rules, traps: obey exactly); the art library README (use it, do not redraw what it
provides; extend only in your scene's art/); the film components README; SCRIPT.md for
your scenes AND the scenes either side (continuity); the craft skill's motion-grammar,
camera and traps references.

Hard rules: no image models, bitmaps or other APIs (code-drawn SVG/CSS only). Every
reveal cued with word()/cue() from the real cues.json, never hand-typed times.
Deterministic, one owner per property. <Semantic colour rule>; approved figures only.
Touch only your own scene folders; report shared-helper bugs and work around them locally.

What good looks like:
- Something meaningful moves on every sentence; a still frame over ~3 s while the voice
  talks is a defect. No idle wiggling either.
- Puppets act: walk on, turn, point, hand things over, react; faces change on beats.
- The camera is a storyteller: push-ins on emotional beats, pans that follow an object,
  far/mid/near parallax in every exterior. Signature moves only where the script says.
- Text is sparse (one to three carved words, printed in when defined); numbers count and
  stack physically.
- First frame follows the previous scene; last frame resolves and holds ~0.5 s.
- Foley only, as static <audio> tags cued to words, modest levels. No music or ambience.

Process per scene:
1. PLAN.md: a beat table (VO phrase -> cue word -> picture -> camera), 8-20 beats.
2. Build from the demo scene; gate with stage + lint/check at 0 errors.
3. Review: stills or a draft every ~4 s, LOOK at them, write what is wrong, fix; at least
   two rounds. Static too long? Reveals on the word? Puppets good at this scale? Text
   legible at 1080p? Carved, not vector?
4. Final render; sample 6 frames from the delivered MP4; commit only your files.

Report back (concise, no file dumps): per scene the duration, beat count, strongest
moment, weakest remaining thing, 3 frame paths, and any shared-library bug hit.
```

The "weakest remaining thing" field became the polish list: builders are honest when asked.

## 5. Traps and lessons

Symptom -> cause -> fix, from the real build; read the upstream traps too.

**Pipeline and agents**

1. **Builders idle for hours.** Voice batch died at scene 9 of 27 on a dropped network read
   with no retry; the agent reported "still running". Fix: retry with backoff in the TTS
   call, per-scene progress log, coordinator verifies files exist rather than trusting status.
2. **An agent stalled out after 600 s with no progress.** It sat in a foreground
   `until ...; do sleep 5; done` loop waiting for 17 re-renders. Fix: run long renders in the
   background and let completion notify you; never block an agent on a long wait loop.
3. **Model-made art crept in.** The first plan used an image model for characters; the
   person said "generate it with code". Fix: stop that agent and **move its outputs out of
   the repo** so no builder picks them up. Code-drawn puppets were better anyway: joints.
4. **Parallel commits collide** (`index.lock`). Fix: `git commit -- <own paths>` in a
   short retry loop.
5. **A scene was re-voiced after it was built.** Only its durations changed, because every
   reveal was cued to a word. This is why word cues, not seconds, are law. Commit `vo.wav`
   and `cues.json` with the scene; they were once left uncommitted while everything
   depended on them.

**Rendering and determinism**

6. **Script-created audio rendered silent.** HyperFrames compiles the `<audio>` list from
   static HTML before scripts run. Fix: declare foley as static tags; a tool builds the scene
   headlessly and writes the tags; a stale tag fails the gate.
7. **Probe and tag tools timed out (even at 300 s).** They waited for `load`, which never fires
   on a heavy scene with dozens of long WAVs. Fix: navigate to `domcontentloaded`, then await
   fonts with a 15 s cap.
8. **Grain and flicker frozen in probe stills** (renders were fine). The per-frame hook hung
   off `onUpdate`, which `tl.seek()` suppresses. Fix: the setter clock in 3.3.
9. **Sun in the wrong place after scrubbing back.** Build-time state did not equal the
    tween's t=0 state, and pose calls were added out of time order. Fix: one writer per
    property, t=0 reproduces build state, actions in time order; prove with "t=5 cold equals
    t=5 after seeking to 20 and back" (identical PNG). Related: a module scene is safe only
    because it runs before `DOMContentLoaded`; no top-level `await`, `import()` or `onload`
    that adds tweens.
10. **Camera pushes over a live set were 2.5x slower** (8 fps vs 21). Chrome re-rasterises
    every filtered group at each new scale. Fix: `will-change: transform` on static,
    frame-sized set layers only (back to 18-19 fps). Baking the set into an SVG `<img>` gave
    no gain.
11. **The finale's pull-back lost its title, rays and ground.** `will-change` on the camera
    node: at ~0.13 zoom Chrome silently drops a layer whose raster outgrows its tile budget.
    Fix: never on the camera or anything holding the whole world; keep it on set layers at
    zooms of roughly 0.3-2x.
12. **Hidden shots leaked into other shots.** Reveal helpers set `visibility: visible`, which
    shows through a hidden parent. Fix: reveals set `visibility: inherit`; hide shots with
    opacity as well; never hide a container you later attach revealed content to.
13. **Fifty stones took 4.2 s to drop.** Each drop was rounded up to one 1/12 s drawing. Fix:
    batch several items per drawing when the interval is under 1/12 s.
14. **A crane "settled" but a strip of the lower storey's sky lingered.** A power3 ease's long
    tail plus parallax. Fix: shorter crane (2.6 s not 3.5 s) and name the destination mid-move.

**Type**

15. **"500" read as "soo", "150" as "1ſ".** The period face had only old-style figures. Fix:
    a digits-only `@font-face` from a lining-figure face, declared after the main face so it
    wins inside its range, size-adjusted to the cap height (Libre Caslon Text under IM Fell):
    `@font-face { font-family: "Period Face"; unicode-range: U+0030-0039; size-adjust: 86%; src: url(/assets/fonts/Lining.woff2); }`
16. **The digit fix worked in preview, not in the render.** The renderer injects a full-range
    Google Fonts copy for any family named in the page without a local `@font-face` in the
    page itself, and it beat the digit face. Fix: the stager inlines every `@font-face` into
    the staged page. No network in renders.
17. **Blank text on some frames.** Faces load lazily; text first laid out mid-render, or a
    worker's first frame on a reveal, can paint blank. Fix: `document.fonts.load()` every face
    (letters and digits) at startup plus a hidden warmer line in each face.

**Picture and art**

18. **A CSS `fill` on an ink class turned outline hills into solid ink.** Fix: ink classes set
    only `filter`; colour comes from the element.
19. **Frame-filling lantern rays overwhelmed a night room.** Fix: a calmer default halo, the
    old burst kept as an option so already-rendered scenes can still reproduce it.
20. **A decree plate's text overflowed**, and the next scene copied the plate for a matched
    cut. Fix it in both scenes; matched cuts couple scenes, so note them in PLAN.md.
21. **Five delegates were one elder cloned five times.** Fix: distinct costume specs per
    character. A cast grid review catches this before scenes do.
22. **`../` in an asset URL is a lint error; a symlinked `index.html` fails the runtime
    check.** Stage by copying files and linking directories; use root-absolute URLs.

**Sound**

23. **Music pumped between sentences** (rose to ~4 dB under the voice in every short gap).
    Fix: treat VO gaps under 1.5 s as speech in the ducker key; 1.2 s release; bed at
    -23 LUFS pre-master. Measured result: music in short gaps within 0.2 dB of music under
    speech; recovered pauses ~6 dB under the narrator.
24. **First mix peaked at 0.0 dBFS.** Fire-crackle transients set the film's peaks. Fix: cap
    the ambience 12 dB under full scale (loudness unchanged) and limit the sum before loudnorm.
25. **Foley too quiet to matter** (a chain snap at -23 dB against VO at -12). A cue's volume
    cannot rescue a quiet asset: level the asset, then measure it in the delivered file.
26. **A dip silently did nothing.** Its cue name did not exist in the scene. Fix: the mixer
    prints a warning per missing cue; fix the scene's `cues.json`, not the cue sheet.
27. **Nobody had listened.** All numbers came from meters. Before release a human listens to
    loop joins, crossfades and the overall balance; the mix report lists every join time.

## 6. Licensing hygiene

- **Only CC0 / public domain and CC-BY.** No NC, no ND, nothing "personal use only". Music
  was nine CC-BY 4.0 library tracks (incompetech.com); foley was 13 CC0 Freesound
  recordings and one CC-BY crowd walla. Fonts: OFL, vendored with their licence files.
- Keep untouched originals in `assets/*/originals/` and note any processed derivative.
- `assets/LICENCES.md`: per file the title, author, source URL, licence and whether
  modified, plus a **paste-ready attribution block** for the video description (list CC0
  items too). Popular CC-BY tracks can draw third-party Content ID claims even when
  credited; keep the record for disputes.
- **Derivative-work permission:** a close adaptation of someone's essay needs the author's
  permission before a monetised release. Ask early; credit them on screen and in the
  description regardless.

## 7. Where this departs from product-launch-motion

The upstream skill targets 20-90 s launch films. At 23 minutes we added: one composition
per scene joined as sample-exact WAVs; a film-level soundtrack from a cue sheet; forced
alignment of the script as the timing source; a shared code-drawn art library in place of
the "truth pass" of real product captures (the approved-figures list survives); puppets on
twos; parallel builders under one brief with a render lock; two recurring signature moves;
spoken-never margin footnotes. Kept unchanged: three directions kill two, word-locked sync,
determinism, the two-node camera, "green gates are not enough", verify the delivered file,
and the master script.
