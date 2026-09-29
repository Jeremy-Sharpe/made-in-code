# Browser game: a 3D boss fight made entirely in code

A playbook distilled from building a Souls-like boss fight (three.js + Vite + Web Audio) in one Claude Code session from a single loose prompt. Every mesh, skeleton, animation, texture, note of music and sound effect is generated at load time. There are no image, model or audio files in the repo.

Live example: https://ashen-throne-game.vercel.app (keyboard and mouse; `?skip=1` jumps straight into the fight).

## Calibration numbers

| | |
|---|---|
| Wall-clock build | about 5.5 hours from first prompt to "finished": scaffold 10 min, six parallel builders 1 to 2.3 h each, one integration and polish agent about 3 h |
| Human input | one creative prompt, then one piece of feedback ("increase the hit box") |
| Code | about 19,000 lines in `src/`, 1,800 in test scripts, 1,600 in per-module dev viewers |
| Agent cost | 250k to 500k tokens per builder agent |
| Bundle | about 1.05 MB of JS, about 300 KB gzipped; one runtime dependency (`three`) |
| Load | title paints instantly; build plus shader warm-up 2.2 to 4.8 s on an Apple M4 |
| Frame rate | 60 fps locked at 1920x1080, pixel ratio 1.0 (p99 17.7 ms); auto-resolution settles at 1.25 |
| Draw calls | knight 5, boss 6 (five skinned meshes on one 62-bone skeleton plus instanced chains), world about 70 |

## Architecture

### Contracts first, then parallel builders

The single most important move. Before any real code, the coordinator wrote `docs/ARCHITECTURE.md` and a working **stub** for every module, then launched one agent per module with strict file ownership in the same checkout.

The contract doc pins down:
- **Units and axes**: metres, seconds, radians, Y up, every model faces +Z at rotation 0, forward = `(sin(yaw), 0, cos(yaw))`.
- **An ownership table**: one owner per file, "never edit another owner's files", and "you may add to the exported API, never remove or rename".
- **Each module's factory and surface**, for example:

```js
// Contract: boss model. Gameplay codes only against this.
export function createBoss(): Boss
Boss = {
  root: THREE.Group,            // origin at feet, faces +Z, ~9 m tall
  animator: { play(name, opts), update(dt), current, time, duration, done },
  clips: { [name]: { duration, loop, events: { ...seconds } } },
  getHurtSpheres(): Array<{ center, radius, part }>,   // for sword hits
  getPoint(name, out),          // 'handR', 'mouth', 'tail', ...
  setGlow(v), flashHit(), update(dt),
}
```

- **Animation timing tables**: every clip's duration and event times (`hitStart`, `hitEnd`, `recover`, `iStart`, `iEnd`) are fixed by contract. The model agent authors poses to hit those times; the gameplay agent reads the same numbers for hit windows and i-frames. Neither waits for the other.

```
| clip   | loop | duration | events (s)                             |
| roll   | no   | 0.75     | iStart 0.06, iEnd 0.42, recover 0.62   |
| slam   | no   | 2.2      | impact 0.95, recover 1.9               |
```

Also copy the tables into code (`core/contracts.js`) so gameplay can fall back to them if a model's own table is missing an event.

Modules and owners: gameplay (loop, states, player, camera, boss AI, mechanics, combat, HUD), knight model, boss model, world plus renderer, fx, audio. Each non-gameplay owner also built a **dev viewer page** (`dev/knight.html`, `dev/boss.html`, `dev/fx.html`, ...) with URL params to freeze any moment (`?clip=slam&t=0.95&cam=hero`). These viewers are how each agent checked its own work with screenshots, and they stay useful forever.

### Folder layout

```
src/
  core/      contracts.js, animator.js, input.js, params.js (URL flags), math.js
  game/      game.js (states, flow, warm-up), fallbacks.js, debug.js
  player/    player.js (controller), knightModel.js, knight/ (rig, pose, clips, armour, cloth, textures)
  boss/      boss.js (AI), mechanics.js (raid mechanics), bossModel.js, model/ (skeleton, body, clips, poseBaker)
  combat/    shapes.js (circle, arc, ring, beam tests), combat.js
  camera/    cameraRig.js (orbit, lock-on, collision, shake)
  world/     castle.js, architecture.js, textures.js, props.js, sky.js, fire.js, particles.js, colliders.js
  render/    renderer.js (HDR pass, bloom, ACES, grade, auto-resolution)
  fx/        effects.js, particles.js, decals.js, trail.js, lights.js
  audio/     engine.js, reverb.js, instruments.js, score.js, sfx.js, system.js
dev/         one viewer page per module
scripts/     shot.mjs, playtest.mjs, bot.mjs, bot-brain.js, perf.mjs, browser.mjs
```

### Game loop

```js
const game = new Game(canvas, await loadModels());
await game.warmup();                 // compile every shader behind the title screen (see traps)
window.__game = game.api();          // scripting surface for every test script

const timer = new THREE.Timer();
function loop(ts) {
  timer.update(ts);
  const dt = params.fixed ? 1 / 60 : Math.min(Math.max(timer.getDelta(), 0), 1 / 20);
  if (game.manualStep) game.R.render(game.scene, game.camera, 0);  // tests step the sim themselves
  else game.frame(dt);
  requestAnimationFrame(loop);
}
requestAnimationFrame(loop);
```

Clamp `dt` at both ends: the first rAF timestamp can precede the timer's start and give a negative delta (that produced a NaN player position).

Expose a debug API on `window.__game`: `advance(sec)` (fixed-step simulation without rendering), `trigger('slam')` (force any move or mechanic), `setBossHP(frac)`, `press(action)`, plus URL flags `?skip=1 ?phase=2 ?god=1 ?fixed=1 ?mute=1 ?debug=1 ?pr=1`. Every test script is built on these.

### Combat state machine

The player is a small state machine: `move | roll | attack | drink | hit | knockdown | dead`. All timing comes from clip events, never from separate timers:

```js
isInvulnerable() {
  const e = this.ev;                                   // clip clock: now, prev, ev(name), within(a, b)
  if (this.state === 'roll') return e.within(e.ev('iStart'), e.ev('iEnd'));
  if (this.state === 'knockdown') return e.now < e.ev('recover') + 0.1;  // grace past the get-up
  return this.state === 'dead';
}
// leaving a state: allow early exit after `recover` if the player is steering, else wait for the clip to end
if (e.now >= e.ev('recover') && input.mag > 0.1 && !input.peekBuffered()) this.state = 'move';
```

Other details that made it feel like the genre: inputs buffered for 0.3 s; stamina gates sprint, roll and attack and regen stops while spending; attacks turn only slightly during windup; hitstop 65 ms (90 ms on heavies); sword hits sampled between frames (fast swings still connect) and one hit per swing; damage-over-time ticks never flinch the player (otherwise stun-lock).

The boss AI picks moves by distance, angle, phase and cooldowns, **tracks during windup then commits** (so late rolls work), staggers after enough damage in a short window, and gets 1.2x speed in the final phase. Phases trigger at HP thresholds with a transition (invulnerable roar, shockwave, raid-warning text, sky change). Raid mechanics (meteor circles, expanding fire ring, a mark that drops and explodes, rotating beams) are separate from melee and always show a ground telegraph for their whole danger area first.

## Procedural everything

### Characters: rig, pose functions, baked clips

Both characters use the same pattern:
1. Build a `THREE.Bone` hierarchy from a table of `[name, parent, restOffset]`.
2. Model the body from primitives (lathes, tubes, shell patches, extruded profiles), assign each piece rigidly to one bone (two-bone blends only at joints), and merge **one `SkinnedMesh` per material**. That is how a detailed armoured knight costs 5 draw calls.
3. Author each animation as a **pose function of time**: torso angles plus IK targets for hands and feet. Solve IK, then bake to a `THREE.AnimationClip` at 30 to 60 fps once at load.
4. Play through a shared `Animator` (thin wrapper on `AnimationMixer` with `play(name, {fade})`, `.time`, `.done`).

```js
const BONES = [
  ['hips', null, [0, 1.0, 0]], ['spine', 'hips', [0, 0.1, 0]], ['chest', 'spine', [0, 0.2, 0]],
  ['head', 'chest', [0, 0.31, 0]],
  ['upperArmR', 'chest', [-0.2, 0.17, 0]], ['lowerArmR', 'upperArmR', [0, -0.29, 0]],
  ['thighR', 'hips', [-0.1, -0.06, 0]], ['shinR', 'thighR', [0, -0.44, 0]],
  // ... mirror for L
];
function buildRig() {
  const root = new THREE.Group(), bones = {};
  for (const [name, parent, off] of BONES) {
    const b = new THREE.Bone(); b.name = name; b.position.fromArray(off);
    (parent ? bones[parent] : root).add(b); bones[name] = b;
  }
  return { root, bones, list: Object.values(bones) };
}
// Bake a pose function into a clip. sample(t) sets bone rotations (via your IK / euler helpers).
function bake(rig, name, duration, sample, fps = 60) {
  const n = Math.ceil(duration * fps) + 1, times = new Float32Array(n);
  const qs = rig.list.map(() => new Float32Array(n * 4));
  for (let i = 0; i < n; i++) {
    times[i] = (duration * i) / (n - 1);
    sample(times[i]);
    rig.list.forEach((b, k) => {
      const q = b.quaternion.clone();
      if (i && q.dot(new THREE.Quaternion().fromArray(qs[k], (i - 1) * 4)) < 0) q.set(-q.x, -q.y, -q.z, -q.w); // keep hemisphere
      q.toArray(qs[k], i * 4);
    });
  }
  const tracks = rig.list.map((b, k) => new THREE.QuaternionKeyframeTrack(`${b.name}.quaternion`, times, qs[k]));
  return new THREE.AnimationClip(name, duration, tracks);
}
```

The quaternion hemisphere flip matters: without it, interpolation between baked keys takes the long way round and limbs spin.

Secondary motion is cheap and sells the result: a Verlet cloth cape and tabard colliding with body capsules, Verlet chains on the boss's shackles, a tail that lags when the root turns. Locomotion clips are authored so feet stay planted at exactly the gameplay speed (walk cycle duration matches metres per second), so there is no foot slide.

**Validate animation numerically, not only visually.** The knight agent wrote a Node-side checker that bakes every clip and reports IK targets beyond limb reach, any foot below the floor, and the blade tip's path during each hit window. It caught hit windows sweeping 250 degrees, overhead chops driving the tip into the floor and a knockdown foot 38 cm underground. The boss agent checked contract numbers the same way (slam hand 5.2 m ahead and on the ground at `impact`, mouth 6.5 m up and 22 degrees down during breath).

**Filmstrip screenshots**: the viewer renders several copies of the character frozen at successive times of one clip in a single orthographic shot, hit-window frames tinted red, sword segment drawn as a line. One image per clip is enough for the agent to judge the motion.

### Materials and textures

- Textures painted on canvases or `DataTexture`s at load: flagstones, ashlar, banners, steel scratches, chainmail, leather, cloth folds.
- The boss's magma cracks are a shader: 3D Voronoi edges computed in bind-pose space (so cracks stick to the body as it animates), with emission scaled by a `setGlow(0..2)` uniform that rises per phase.
- Architecture: swept profiles (piers, arches, ribs) merged into one draw call per material. Instanced rubble.
- A small generated environment map makes metal read as metal regardless of arena lighting.

### Environment and VFX

- Sky dome shader (dying sun, halo, drifting ash clouds, an eclipse for the last phase), GPU-animated ash and ember particles, light shafts, height fog patched into world materials.
- Post: multisampled HDR pass, bloom on emissives only, ACES tone mapping, then a grade pass (vignette, grain, chromatic fringe, split toning, `screenFlash`).
- FX: 8 pooled particle systems with hard caps (one draw call each, nothing allocated per frame), pooled meshes for telegraphs, decals recycled oldest-first, every handle's `remove()` safe to call twice. A leak test spawns 6,400 effects in two passes and checks geometries, textures and programs don't grow.

### Audio: a synthesised orchestra and SFX

Buses (music, sfx, ambience, UI) feed one shared reverb so everything sits in the same hall. The master chain is compressor, limiter, soft clipper. Positioned sounds use HRTF panners with per-sound reference distances. Voices are capped per sound and globally (40), oldest stolen first.

The reverb impulse is generated: about 30 short diffuse bursts in the first 120 ms, then exponentially decaying noise through a lowpass whose cutoff falls over time (highs die first, like stone and air).

Every SFX is layered from a few primitives (a pitched thump, a filtered noise burst, inharmonic partials, crackle) with a little random jitter on every play so repeats never sound identical:

```js
function hitSfx(ctx, out, t, pitch = 1) {
  const j = pitch * (0.92 + Math.random() * 0.16);
  // thump: sine sweeping down, fast decay
  const o = ctx.createOscillator(), og = ctx.createGain();
  o.frequency.setValueAtTime(140 * j, t);
  o.frequency.exponentialRampToValueAtTime(45 * j, t + 0.07);
  og.gain.setValueAtTime(0.8, t); og.gain.exponentialRampToValueAtTime(0.001, t + 0.25);
  o.connect(og).connect(out); o.start(t); o.stop(t + 0.3);
  // crack: short bandpassed noise burst
  const n = ctx.createBufferSource(); n.buffer = noiseBuffer(ctx);   // 1 s of white noise, built once
  const bp = new BiquadFilterNode(ctx, { type: 'bandpass', frequency: 2500, Q: 0.7 });
  const ng = ctx.createGain(); ng.gain.setValueAtTime(0.5, t); ng.gain.exponentialRampToValueAtTime(0.001, t + 0.08);
  n.connect(bp).connect(ng).connect(out); n.start(t); n.stop(t + 0.1);
  // ring: a few inharmonic partials for metal
  [2150, 3320, 4710].forEach((f, i) => {
    const p = ctx.createOscillator(), g = ctx.createGain();
    p.frequency.value = f * j; g.gain.setValueAtTime(0.08 / (i + 1), t);
    g.gain.exponentialRampToValueAtTime(0.001, t + 0.15);
    p.connect(g).connect(out); p.start(t); p.stop(t + 0.2);
  });
}
```

Instruments are the same idea at a larger scale: strings and brass are detuned saws through filters with attack/release envelopes; the choir is detuned saws with vibrato through a vowel formant filter bank; taiko and timpani are pitched thumps plus noise; bells are inharmonic partial stacks. The roar is a voice model: falling pitch curve, morphing vowel formants, rough modulation, distortion.

Music runs on a **look-ahead sequencer**: a timer fires every ~25 ms and schedules every note that starts in the next ~0.15 s at exact `AudioContext` times. Never schedule with `setTimeout` directly: it jitters audibly.

```js
class Sequencer {
  constructor(ctx, bpm, bars) { Object.assign(this, { ctx, beat: 60 / bpm, bars, bar: 0, next: ctx.currentTime + 0.1 }); }
  pump(ahead = 0.15) {                                // call from setInterval(..., 25)
    const until = this.ctx.currentTime + ahead;
    while (this.next < until) {
      const notes = this.bars[this.bar % this.bars.length];   // [{ beat, midi, len, inst }]
      for (const n of notes) n.inst(this.ctx, this.next + n.beat * this.beat, n.midi, n.len * this.beat);
      this.next += 4 * this.beat; this.bar++;
    }
  }
}
```

One boss theme was written at three intensities (118, 124, 134 bpm, the last a semitone up); phase changes land on the next bar line and add layers, while victory and death cut immediately. The title, ambient, victory and death pieces are separate.

**Verify audio without ears**: render through `OfflineAudioContext` to WAV, then check peaks, RMS/LUFS, NaNs, DC offset, loop seam discontinuities, and look at spectrograms of each stem. Pump long offline renders incrementally with `suspend()`/`resume()` exactly like the live scheduler, or allocating every node up front crashes the renderer. Measurement is necessary but not sufficient: get a human to listen before calling the mix done.

## The testing loop (no human playing)

All scripts use Playwright against the Vite dev server and read or drive the game through `window.__game`.

### Real GPU in headless Chromium

The default headless renderer (SwiftShader) is a CPU rasteriser: fine for "does it render", useless for performance and painfully slow for a full 3D scene. Launch flags:

```js
export function launchOptions({ gpu = false, uncapped = false } = {}) {
  const args = ['--ignore-gpu-blocklist', '--autoplay-policy=no-user-gesture-required'];
  if (gpu) args.push('--use-angle=metal', '--enable-gpu', '--enable-gpu-rasterization');   // Apple silicon
  else args.push('--use-angle=swiftshader', '--enable-unsafe-swiftshader');
  if (uncapped) args.push('--disable-gpu-vsync', '--disable-frame-rate-limit');           // measure true frame cost
  return { headless: true, args };
}
```

- `--use-angle=metal --enable-gpu`: real GPU via ANGLE's Metal backend (on Linux/Windows use `--use-angle=vulkan` or `d3d11`). Confirm it took by reading `WEBGL_debug_renderer_info` from the page; it should name your GPU, not SwiftShader.
- `--ignore-gpu-blocklist`: stops Chromium refusing hardware WebGL in headless.
- `--autoplay-policy=no-user-gesture-required`: lets the `AudioContext` start without a click, so audio code paths run in tests.
- `--disable-gpu-vsync --disable-frame-rate-limit`: without these every frame reads 16.7 ms and hides the real cost.

### Freeze Vite's HMR while agents are editing

With several agents saving files, the dev server reloads every open page constantly, which wrecks screenshots and long runs. Stub the HMR client per page:

```js
await page.route('**/@vite/client', (r) => r.fulfill({ contentType: 'application/javascript',
  body: 'export const createHotContext=()=>({accept(){},dispose(){},prune(){},invalidate(){},on(){},off(){},send(){},data:{}});export const updateStyle=()=>{};export const removeStyle=()=>{};export const injectQuery=(u)=>u;' }));
```

(Several builders independently ran a private Vite instance on another port with HMR off; the route stub is simpler.)

### The four scripts

- **`shot.mjs <url> <out.png> [waitMs] [js] [--gpu]`**: one screenshot, optional JS to run first (`__game.trigger('breath')`), prints console errors. The workhorse of every agent's visual self-check.
- **`playtest.mjs`**: ten scripted scenarios (flow, movement, combat, every melee move, every mechanic, phases, death and respawn, victory, pause, HUD), run with `?fixed=1` and `manualStep = true` so only `advance()` moves the simulation. Asserts on game state, saves a screenshot per step, and **fails on any console error**. Uses `boss.passive = true` so the AI does not attack between forced moves.
- **`bot.mjs`**: an in-page bot that plays full fights in real time **through synthetic input only**, reading the game the way a player reads the screen (boss animation, telegraphs, cast bar, hazards). It locks on, rolls on timed windows, walks out of circles, punishes recoveries, drinks when safe, retries after death (which also exercises the reset path). Reports win rate, fight duration, damage taken by source, time the camera spent inside the boss, boss idle streaks. `--passive` measures how long a stationary player survives. `--lab` is a deterministic dodge lab: for every boss attack it sweeps the roll start time on the fixed-step sim and reports the window (in ms) that avoids all damage.
- **`perf.mjs --gpu [--uncapped] [--pr=1]`**: a real-time scripted fight through every move, mechanic, transition and the victory. Wraps `game.frame` to log per-frame interval and CPU time, draw calls, triangles, pixel ratio and **new shader programs** (`renderer.info.programs.length`), annotating the worst hitches with what was happening. That annotation is what found every stall below.

## Balance and feel, measured by the bot

- **Reach**: the one human note was "I have to be too close to hit the boss". Fix without changing the visuals: extend the hit-test blade 0.9 m past the visible tip, pad the boss hurt spheres from 0.12 to 0.45 m, add hurt spheres at sword height under a raised tail and under the belly (a 9 m boss standing over you had nothing hittable in front). Verified by attacking from 2.5 and 3 m in real time.
- **Then re-balance, because reach broke it**: the bot started winning in under 2 minutes, so boss HP went 5,000 to 7,500.
- **Fairness from the dodge lab**: every attack must have a roll window. Tail Lash hurt a whole 3 to 9 m ring for 0.45 s, longer than the roll's 0.36 s of i-frames: window 50 to 117 ms. Hurting only where the tail actually is gave about 300 ms. Final windows 283 to 567 ms for most attacks, 133 to 333 ms for beams.
- **Telegraph honesty**: the breath telegraph showed a 30 degree cone while the flame swept 166 degrees. Show the whole danger area.
- **Get-up frames**: the bot kept being hit by sweeps timed onto the knockdown recovery. Extend i-frames 0.1 s past getting up.
- **Damage and pacing**: boss damage down 15 to 33 %, longer think time in every phase, meteors every 2.4 s instead of 1.8.
- **Result**: bot wins 3 of 6, wins take 3.3 to 4.1 minutes, deaths all in the final phase. A useful target band for "hard but fair".
- **Lethality is a design decision, not a bug**: a passive player dies in 9 to 17 s. Stretching that to 60 s would need 4 to 5x less boss damage and make the active fight trivial. The agent kept genre lethality and surfaced the trade-off to the human rather than deciding silently.

## Traps and lessons (symptom, cause, fix)

**Performance**
- *Hitches of 100 to 930 ms the first time each mechanic fires (15 to 18 compiles per fight).* The warm-up compiled shaders for drawing to the canvas, but the game draws into the post-processing HDR target, which is a different program variant. Fix: bind a render target while calling `renderer.compileAsync(scene, camera)`, then actually render about 30 frames from six camera angles with every effect spawned, and call `gl.finish()`, all behind the title's loading mark. Result: 0 compiles after the first 5 s.
- *Materials recompile whenever an explosion adds a light.* three.js bakes the light count into shaders. Keep a fixed pool of point lights (3 for fx) permanently in the scene at intensity 0 and reuse them. Same for the flask light.
- *Pixel ratio 1.5 misses 60 fps (p95 27 ms).* Cost was 9 point lights evaluated per pixel, not geometry. Options are fewer lights or a lower pixel ratio; the build chose auto-resolution.
- *Auto-resolution only ever stepped down.* Probe the next level up after two good windows, back off 45, 90 then 180 s after a failed probe. Each switch costs a 58 to 72 ms reallocation, so do not flap.
- *A 780 ms freeze when the final-phase channel started.* A drone sound buffer was generated on first use. Build expensive buffers in idle time during load.
- *Texture generation took 12 s.* `getImageData` readback on a GPU-backed canvas. `getContext('2d', { willReadFrequently: true })` keeps it on the CPU: whole texture build dropped to about 170 ms.
- *Black screen for 2 s on load.* Yield one frame (`await new Promise(r => requestAnimationFrame(() => setTimeout(r, 0)))`) before the heavy synchronous build so the title can paint.

**Integration with parallel agents**
- *Pages reload mid-screenshot.* Other agents' saves trigger HMR. Stub `@vite/client` (above).
- *Game fails to boot while a model file is mid-edit.* Static imports die on one broken module. Load models with dynamic `import()` and a placeholder fallback (a capsule with the same API and the contract clip table), and wrap every module call so a throw logs one `console.error` instead of crashing. The playtest counts those errors as failures, so placeholders can never ship silently.
- *Machine load average 30 to 45 with six agents plus headless browsers.* A full playtest took 30 minutes instead of 3, and perf numbers measured under that load were meaningless. Re-measure performance on an idle machine before believing any number.
- *Flaky playtests on a real GPU.* The real-time loop kept moving the simulation between scripted steps. Add a `manualStep` flag so only `advance()` moves it.

**Rendering and modelling bugs**
- *Mirrored right-side armour pieces came out wrong.* The skin builder mutated its input geometry, so clones for the mirror were already transformed. Clone before transforming.
- *Fine hatching inside the light shafts.* The shaft's noise seed was `Math.random()` per vertex, not per shaft, so it was interpolated across triangles. One-line fix; found only by rendering the layer in isolation.
- *Wood-grain rings on large curved surfaces.* Shadow acne from a grazing sun. Raise shadow bias and normal bias.
- *Everything in shade far too dark.* Physical lighting in three.js divides by pi; the hemisphere fill needed about double.
- *Boss arms fling up when the torso rears back.* Forward kinematics carries the arms with the torso; compensate per clip or drive hands with IK targets.
- *Tail swept at 3 m height, over the player's head.* Check hit geometry against the player's height, not just the silhouette.
- *Frozen-frame screenshots mislead.* A hit flash looked washed out at its frozen peak but fades in 0.14 s in play. Judge transient effects in motion or at several times.

**Game logic**
- *Stamina never regenerated after rolling out of a sprint.* A `sprinting` flag stayed true. Found by having the agent re-read its controller end to end for logic slips.
- *Retry left the old phase's sky, stray effects and camera shake.* Reset world phase instantly, `fx.clear()`, zero shake; test three retries in a row.
- *HUD bars wrong after a balance change.* The HUD hardcoded max HP. Read the real values.
- *Camera clipping inside a 9 m boss for 1 to 2.3 s per fight.* Measured by the bot; push the camera out of the boss's body. Now 0.1 to 0.4 s.
- *Headless cannot test pointer lock or gamepads.* Say so explicitly; mouse sensitivity needs a human.

## Shipping

- **Static deploy**: `npx vite build`, then deploy only `dist/` with `vercel deploy dist --prod` (no git integration needed). Check the project name is free (a generic name may already be taken, giving you a random suffix; add a readable alias). New projects can have deployment protection on by default, which puts a login wall in front of a public game: turn it off, then confirm a plain `curl` returns 200 and a real-GPU headless screenshot of the live URL shows the title with no console errors.
- **Add Open Graph tags before sharing.** The first public build had none, so link previews on social platforms were bare. In `index.html`:

```html
<meta property="og:title" content="Game Title" />
<meta property="og:description" content="A boss fight in your browser. Every model, texture and note is generated in code." />
<meta property="og:image" content="https://your-game.example/og.png" />
<meta property="og:type" content="website" />
<meta name="twitter:card" content="summary_large_image" />
```

  Make `og.png` (1200x630) with `shot.mjs --gpu --size=1200x630` against a good moment (`?skip=1&phase=3` plus a `trigger()`). That is a screenshot of the game's own rendering, so it keeps the zero-authored-assets rule.
- Note: the only external fetch is a Google Font for the HUD. If you want literally zero external assets, use a system serif stack.

### Launch film (35 s, 60 fps, cut to the game's own music)

- **Capture deterministically, not by screen recording.** Screen capture of a 3D game drops frames. Run with `?fixed=1`, turn off the real-time loop, and per frame: `advance(1/60)`, step the HUD's CSS animations by the same amount, render at 2x pixel ratio, screenshot, pipe to the encoder. Seed `Math.random` with an init script so every take reproduces. Drive the knight with the bot or scripted input under `?god=1`. Cost: 1 to 2 minutes of capture per second of footage, so plan shots and take one take each.
- Adjust the camera at runtime for the film (the lock-on distance was pulled from 4.5 to 6 to 7.5 m so the boss fits in frame) instead of editing game source.
- **Audio from the game itself**: render music and SFX to WAV through the game's own offline audio path. The score's tempo is known from code (134 bpm, beat = 0.4478 s), so every cut sits on a whole or half beat without beat detection.
- **One timing grid**: snap every cue and clip length to the 60 fps frame grid, and land visual hits just before their frame, not after. An unsnapped cue put a flash 21 ms (1.3 frames) behind its explosion. Bake clip timing into each frame's composition rather than timing it at load, or clips play from the wrong point in the assembled film.
- **Disk space**: 1080p60 PNG sequences are huge; encode straight from the capture pipe and keep a free-space check in the render script. Low disk killed two full renders.
- Master to about -14 LUFS integrated, true peak below -1 dBTP; verify picture-to-audio sync on the delivered file (all hits within one frame).

## Prompting pattern that worked

1. One loose creative prompt with full creative liberty.
2. The coordinator writes the contract doc and stubs, checks headless WebGL renders, commits the scaffold.
3. Six builder agents in parallel, each owning its files, each with a dev viewer and headless screenshot self-checks, each reporting known weaknesses honestly.
4. One integration agent with write access to everything: real-GPU perf harness, bot, dodge lab, balance, loading, README. Human feedback during this phase goes to that agent, so two agents never edit the same code.
5. The human plays it and gives feel notes; the bot re-measures after every change.
