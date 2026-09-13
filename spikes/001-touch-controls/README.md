# Spike 001: one-thumb touch controls

**Status: VALIDATED** (one of three variants rejected, with evidence)

## Question

Given a landscape phone, a quantized Scorched Earth simulation, and all controls confined
to one side — **which control scheme is accurate enough to preserve the precision the genre
runs on, and how does each feel one-handed?**

## Why it mattered

This was the only genuinely unproven claim. Everything else (quantized physics, a bracketing
AI, an out-of-match economy) is engineering with a known answer. If no one-handed scheme
could land an exact angle and an exact power repeatably, the mobile premise was wrong and
the honest move would have been desktop with a mouse and keypad.

## What was built

Single file, no dependencies, no build step, no backend. Canvas for the field, DOM for the
HUD so the touch targets are real. Two field shapes, three control schemes, live toggle for
each. World coordinates fixed; physics on integer inputs with a fixed timestep.

## Verdict

### What worked

- **Split pad (A).** A 36px × 64px one-thumb drag moved angle +30° and power +40 — exactly
  the designed 1.2px/° and 1.6px/unit. A 1px nudge moved angle exactly 1°. Values are
  quantized on release, not during the drag, so **what you read is what fires.**
- **Sliders (B).** Absolute mapping exact: 25% along the angle track → 46°; the top
  quartile of the power track → 750.
- **Determinism — the core claim.** Repeated shots with identical integer inputs produced
  byte-identical trajectory hashes (`146:293913`, `193:325717` in the two field modes).
  Same inputs ⇒ same shot, always. Precision is enforced by the simulation, not demanded
  of the finger.
- **Bracketing AI.** First shot errs 110–155px; within 2–3 turns it is inside ~5px. Then
  ±18 power of jitter keeps it near-missing rather than perfect. It reads as competence.
- **Zero occlusion, both modes** (measured, not assumed): 4:3 field at x 250–770 vs HUD
  column at x 0–176; wide field at y 0–303 vs HUD band at y 304–390.
- **Hand mirroring** works in both modes, and in column mode it relocates the field with it.

### What didn't

- **Variant C (direct aim) — rejected.** Aiming at the vertical centre of the screen
  produces 2°: the tank sits low in the field, so anything at or below its height collapses
  to ~0–2°, and the whole 0–90° range is compressed into the few hundred pixels between the
  tank and the top of the screen. Worse, it is **unrepeatable** — you cannot place a thumb on
  the identical pixel twice, so an exact angle cannot be reproduced. The precision objection
  to gestural aiming, now with evidence instead of assertion.
- **Unclamped off-field landings.** A shot that leaves the field reports where it crosses
  the bottom edge, not where it would have hit ground, so its miss-distance in the log is
  measured against a different reference height than an in-field landing. The sign is
  right so bracketing still converges, but the number is not comparable. Needs a fixed datum.
- **16:9 + a screen-height HUD do not fit in 844×390.** The first layout put the HUD panel
  on top of the player's own tank; moving the tank simply occluded the enemy instead. This
  was not fixable by nudging — it forced the layout decision below.

### Surprises

- **Wind drift is a constant fraction of field width, not a bigger deal on a wide field.**
  Because VMAX has to scale with √W for the power dial to stay calibrated, flight time scales
  with √W, and drift ∝ t² ∝ W — so the two cancel. Measured spread from wind −100 to +100 on
  a high lob: 45% of field width at 4:3, 49% at wide. What the wide field actually changes is
  the *bracket*: 63.6 tank-widths across versus 36.4. Smaller targets, more correction.
- **Destroyed ground invalidates a bracketed solution.** After the computer brackets to
  within 5px, its next identical shot lands 119–126px long — because the craters it dug near
  the target changed the impact geometry. Terrain destruction is a live counter to a solved
  shot. Emergent, not designed: the strongest thing in the build.
- **The computer ignores the wind entirely.** Its error correction absorbs it. Cheaper than
  modelling wind, and it *reads* as expertise.
- **The original's proportions reproduce almost exactly.** 400px field ÷ 11px tank = 36.4
  tank-widths across, against the DOS original's ~35 (320 ÷ ~9). Same sense of scale as 1991.

### Recommendation for the real build

- **Field: 4:3 as the default.** Not for nostalgia — because a 4:3 field on a 19.5:9 phone
  leaves a 176px side pillar, and that pillar is exactly what a full-height one-thumb control
  column wants. Keep the wide field available; the toggle costs nothing and it is a real
  alternative (more bracket, more field, bottom HUD band).
- **Controls: split pad.** Everything in one column, mirrored for handedness.
- **Keep quantized physics.** Integer angle/power/wind, fixed timestep, seeded RNG. This is
  the entire reason the game feels exact; it is not an implementation detail.
- **Unresolved: haptics.** `navigator.vibrate` exists in Chromium (so: Android) and is
  absent in iOS Safari. If the 10°/50 detent feel is load-bearing, that is a platform
  decision, not a UI polish item. The HUD reports which you have.
- **Unresolved: interruption survival.** The spec promises a match you cannot lose to a
  phone lock. Terrain is *not* reconstructible from the seed once craters exist — you must
  persist the dirt mask or replay the shot log. Deferred, deliberately, and not pretended
  away.

## Persistence — solved, and three ways to get it wrong

The spec promised a match you cannot lose to a phone lock or a discarded background tab.
That is now built and verified, and it is cheap because the simulation is already deterministic.

### Mechanism

A match is reconstructible from **the terrain seed, the ordered list of literal carve
operations, and the tank/AI/log state.** The 120,000-byte dirt mask is never stored.

| Approach | Size | What breaks it |
|---|---|---|
| Store the mask | ~20KB packed | nothing — literal, but large and inert |
| Replay the shot log | ~1KB | **any retune of the ballistics silently corrupts every save** |
| Seed + literal carve ops | **~860B** | nothing — carves are coordinates, not physics |

Measured: **859 bytes** for a two-crater board including a 128-point tracer path; **366 bytes**
for a board with a shot still in the air.

Replay would have been the elegant choice and it is the wrong one. A crater recorded as
`(351,183,24)` means the same thing forever. A crater recorded as "shot 4 at angle 52, power
690" means something different the moment VMAX changes.

### Three ways to get it wrong — I hit all three

1. **Recording rounded carve coordinates while rasterizing the unrounded ones.** Every crater
   lands up to half a pixel off on replay, so the terrain drifts a few pixels per explosion.
   Measured divergence: mask hash `1134081210` vs `66906296`, 27 pixels of dirt. Silent and
   cumulative. Fix: quantize the crater centre *before* both recording and rasterizing.
2. **Not restoring `state.seed`.** The terrain is rebuilt from the right seed, so it *looks*
   perfect — but the next save writes the wrong seed, and every reload after that is quietly
   wrong. A delayed fuse. This is the one that would have shipped.
3. **A stale `setTimeout(aiTurn)` queued before backgrounding.** On resume it fires a spurious
   computer turn. Fix: gate `aiTurn` on `phase === 'wait'` rather than merely "not over".

### Verified against real page reloads

- **phase `aim`** — terrain bit-identical (mask hash `66906296`, dirt `47593`), seed, AI bracket
  memory and carve log all identical. Zero differences across every field compared.
- **phase `flying`** — a shot saved at `t=0` in mid-air resumed to the same outcome as the same
  shot stepped synchronously: same crater `[29,185,38]`, same landing, same mask hash, same HP,
  and **exactly one log entry** — no duplication.
- **phase `wait`** — a queued computer turn survived the reload and fired exactly once.

### Accepted tradeoff

The tracer path is decimated 3:1 on save (128 points → 43). Cosmetically identical; a resumed
match's tracer line is very slightly coarser.

### Not covered

- Device rotation while backgrounded (landscape is assumed throughout).
- A save written by an **older build after a tuning change**, when the interruption happened
  mid-flight. Carves are immune to retuning; a projectile in the air is not. The schema is
  version-gated (`v:1`) but the version does not cover the physics constants.

## The invariant — closing the class, not the instances

The two bugs above, plus the tracer erosion, are three instances of one failure:
**`restore(snapshot())` was not an identity function.** So the fix is not a third missing line.
The class has four parts:

1. **Canonicalize at the choke point.** A value that gets recorded must be byte-identical to the
   value that gets used. Quantize craters at the single point where they enter the terrain.
2. **Every field the loader is handed, it must write.** No partial assignment, ever.
3. **No lossy transform in the save path.** `Math.round` is idempotent and safe; 3:1 decimation
   is neither.
4. **Check it, do not assume it** — and fail safe when it trips.

```js
const SETTLED = ['v','field','side','variant','seed','wind','round','wIdx',
                 'tanks','ai','carves','log','ghost'];
function selfCheck(sv){
  const now = snapshot();
  return SETTLED.filter(k => JSON.stringify(now[k]) !== JSON.stringify(sv[k]));
}
```

It runs at the end of `restore()`, *before* an in-flight shot is allowed to change anything. On a
non-empty result the game stops autosaving and says so in the HUD: a rejected save is
recoverable, a silently corrupt board is not, and continuing to save would write the corruption
forward.

### Verified

| check | result |
|---|---|
| 5 save/restore cycles | tracer `126 → 126 → 126 → 126 → 126 → 126` (was `113 → 38 → 13 → 5 → 2 → 1`); terrain stable, seed stable, check clean every cycle |
| 12 deliberate violations | all 12 detected — `log`, `carves`, `ghost`, `seed`, `wind`, `ai` (two ways), `wIdx`, `round`, `side`, `field`, `variant` |
| a deliberate no-op | correctly silent |
| a genuine restore | clean |
| an in-flight resume | passes before the catch-up; no false rejection |

### The checker found a hole in the code it was checking

`field` tripped as a **false positive**. `restore()` never wrote `state.field` — the field shape
was applied by `setField()` at boot, *before* restore ran. The identity held only by accident of
call order, which is exactly the kind of thing that breaks later when boot order changes. Fixed by
making `restore()` self-contained: it applies the shape itself now.

### Cost

The save grew from ~460 bytes to ~1.6KB. The tracer path is stored whole (126 points) instead of
decimated, because the decimation *was* the lossy transform. Against a 5MB localStorage budget
this is not a real cost, and state integrity was the thing worth buying.

### A note on the tests, because three of mine were worthless

Three detector probes came back silent and looked like a broken checker. They were broken
*probes*: `log.slice(0,1)` on a one-entry log; `ghost` on a board where only the computer had
fired, so no tracer existed; `ai.last = {x:null,...}` on a board where the computer had never
shot. **A test that changes nothing proves nothing** — the same trap, moved one level up, that
let bug 2 hide from the first reload test.

## Armory — the economy, verified

Built on top of the deterministic core. Screens, economy and the save-shape gate, all checked live.

| check | result |
|---|---|
| starting state | 1000 CR, nothing but free shells, FIRE disabled while shopping |
| buy / sell | 3 missiles → 820 CR; sell one → 880 CR (full refund) |
| cannot overspend | 0 CR cannot buy a 520 CR nuke |
| stock reaches the match | chips read `MISSILE ×2`; the free chip carries no count |
| a paid round is spent | stock 2 → 1 on fire |
| the free shell is never spent | stock 0 → 0, fires anyway |
| empty gun | refuses, phase unchanged, `OUT OF AMMO — PICK ANOTHER` |
| computer's bankroll | 1000 CR → picks MISSILE (−60); 0 CR → falls back to the free shell |
| armour | absorbs 22 of a 22-point hit; hull untouched |
| settlement | 170 damage + 400 win bonus = 570 CR; `settle()` is idempotent |
| reload | bank, stock, armour and payout all identical, differences NONE, **no double payout** |
| invariant | `selfCheck` clean with the nine new economy fields in `SETTLED` |
| phone layout | cards 202×99, none clipped, grid 311px and not scrolling, header fits |

### The save-shape gate, because I walked into the same hole twice

Two schema changes today both left old saves loading as `undefined` — and `undefined === undefined`
is exactly what the invariant check waves through. So the version is now one constant
(`SAVE_VERSION`) and `readSave()` additionally compares the **key set** against `Object.keys(snapshot())`.
A save written by a build with a different shape cannot load, and adding a field can no longer be
forgotten in the loader. Verified: `v1 → REFUSED`, wrong shape `→ REFUSED`, bad field `→ REFUSED`,
correct shape `→ accepted`.

### One self-inflicted wound worth recording

Replacing the old `#overBtn` element without removing its listener threw at boot and aborted
everything after it — no weapon chips, no armory, no save. Silent in the console unless you look.
Deleting a DOM node is half a change; the other half is the code that reaches for it.

## Rendering — the dpr bug, and why a dozen screenshots missed it

The barrel ghosted when sweeping the angle. Cause: `resize()` sizes the backing store in **device**
pixels (`canvas.width = cw*dpr`) while `render()` cleared it in **CSS** pixels
(`ctx.setTransform(1,0,0,1,0,0)` then `fillRect(0,0,cw,fieldH)`). At any `dpr > 1` the sky fill
covered only the top-left `1/dpr × 1/dpr` of the canvas, so **63% of the canvas was never
repainted**. The terrain kept looking correct because it is drawn opaquely every frame; the sky
became a surface that accumulated every past frame, and a bright white barrel sweeping across it
left the most obvious trail.

The same mistake broke the gradient the same way: `createLinearGradient(0,0,0,fieldH)` in identity
space spans only the top `1/dpr` of the canvas, so the sky never had its intended vertical range
on a phone either.

**Fix:** one line — `ctx.setTransform(dpr,0,0,dpr,0,0)`. The sky/HUD layer and the world layer
(`dpr*fit`) now live in the same coordinate space.

| at dpr = 3 | before | after |
|---|---|---|
| a mark painted in the far corner survives a render | yes | no |
| canvas left unpainted after a frame | **63%** | **0.00%** |
| fully transparent pixels · iPhone 844×390 | — | **0** |
| fully transparent pixels · Pixel 915×412 | — | **0** |
| sky gradient spans the field | no | `#060a12` top → `100,69,41` low |

### Why every screenshot taken today missed it

This browser renders at `dpr = 1`, and at `dpr = 1` an identity-space fill of `(0,0,cw,fieldH)`
happens to cover the whole canvas. The bug is **invisible at exactly the value my instrument was
pinned to** — so a dozen screenshots, including ones I looked at carefully, could not have shown
it. The harness now forces `devicePixelRatio = 3` on every frame before it renders. A harness that
claims to show a phone has to lie about the pixel ratio the same way a phone does.

This is the same failure as the three no-op detector probes: **measuring something adjacent to the
thing that matters, and reading the result as reassurance.**

## Falling — tanks obey gravity; dirt does not

The 1991 manual settles this, and it inverts the intuition (both quotes are in SPEC.md):

- **Suspend Dirt, default `0`** — the number is the chance dirt *falls*, so at the default it
  never does. Hanging terrain is the original's normal state.
- **Tanks Fall, default `ON`** — the mechanic we were missing. Tanks drop when the ground is shot
  out from beneath them and take damage proportional to the fall.

So the fix was never "add dirt physics". It was "add falling tanks", and leave the dirt exactly
where the explosion put it.

| check | result |
|---|---|
| ground removed under a tank | falls 27px, takes 6hp — exactly `(27-10)/3` |
| dirt does not fall | a void 16px under the surface keeps its roof, still there 20 frames later |
| dug clean through | 3 carves to the floor, `surf` = `H`, tank destroyed |
| determinism | same seed → same mask hash, same player y, same enemy y |
| fall is a pure function of terrain | `savedY 205 === rederivedY 205` after save → restore |
| damage applied once | hp identical across the round trip, invariant clean (`[]`) |
| rendered | tank stands on a 13px dirt bridge over a void; thin roof visibly hanging over a tunnel |

### Why the invariant now covers tank position

`tanks` gained `x` and `y`, which are **derived, never restored**. So the invariant check asserts
that replaying the terrain puts the tank precisely where the live board had it — the same
live-versus-replayed check that had nothing to catch before, because nothing had ever moved a tank.

### What this unlocks

DIGGER and ROLLER had no teeth: removing ground beneath a tank did nothing, so the whole
earth-moving weapon class was damage-with-extra-steps. Now the ground is a weapon. And the
manual's counter — **Parachutes** (deploy when the predicted fall damage exceeds a safety
threshold) — is suddenly a real thing to sell in the armory, which is a defence row the original
had and we didn't know why.

## Parachutes — the counter, and it costs something

| scenario | fall | predicted | result |
|---|---|---|---|
| no chutes owned | 31px | 7hp | takes 7 |
| armed, above threshold | 31px | 7hp | **takes 0**, chute spent (3→2), canopy up, descends at 3px/frame not 7 |
| armed, at/under threshold (21px → 4hp ≤ 5) | 21px | 4hp | takes 4, **chute kept** — not worth opening |
| passive | 31px | 7hp | takes 7, chute kept |
| bottomless, in one carve | — | — | hp 0, **chutes unchanged at 3** |
| bottomless, dug in three steps | 3 falls | — | 2 intermediate falls spent 2 chutes, the last did not |
| threshold dial | — | — | 5 → 10, persisted, invariant clean |
| computer | — | — | buys one at 1000 CR (→860, then a 60 CR missile) |
| round trip | — | — | chutes/armed/threshold all restored, invariant `[]` |

The three-step bottomless case is the interesting one: it *looks* like a chute leak (3 → 1) until
you count the falls. Digging through a tank makes it collapse twice before it goes, and each of
those is a real fall that legitimately opens a chute. The original's rule is per-fall, so this is
correct — and it means digging a defended tank out is attrition, not a free kill.

### Fixed in passing

The shot-log box printed its hint twice: the heading read "SHOT LOG · bracket on it" and the
empty-state placeholder *also* read "shot log: bracket on it". Pre-existing, cosmetic, and only
visible when the log is empty — which is exactly when you are looking for guidance. Heading now
names the box; the body carries the hint.

## Terrain generation — 100-seed sweep

A generator can't be verified by looking at one map. Swept 100 seeds:

| check | result |
|---|---|
| seeds with roofed-over air (overhangs/caves) | 96/100 |
| seeds with a vertical wall (step ≥ 10px) | 80/100 |
| seeds with **neither** a wall nor an overhang | **0** |
| entombed tanks / spawns without ground / spawn footprint not solid | **0 violations** |
| regeneration with the same seed | byte-identical mask, same spawn y |
| surface relief (max−min of `surf`) | min 30, mean 72, max 125 of a 300px field |
| sky/rock probes via `solid()` | air under a roof reports non-solid; rock reports solid |

### And the DIGGER, which was lying

The `tunnels` flag carved the last 22 points of the *approach path* — a projectile stops at first
contact, so those points are air it already flew through. Traced: carving the first three approach
points left the solid count unchanged at 45738. It only ever bit the last few, at the impact.

| | old (carve the approach) | new (bore along the heading) |
|---|---|---|
| rock cells removed | 293 | **773** |
| covered tunnel cells created | 3 | **10** |
| depth into the rock | at the impact only | 29px |
| follow-up identical shot | 374 → 374 (no gain) | 366 → **628**, off the field |

Also fixed: it claimed `dmg 18` and damaged tanks on contact. The manual — *"Most of these weapons
cannot directly harm a tank, though they can cause them to fall and take damage that way"* — so
direct damage is now 0, and *"if they hit a tank, they fizzle"* is implemented and verified (3px
from the enemy, zero damage).

### One measurement error worth recording

I first reported these as `rock_removed: -293` and `-773`. Negative because I computed
`after − before` and removal *is* negative — the arithmetic was right and my label was wrong. I
stopped and instrumented rather than publish the comparison, which is how the trace above (and the
proof that the approach carves hit air) came out. Worth the detour: the sign error was cosmetic,
but the trace was the actual evidence.

## The three outstanding defects — fixed

Found by looking at them rather than trusting the list. All three were real, all three reproduced,
all three fixed.

| | reproduced | now |
|---|---|---|
| Off-field miss printed as a distance | `land 548, err +171` — the exit point, not a landing | flagged `off`, renders `OFF · LONG`; in-field still shows `-2px` |
| Mid-air resume × retune | save held `t=20` and only angle/power; a 5% `VMAX` change moved the landing **40px** | save carries `x,y,vx,vy`, resume continues from there: **584 → 584, no divergence**; retunes announced |
| Rotate while hidden | rect frozen at `fit 1.923, cw 1280` before, during *and* after | `fit 0.535, cw 390, ch 844` on return, canvas rematched; `0×0` keeps the last good layout |

### Correction: my "broken instrument" finding was wrong

I reported that the verification console could not see the page's top-level lexical bindings —
`W`, `VMAX`, `WEAPONS`, `chuteOn` unreadable while `window.__spike` resolved — and wrote it up as a
blind spot that explained how three defects had survived. **It wasn't true.** Re-probed on a page
that was actually loaded:

| name | bare | as `window.` property |
|---|---|---|
| `W`, `H`, `VMAX`, `GRAV`, `cw`, `fit` | `number` | `undefined` |
| `WEAPONS`, `chuteOn` | `object` | `undefined` |
| `carve`, `selfCheck` (function declarations) | `function` | `function` |

Bare lexical identifiers resolve perfectly. It is the `window.`-qualified route that fails for
`let`/`const`, which is ordinary JavaScript: top-level `let`/`const` are not window properties.

The original probe returned `undefined` for *everything* — including `carve` and `selfCheck`, which
are function declarations and therefore *are* window properties. That is impossible on a loaded
page. The page was `about:blank` (the browser had dropped it), so every identifier read `undefined`
and the realm was simply empty. I had a tell — the impossibility — and read straight past it.

The lesson is the opposite of the one I first wrote down: **when a probe returns null for
everything, suspect the premise before concluding something exotic about the world.** A blank page
and an exotic scope rule produce identical readings, and only one of them is plausible.

Extending `__spike` was still worth doing, on grounds that stand alone: `window.*` is reachable
regardless of which world the eval lands in, and the accessors let a test *write* lexical state
(`set VMAX`), which a bare identifier cannot do to a `const`. The justification in the code comment
was wrong and is corrected.

Unlike the `dpr=1` screenshots and the three no-op detector probes — which were genuine instrument
blind spots — this one was a false alarm *about* the instrument. Two real, one imagined.

## Quality of life: the empty gun, and the bug under it

Asked for: when a weapon is empty, show the alert and then auto-swap to the free shell.

**The swap had a trap in it.** `wIdx` is in `SETTLED`; `msg` is not. A *deferred* swap would mutate
`wIdx` from a timer, so if a save landed in between — and `pagehide` fires exactly then — the next
autosave's `selfCheck` would report the board unsettled and shut autosaving off. So the swap is
immediate and only the message lingers. Same thing the player sees; no pending mutation.

**And testing it found a worse bug underneath.** The computer's turn did this:

```js
state.wIdx = aiPickWeapon();     // state.wIdx is the PLAYER's selection
```

Nothing restored it. The computer kept no weapon of its own — `const ai = { last:{...} }` had no
weapon slot — so it wrote its choice into the player's. Reproduced:

| | before the fix |
|---|---|
| player picks | `wIdx 3`, chip highlights **NUKE ×2** |
| computer's turn | `state.wIdx → 1`, fires MISSILE |
| player's selection survived | **false** |
| chip still highlighted | **NUKE ×2** |
| next player shot would fire | **MISSILE** |

You plan a nuke, the computer takes the choice away, the HUD lies about it, and the game fires
something else. Fixed by giving the computer `ai.wIdx` and having `fire(who)` read the weapon from
whoever is shooting.

| | after the fix |
|---|---|
| computer fires | MISSILE (its own pick) |
| player's selection | survives: `state_wIdx 3`, chip "NUKE ×2" |
| player's shot | fires NUKE, `used_own_selection: true` |
| empty press | `OUT OF BABY NUKE · SWITCHED TO BABY MSL`, chip moves to BABY MSL |
| invariant | `[]` clean; v8 save; a v7 save refused |

Worth noting how it was caught: not by looking for it, but because verifying the *requested* feature
meant asserting that the chip highlight followed the selection — and it didn't. The stale highlight
was my own new code (the auto-swap called `syncHud()` but not `buildWeapons()`), and while chasing
that I read the line above it.

## The "speckled fringe" on the terrain — investigated, not a bug

Reported by the agent as a possible rendering fault: a dark speckled line tracing the terrain
surface. **Verdict: antialiasing, working correctly, but revealing a real choice.**

`redrawTerrain()` paints a per-pixel hash dither (±7) plus 9-row strata, then a 1px `#c99155`
highlight line at `surf[x]` for every column. The terrain is 400×300 logical and drawn at 769px
wide — a **1.923× upscale** — and `imageSmoothingEnabled` is never touched anywhere in the file, so
it defaults to on. Bilinear filtering therefore blends that highlight line into the sky on whichever
destination pixels happen to straddle the source boundary.

Confirmed by arithmetic and by pixels. At x=210 the row above the surface reads `59,54,50`, and
`0.25×(201,145,85) + 0.75×(16,28,41) = (62,57,52)`. That is a 25% blend, not a dark halo.

| measured over 320 columns | smoothing ON | smoothing OFF |
|---|---|---|
| columns with a fringe pixel above the surface | **20.3%** | **0%** |
| dirt texture σ (dither amplitude surviving) | 5.912 | **7.446** |
| surface line at x=210 | `143,108,72` (diluted) | `201,145,85` = exactly `#c99155` |

So the crisp blit is both cleaner and closer to what the code actually asks for. The one-line
option is `ctx.imageSmoothingEnabled = false` before the terrain blit. **Applied** — crisp is more
in the spirit of the original, and the numbers agree it is what the code was asking for:

| verified after the change | faithful 4:3 | wide | faithful at dpr=3 |
|---|---|---|---|
| fringe columns | **0%** | **0%** | **0%** |
| columns carrying the exact authored `#c99155` | **100%** (336/336) | **100%** (588/588) | **100%** (336/336) |

dpr=3 matters most: that is a phone, where the upscale is three times larger and the blur (and
therefore the fringe) would have been three times worse.

`ctx.drawImage` appears **once** in the entire file — this blit — so nothing else is affected;
`imageSmoothingEnabled` has no effect on vector fills or strokes.

What crispness exposes, now visible and worth a decision later rather than now: the 9-row strata
(±13 on a base of ~100) read as strong horizontal bedding across the whole mass, and the tower
feature is a plain rectangle by construction.

### A false finding about the instrument, corrected

While investigating, the agent reported that the verification console could not see the page's
top-level lexical bindings, and wrote it up as the third instrument blind spot of the project. It
was **wrong** — see "Correction" above. Bare `let`/`const` identifiers read fine. The page had
dropped to `about:blank`, where every identifier reads `undefined`, so a probe of *everything*
returned null and an exotic scope rule was inferred from an empty realm.

The transferable rule: **when a probe returns null for everything, suspect the premise before
concluding something exotic.** There was a tell — `carve` and `selfCheck` are function declarations
and therefore window properties, so their reading `undefined` was impossible on a loaded page — and
it went past unexamined. Asserting `location.href` at the top of a test costs one line.

## Fullscreen — the control the game was missing

The `⛶` button existed but was unusable in three ways, all found by reading it: a bare glyph nobody
would find, **enter-only** (pressed while already fullscreen it skipped the request, reported
`fullscreen ✓`, and did nothing — with no way back out but the browser's own escape), and no
`fullscreenchange` handling.

Now: a labelled `⛶ FULL` in the top bar, a `⛶ FULLSCREEN` beside START MATCH in the armory, a real
toggle, and honest platform reporting.

| verified (API stubbed — a scripted browser cannot enter real fullscreen) | |
|---|---|
| enter / exit | `entered: 1, exited: 1` — the old one could never exit |
| landscape orientation lock | `locked: 1`, requested after entering, as required |
| labels | `⛶ FULL / ⛶ FULLSCREEN` → `⛶ EXIT / ⛶ EXIT FULLSCREEN` → back |
| messages | `FULLSCREEN ON` → `BACK TO THE BROWSER` |
| `fullscreenchange` re-measures | `fit 1.923 → 1.3`, `cw 1280 → 844`, `ch 577 → 390` |

**Not verifiable here:** real fullscreen entry (needs a real user gesture and a real browser), and
the iOS branch. The stubs cover the logic; the phone covers the rest.

Also checked, because a second button in a tight footer is exactly how layouts break: at a **real
844×390 viewport** the armory footer does not overflow (`scrollWidth 828 == clientWidth 828`, bottom
382 of 390), all 8 cards fit, and both buttons are inside the viewport (`⛶ FULLSCREEN` 136×33,
`START MATCH` 124×31).

### Testing correction: overriding `window.innerWidth` does not resize the layout viewport

The first attempt at that layout check set `window.innerWidth = 844` and measured with
`getBoundingClientRect()` — and concluded the footer overflowed and sat below the viewport. Both
were **artifacts of the test**. Overriding `innerWidth`/`innerHeight` changes a JS property; it does
not move the browser's real layout viewport, so DOM geometry is still measured against the true
1280×600 while being compared against a fake 390.

This is fine for testing the *canvas* — `cw`, `ch`, `fit`, `ox`, `oy` are JS values computed from
`window.innerWidth`, so they do follow the override — and invalid for testing *DOM* layout. For DOM
geometry, use a **real layout viewport**: an iframe sized to the target. Same origin, so its
`contentDocument` is reachable and its `position:fixed` elements resolve against the iframe.

The distinction matters because the canvas path was verified this way repeatedly and correctly, so
the method looked proven — and then it was pointed at the DOM, where it silently measures the wrong
thing. It is the same trap as the others in this file: an instrument that works on one target and
lies on another.

## The armory clipped its own top — windowed only

Reported as: *"when not in full screen, the top of the armory gets clipped."* Exact, and the cause
was one property value.

`#armGrid{flex:1; overflow:auto; align-content:center}`. Plain `center` pushes overflow in **both**
directions, so when the rows are taller than the grid box the overflow goes *above* the scroll
origin — and no scroll offset can reach a negative region. The top row is clipped *and* unreachable.

Why fullscreen was fine: it is tall enough for the rows to fit, and centring then does what it
looks like it does. Windowed phone browsers are short — toolbars cost roughly 100px of a 390px
landscape height — which is the difference between fitting and not.

Live A/B in a real layout viewport, reproducing the old value with an inline `align-content:center`:

| viewport | shipped (`safe center`) | old (`center`) |
|---|---|---|
| 844×390 fullscreen | fine, first card +31px | fine, +31px |
| 844×346 windowed | fine, +9px | fine, +9px |
| 844×290 windowed, short | **+0px · clipped false · reachable true** | **−19px · clipped TRUE · reachable FALSE** |

Fix: `align-content:center; align-content:safe center`, `overflow-y:auto` on the overlay as a net,
and a `@media (max-height:430px)` block tightening gaps and padding so it fits more often before
scrolling is needed. The plain value stays first as the fallback for engines that don't know `safe`.

Verified at a real 840×286: header fully visible, first row fully visible *including the card names*,
footer fully visible, grid scrolling for the remainder.

**The first screenshot of this was misleading, and it was mine.** The probe had left
`grid.scrollTop` at the bottom (it was testing the last row) and I photographed the result, then
briefly read it as still-broken. A probe that leaves state dirty produces a photograph of the wrong
thing — the same family as the three detector probes that changed nothing and "passed".

## Not a finding

Frame rate, memory, and battery were not measured. Not relevant to the question asked.


---

## Slice 1 — the opponent ladder and information weapons (2026-09-12)

The difficulty setting is the *opponent*, because the 1991 manual contains no difficulty setting at
all — it contains a roster, ordered by what each opponent knows how to do rather than by how badly
it aims. So the implementation is a set of competences (`randomShot`, `losClear`, `bracketShot`,
`solvedShot`, `chooseMethod`) and each opponent is a set of them. Adding a rung later is a table
entry, not a code path.

**The ladder, measured.** 40 seeds each, first shot, no wind, in `ladder.html`:

| opponent | first-shot hits | median | mean | worst | character |
|---|---|---|---|---|---|
| MORON | 0/40 | 235 | 216 | 357 | no feedback at all |
| SHOOTER | 6/40 | 234 | 191 | 357 | solves *only* with a clear lane — 34/40 maps blocked it |
| TOSSER | 0/40 | 134 | 139 | 287 | brackets; its first shot is deliberately crude (matches SPEC's 110–155) |
| SPOILER | 37/40 | 2 | 18 | 288 | solves wind and gravity; its 3 misses are terrain in the way |
| CHOOSER | 40/40 | 2 | 2 | 4 | solves *and* checks the lane before committing |

That ordering is the manual's, arrived at by measurement rather than by design intent — including
the detail that SPOILER's failures are exactly the case the manual excuses it from ("assuming
nothing is in the way").

**The discriminator.** Error against wind 0/30/60/90:

```
TOSSER   122 → 132 → 145 → 157     degrades with wind
SPOILER   29 →  27 →   2 →  14     independent of wind
```

This is the test that matters. An opponent that merely had a *smaller error constant* would fail it;
a Spoiler passes it because it solves the same equations the shell flies on. Solving is done by
**simulating** — `simShot` runs the engine's own integration — so there is no separate ballistic
model that can drift away from the one that decides where the shell lands.

**Determinism.** All five opponents fire identical shots when the same seed and turn are run twice.
This closed a real hole: the old jitter came from `Math.random()`, so an AI turn interrupted by a
phone lock fired a *different* shot on resume. It was never caught because trajectory hashes had only
ever been compared for player shots. The AI now draws from a stream derived from
(seed, round, turn) — nothing to save, and reproducible on re-entry.

**Two bugs found in this work, by test rather than by reading:**

1. SPOILER and CHOOSER passed their elevation list through without the direction conversion, so a
   computer facing west aimed east and shot off the far edge — mean error 631px, *worse than a
   Moron*. Worse, `simShot` decided which way the target was from the shot's own velocity, so a shot
   flying away "crossed" the target's x on tick 1 and scored itself near-perfect. Both fixed; the
   ladder above is the post-fix measurement.
2. The buried case detonated on tick 1 as designed but carved nothing — which quietly deleted the
   third way out of a burial ("keep firing and dig yourself out badly"), the option the whole
   no-damage rule rests on. It now bursts where it stands and carves, and deals no damage.

**Schema v9.** `opponent` joins `SETTLED` — a match resumed against a different AI is a different
match, and `aiTurn()` is re-entered after a reload, so it cannot be re-derived. `smokes` joins it for
the same reason. Verified: `restore(snapshot())` is an identity with both fields present, and an
unrecognised opponent name is *not* silently defaulted — it is written through and rejected by
`selfCheck` (mismatched keys `[]` on the clean path).


---

## Slice 2 — the Earth family and burial (2026-09-12)

The category that was missing, and the reason "more armory content" was never a content exercise.
Earth Producing *builds* terrain, Earth Destroying removes it, and both rest on a BURIED state that
the 2D mask already enforces for free: a shot fired from inside rock bursts where it stands.

**Terrain ops are typed now (schema v10).** `deposit()` is the mirror of `carve()`, so a crater and a
deposit are different operations and the ordered op list carries a type: `c` circle carve, `d`
deposit, `l` liquid, `w` wedge. `restore(snapshot())` is still an identity with all four present.

**The burial cost, measured** (turns to dig yourself out by firing — free shell / missile / nuke):

| dropped on you | free shell | missile | nuke | riot charge |
|---|---|---|---|---|
| Dirt Clod | 3 | 2 | 1 | **1** |
| Dirt Ball | 5 | — | — | **1** |
| Ton of Dirt | 10 | 6 | 3 | **1** |

That spread is the mechanic. Firing is a real way out and a bad one, costing more the more was
dropped on you; the Riot Charge is flat and always exactly one turn. Six measurements per row, in
`ladder.html`.

**What BURIED means**, after three attempts. Buried = *there is a roof overhead*: ≥4px of solid
within the 24px above the hull. Not the hull row (reports CLEAR while every shot still hits the
roof — a readout disagreeing with reality, the worst failure mode this project has produced). Not
the muzzle (flickers as you traverse; a self-dug crater clears exactly the muzzle's own cell and
nothing beyond it).

**Six defects found, all by test, all in this slice's own work:**

1. `endShot()` reads `proj.owner`, and the Riot Charge calls it with no projectile — a hard crash on
   first use of the entire Riot family. Fixed by giving it a shot that has already arrived.
2. `dropTanks()`'s "dirt piled on top → the tank rides up" rule silently *unburied* every tank the
   moment it was buried, cancelling the whole category. Now a buried tank stays put.
3. `paintLiquid()` took the deepest column as its fill level, so it filled nothing — a no-op on
   exactly the terrain it exists for. It fills to the highest rim now.
4. Four placements for the self-dug crater, each failing the same way: removing the *same region*
   every shot so nothing accumulates. At the hull the tank sinks and the roof follows it down
   (30 shots, still buried, burrowing into bedrock); at the muzzle it clears the cell the muzzle
   already had; just above the hull it clears a fixed band the overburden survives. Cutting from the
   **top of the covering downward** is the only one that accumulates.
5. `buried()` was wrong three times (see above), each version producing a readout that disagreed
   with the engine.
6. The harness regenerated terrain without setting `state.seed`, so `restore()` rebuilt a *different
   map* and the invariant reported `tanks` mismatched. My test's bug, not the product's — third time
   this session a harness has abused the API and been caught by `selfCheck` doing its job.


---

## Slice 2b — the specials, and a bug they exposed (2026-09-12)

MIRV, Death's Head, Leapfrog, Funky Bomb. One mechanism — a multi-warhead shell — with different
numbers. Measured in `ladder.html`:

| weapon | warheads | craters |
|---|---|---|
| LEAPFROG | 3 stacked | 3 |
| MIRV | 5 | 5 |
| FUNKY BOMB | 12 scattered | 12 |
| DEATH'S HD | 9 | 9 |

A MIRV's children are **real projectiles with their own ballistics**, so the turn stays open until the
last one lands; a save taken mid-salvo resumes with the salvo still in the air (schema v11, and
`proj`/`extras` joined the invariant, which had never covered in-flight ordnance). The dud rule —
*"if the warhead hits something before reaching apogee, it will not explode"* — is implemented
exactly: one tick into terrain while still rising, zero craters, no split. All four are deterministic
on repeat runs. The Funky Bomb's scatter is seeded, so it is unrepeatable in appearance and perfectly
repeatable in fact.

**And one shippped bug, found by reading rather than by testing** — which is the part worth recording:

```js
if(who === 'player') state.ghost = proj.path;
if(w.smoke) state.smokes.push(...);
else ai.last.x = landX ...         // bound to if(w.smoke), NOT to the player check
```

The smoke-tracer branch added in Slice 1 silently rebound that `else`, so **every shot the player
fired overwrote the computer's memory of where its own last shell landed** — and the computer then
bracketed off *your* landing using its own power-sensitivity maths. The ladder sweep never caught it
because in the sweep only the computer fires. Now a separate statement, with the AI's bracket
updating only from its own shots, and both directions asserted.


---

## Regression report — charged but not registered, and a cropped control panel (2026-09-12)

Both reported by playing the live build. Both real, both mine, both from the last two pushes.

### FIXED — the armory charged credits and registered nothing

Reproduced by driving the real DOM controls at phone size with credits available:

| card | paid | registered | | card | paid | registered |
|---|---|---|---|---|---|---|
| MISSILE | 60 | ✓ | | **MIRV** | 240 | **nothing** |
| NUKE | 520 | ✓ | | **FUNKY BOMB** | 340 | **nothing** |
| DIRT CLOD | 30 | ✓ | | **LEAPFROG** | 200 | **nothing** |
| RIOT BOMB | 120 | ✓ | | **DEATH'S HD** | 420 | **nothing** |

Exactly the four newest weapons. `ammo:[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]` — a hand-written list of
fifteen zeros — was never extended when the catalogue grew to nineteen, so `ammo[15]++` was
`undefined + 1` = **NaN**: charged, stored as NaN, and displayed as 0 because the card renders
`state.ammo[i] || 0`. The restore path had a *second* copy of that same fifteen-zero list, so a
pre-existing save restored short and poisoned itself again.

Fixed by deriving the array from `WEAPONS` (`WEAPONS.map(() => 0)`) and by adding `padAmmo()`, which
pads the array to the catalogue's size and repairs any slot that is not a finite number. The loaded
copy is repaired too, because `selfCheck` compares state against the loaded save — leaving the null
slots in place would fail the check on a now-good save, and a failing check stops autosaving, so a
repaired save could never be written back. The invariant had *already* flagged it: the live page
showed **SAVE REJECTED · ammo**.

### FIXED — the in-combat controls were cropped

The control column measured **497px tall inside a 386px window** and, with `justify-content:center`,
that overflow went **both ways** — the stat row landed at **−64px**, off the top of the screen and
unreachable. In the band the box was 117px inside a 104px band: 6px off each end.

Cause: the weapon chip list grows with the catalogue (8 → 15 → 19 chips), and the HUD had no bound on
it. The catalogue is now bounded and scrollable in both layouts — a horizontal strip in the 104px
band, a vertical grid in the column — so the HUD no longer depends on how many weapons exist. Both
HUD containers also use `safe center`, which degrades to `start` instead of overflowing both ways.

### FIXED (found while reproducing) — TRACER and SMOKE TRC had no armory card at all

They shipped with **no `cat` field**, and the sectioned armory renders by category — so both cards
were silently dropped. The catalogue was 19 cards where it should have been 21. They are Standard
Weapons per the manual, so they have a category now; and the renderer has a completeness guard that
gives a card to any weapon whose category it does not recognise, under an `UNFILED` heading.

### The pattern, which is the part worth keeping

Three separate faults in one day, all the same shape: **a second structure that had to be kept in
step with `WEAPONS` by hand.** Positional indices in `aiPickWeapon` (found by reading). A fifteen-zero
ammo list (found by playing). A renderer that only draws weapons whose category it recognises (found
by counting cards). Two of the three are now derived rather than written, and the third fails loudly
instead of silently.

Verified after the fixes: 21 cards, 4 sections, **every weapon in the catalogue charges once and
registers once**, zero NaN slots, and **nothing off-screen in either field mode**.


---

## Regression report — damage stopped registering (2026-09-12)

### FIXED — every standard weapon carved craters and hurt nobody

Reported from play as "damage appears to no longer register". Cause, in `explode()`:

```js
  carve(x, y, w.r);
  state.fx.push({x, y, r:w.r, t:0});
  if(w.earth){ endShot(...); return; }
  endShot(Math.round(x), 'impact');     // ← the blast() call was never added here
```

When the specials were built, the damage loop moved out of `explode()` and into `blast()` — and the
standard warheads were never given their call to it. Baby Missile, Missile, Baby Nuke, Nuke and the
Roller (which routes through `explode()` via `roller()`) all carved correct craters and dealt zero
damage. The specials were unaffected, because they *do* call `blast()`.

Fixed by calling `blast(x, y, w.r, w.dmg, proj.owner)` on the standard path, with the `w.earth` crater
branch kept separate and deliberately damage-free. Audited every `endShot()` in `explode()` afterwards:
the only damage-free exits are tracer, dirt, Earth Destroying, MIRV dud, off-field, tunnel, the Riot
wedge, and a buried burst — all intended.

### The process failure, which is the part worth recording

**My own regression check printed the symptom and I explained it away.** In the specials verification I
ran:

```
regression_missile: {carves: 1, damage: 0}     regression_babymsl: {carves: 1, damage: 0}
regression_tracer:  {carves: 0, damage: 0}     regression_digger:  {carves: 29, damage: 0}
```

Two of those four should have dealt damage and read 0. I read it as "the shot missed, it landed away
from the target" and moved on — because among those four rows, 0 *is* the right answer for the tracer
and the Digger, and that made the pattern look plausible. A test result that is *compatible* with
success is not evidence of success. Every check in this file counted craters, and craters were still
appearing, so nothing failed.

`ladder.html` now drops every warhead point-blank on a tank and asserts the damage explicitly, with
Earth weapons judged on damage that a fall does not explain.

Verified after the fix: all nine damaging weapons hurt (14–100hp), and every deliberately harmless
one deals exactly 0.


---

## Two interaction faults from play (2026-09-12)

### FIXED — a buried shell's impact appeared at the surface, not at the barrel

Carving only at the top of the covering (the one placement that accumulates) meant the crater opened at
the skyline, reading as a shell impacting in mid-air above you. Carving only at the muzzle is the
opposite failure: it clears the single cell the muzzle occupied and nothing beyond, and the tank can
never dig out. A shell bursting in confined space does both, so now there are two craters per shot —
one where the impact is SEEN, one where the digging HAPPENS. Measured after: the covering descends 10px
per shot, and the dig-out cost is unchanged at 3 / 5 / 10 turns for a Clod / Ball / Ton.

### FIXED — a tank dropped through the floor was lifted to the nearest high ground

`groundUnder()` skipped any column with `surf[x] === H` as "bottomless". So when a crater punched
through under a tank, the only surviving columns in its footprint were the crater's **rim**, and the
tank was placed on the highest nearby ground instead of falling into the hole it had just been dropped
into. `solid()` returns true at `y >= H` — the floor of the map IS a surface — so those columns are now
counted, a tank rests on the floor, and the old "dug clean through → destroyed" rule (which would have
killed every tank that reached the bottom) is gone. Arriving at the floor is now an ordinary landing
with ordinary fall damage, which is also what lets a parachute mean something down there.

### And the defect I introduced while fixing them, worth recording

Tidying stale comments, I replaced a slice of text running from one obsolete comment block to another.
The `const r` declaration and the `state.fx.push` line were sitting **between those two anchors**, so the
slice deleted live code and the whole buried branch threw `ReferenceError` at runtime. Caught
immediately by the next test run. The lesson is not "don't tidy comments", it is: **after any
text-slice edit, assert that what was removed contained nothing but comments.**

### The definitional pattern, once more

The buried branch asked *"is the muzzle's own cell solid"* while the HUD and `dropTanks()` asked
*"is there a roof overhead"*. Those two agree right up until something clears the muzzle — which this
branch's own barrel crater does on the first shot. After that the tank stopped bursting entirely and
fired ordinary shells for the rest of the match. There is now exactly one `buried()`, and the HUD, the
fall handler and the firing check all call it.

Verified: the covering descends monotonically 110→160 over five shots with a crater at the barrel every
time, Clod/Ball/Ton dig out in 3/5/10, the Riot Charge still frees in one turn, a punch-through leaves
the tank resting on the floor and alive, and ordinary falls are unaffected.


---

## The new-game screen (2026-09-12)

The armory was doing two jobs that pull in opposite directions: a *shop* (spend your bank, keep it
between matches) and the place a *match gets defined* (who you fight). Different kinds of decision, so
they are now different screens.

**Flow: NEW GAME → ARMORY → MATCH.** The setup screen is the entry point when no match is running, and
the armory has a `← SETUP` button so walking back to change something never costs you your shopping.
`NEW` starts at the setup screen, not the armory.

- **Moved out of the armory:** the opponent roster. It is a match setting, not a purchase.
- **Stated, not faked:** `ARMS LEVEL`, `TERRAIN`, `COMPUTER SKILL` appear as the current *facts*
  ("every weapon and accessory is on sale", "the only generator built so far", "one rung for the whole
  match"). Each becomes a real control with the work that makes it real — a dead button would be worse
  than none.
- **Persistence:** which screen you were on is part of the save, so locking the phone in the armory
  comes back to the armory and a lock mid-match comes back to the match with the turn intact.

### The first schema migration (v11 → v12)

`armory: true|false` became `screen: 'setup'|'armory'|'match'`, and rather than refusing a v11 save the
reader **carries it forward**: `armory ? 'armory' : 'match'`. The mapping is total — the old boolean
only ever meant those two things — and a schema change is not a reason to throw away someone's bank
balance, record and match in progress. It runs *before* the version check and before the key-shape
check, because the shape check compares against `snapshot()` exactly and the migration is what makes
the keys line up again. A v9 save is still refused, so the migration does not over-reach.

### Two mistakes worth recording, both in the verification rather than the feature

1. **`readSave()`'s `catch(e){ return null; }` hid its own failure.** When the reader throws, the save
   silently becomes *no save* and the player is told nothing. It now logs the error and says so on the
   HUD. A swallowed exception in the save path is the same class of fault as a readout that disagrees
   with the engine.
2. **Two end-to-end tests were invalid because the app overwrote the fixture.** Writing a synthetic
   v11 save from a running page and then reloading does not test the migration: the app's own
   lock-survival autosave fires on the way out and writes the *current* state over the fixture. The
   migration had to be proven by unit-testing `readSave()` directly, plus `restore()` and
   `showScreen()` in isolation — at which point the whole chain passes: a v11 save with `armory: true`
   loads as `screen: 'armory'` and the armory is shown.

Verified: the flow both ways, the opponent pick sticking, 21 cards, the invariant at v12 with an empty
mismatch list, the round-trip, and the migration's both branches with a v9 still refused.


### A refused save is no longer defenceless

`saveBad` blocks autosaving when the *invariant* fails, but a **version or shape refusal** just
returned `null` — and the fresh state that follows would overwrite the stored bytes on the very next
autosave. A schema change is not a reason to throw away someone's bank balance, so a refusal now stashes
the raw save under `scorched.save.v1.refused` with the reason and a timestamp before the game moves on
without it. Verified: a v9 save and a shape mismatch both return `null` *and* stash their bytes; a good
save loads and is not stashed.

### And a note on the verification, which cost more time than the feature

Three separate end-to-end attempts to prove the v11 migration by writing a synthetic v11 save and
reloading all failed — because **the app's own lock-survival autosave fires on the way out and writes
the current state over the fixture**. So the migration kept being tested against a save I had not
written. The app was right every time; the test was wrong every time. It is the same shape as every
other false alarm this session: a harness driving the game in a way the game never drives itself.

The lesson taken: when a fixture must survive from one page load to the next, *nothing else may be
running* — or the assertion has to be made at a level the autosave cannot reach. The migration was
therefore proven by unit-testing `readSave()` directly, plus `restore()` and `showScreen()` in
isolation: a v11 save with `armory: true` loads as `screen: 'armory'`, the migrated keys match
`snapshot()` exactly (35 keys, no differences), and the armory is shown.


---

## The slider scheme: both axes horizontal (2026-09-12)

Variant C was angle-across / power-up. The problem with that is not taste, it is geometry: vertical
travel is the thing this game has least of. In the **wide** field the HUD band is **104px** tall, while
the same control had ~386px in the **4:3** column — so one of the two field modes was always going to
be the awkward one. One axis for both means the scheme reads and behaves the same in either layout, and
the gesture is the same gesture twice: **drag across to set the value.**

```
ANGLE                    45°
[=====|          ]              ← 32px rail, knob and fill
POWER  520        WIND +74
[========|      ]               ← same gesture, same direction: left 0 → right 1000
```

- **Identical mapping on both rails**, verified by sweeping each at 0/25/50/100% of its width:
  angle → `1° / 46° / 91° / 180°`, power → `0 / 250 / 500 / 1000`. Knob position and fill width agree
  with the number (power 520 → both 52%).
- **Precision is unchanged.** A rail is about 3 units per pixel, so the steppers remain the way to
  land an exact figure. The rail is for coarse aiming, the steppers for the last few units — the same
  division the split pad already had.
- **The value is written on the rail**, right where the thumb is, and the wind moved up beside POWER
  instead of occupying a row of its own.
- **Both rails got a fill and a knob** — previously the angle rail had only a knob and the power rail
  only a fill, so they did not read as the same kind of control.

### Measured, not assumed

The first attempt overflowed the wide-mode band by 13px, and the second by 5 — both caught by
measurement, and both the same clipping class that has bitten this project twice. The arithmetic that
settles it, in the 104px band:

| | content height |
|---|---|
| gap 6 + 34px tracks | 98px → **5px over** |
| gap 2 + 32px tracks | **88px → 5px inside** ✓ |

The rail *block* (label 10 + gap 1 + track 32) lands at 43px, and the knob overhangs it to 40px, so the
grab affordance is close to the 44px target; `@media (max-height:430px)` shrinks the tracks to 26px for
a short window, the same breakpoint the armory uses. Verified in the wide mode: **0 elements past the
band, 0 past the viewport.**

### One thing found while measuring, and left alone

The `FIRE` button is 34px and the weapon chips render close to the band's bottom edge. Nothing is
clipped at 1280×577 and no scroll container is involved, so this is not a defect today — but on a
narrower window those chip rows are the thing to watch, and they belong in the real-device pass rather
than in this change.


### When the deploy stalls: clear the concurrency lock

Four pushes in quick succession left the Pages workflow with runs stuck in `queued`/`pending` and the
live site four commits behind — while `gh run list` showed the *oldest* stuck run holding the slot.
This is a known GitHub bug (`concurrency: group: pages` serialises runs and a stalled run blocks the
queue behind it). **The remedy is to cancel the stale queued runs** — `gh run cancel <id>` on each
non-completed run — after which the newest run starts immediately. Verified: cancelling the two stuck
runs took the newest from `pending` to `completed success`, and the served bytes then matched local
exactly.

Worth knowing because the symptom looks like a caching problem and isn't: check
`gh run list --json status,conclusion` before blaming the CDN. If the newest run is not `completed
success`, nothing is wrong with the code and nothing needs re-pushing.
