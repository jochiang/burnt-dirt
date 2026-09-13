# Burnt Dirt — mobile-native Scorched Earth

**Naming:** the project is *Burnt Dirt*. The 1991 original it answers to stays
*Scorched Earth*. Two synonyms, one doctrine.

**One line:** a touch-first Scorched Earth for the two-to-five minute gaps — waiting on a
build, a deploy, a test run. Landscape. One thumb. Played against the computer.

## The bet

Scorched Earth's pleasure is *allocation under uncertainty*: you commit to a shot before you
know what the wind takes from it. Modern artillery games (Worms, and the Worms-shaped ports
of everything since) optimized for a different pleasure — performance. Characters, deaths,
movement, forgiveness. They also cut the part that looks like a spreadsheet: the buy phase.

That leaves a real gap. Not a business, necessarily, but a gap.

## Session shape (the constraint that drives everything)

Sessions are **2–5 minutes, unpredictable start, unpredictable stop, one hand, often no
audio, sometimes bright sunlight.** That is not a compromise — turn-based artillery is
*natively* made of interruptions. A queue of turns with a wind number on top.

But it cuts:

- **The buy phase does not fit inside a session.** Open an armory mid-gap and you have lost
  the gap. → The economy lives *outside* the match: you set a loadout, then matches are pure
  shots. Armory is a screen you visit, not a phase you sit through.
- **The AI turn must be fast or absorb-able.** With ninety seconds you cannot spend forty
  watching an enemy arc. → Constant-tick simulation decoupled from frame rate, ~3× real time.
- **State has to survive an interruption that arrives without warning.** → Deterministic
  match state, persisted at every turn boundary *and* on `pagehide`/`visibilitychange`/`blur`.
  Terrain is not reconstructible from the seed once craters exist, so a match is stored as the
  seed plus the ordered list of literal carve operations plus tank/AI/log state — ~860 bytes,
  against 120,000 for the raw mask. **Solved and verified against real reloads.** The save path
  is also governed by an enforced invariant — `restore(snapshot())` must be an identity on the
  settled match state, checked at load time and failed safe — because every persistence bug found
  was a violation of it. See spike 001.

## What is fixed — the precision argument, made structural

The claim: **precision is a property of the physics, not the input device.** In 1991 that was
literal. 320×200, integer fixed-point trajectories, finite angles, finite powers, finite
winds. The solution space was *discrete and enumerable*, which is why the game felt exact. A
float-physics remake would be *less* precise than the DOS original, because there would be
infinite shots between the two you already tried.

Non-negotiable, therefore — and now **verified in spike 001**:

| Quantity | Domain | Why |
|---|---|---|
| Angle | integer, 1–180° | finite, exactly re-enterable |
| Power | integer, 0–1000 | finite, exactly re-enterable |
| Wind | integer, −100…100, re-rolled per round | the thing you cannot buy |
| Simulation | fixed timestep, integer inputs, seeded RNG | same input ⇒ identical shot, always |

**Consequence:** whatever the thumb does, the game resolves on the same grid it did in 1991.
The finger is not asked to be precise — the *space of answers* is small enough to find.

Corollary: **the skill is bracketing, not solving.** Nobody solves for the wind analytically.
You shoot, observe, correct. Tracer was the original admitting this out loud. Bracketing is an
iteration tool, and it is ideal for short sessions, because the third shot landing is the reward.

One more thing the spike surfaced: **destroyed ground invalidates a solved shot.** The computer
brackets to within 5px, and then the craters it has dug near you change the impact geometry, so
the same shot now lands 120px long. Terrain is not scenery. It is a live counter to precision,
and that is the best argument that precision is worth having.

## Field shape

The original was VGA mode 13h — **320×200**, which is 8:5 in the framebuffer, but mode 13h had
non-square pixels and 4:3 CRTs stretched it, so the field as *seen* was **4:3**. What matters
most for feel is the scale ratio: **320 ÷ ~9px tank ≈ 35 tank-widths across.**

| | 4:3 faithful (default) | Wide |
|---|---|---|
| Screen | 520×390 | 844×301 |
| Tank-widths across | 36.4 | 63.6 |
| Wind drift, max, high lob | 45% of field | 49% of field |
| HUD | 176px side column | 104px bottom band |

The wide field does **not** make wind matter more — that was the assumption, and it is wrong.
Because VMAX scales with √W to keep the dial calibrated, flight time scales with √W and drift
scales with t² ∝ W; the two cancel. What the wide field really changes is target size: more
tank-widths, smaller tanks, more bracketing required.

**4:3 is the default** for a structural reason: on a 19.5:9 phone a 4:3 field leaves a 176px
side pillar, and that pillar is exactly what a full-height one-thumb control column wants.

## Layout

- **Landscape.**
- **All controls in one column**, mirrored left/right for handedness.

- **Fullscreen is a first-class control, not a glyph.** There is a labelled `⛶ FULL` in the top bar
  and a `⛶ FULLSCREEN` beside START MATCH in the armory, because the moment you decide to play is
  the moment you want the browser chrome gone. It is a **toggle** (enter *and* exit) and the label
  tracks the real state. `fullscreenchange` re-measures explicitly: entering fullscreen shrinks the
  viewport, some browsers fire no `resize` for it, and a layout that doesn't re-measure is a field
  drawn for the wrong shape.
- **iOS gets the truth, not a dead button.** iPhone Safari has no `requestFullscreen` (only on
  `<video>`), so the control reports `⛶ ADD TO HOME SCREEN` and the message says
  *SHARE ▸ ADD TO HOME SCREEN* — the only real fullscreen iOS offers is a standalone PWA window.
  A button that silently fails is worse than no button.
- **The controls are never drawn over the field.** Not "mostly clear" — architecturally
  disjoint rectangles. This was learned the hard way: a 16:9 field and a screen-height HUD
  cannot coexist at 844×390, and moving the tanks around only relocates the occlusion.
- The control column must carry enough state — angle, power, wind, last shot — that you never
  have to look down at it. **The field is the feedback. The HUD is the instrument panel.**

## Controls — decided by spike 001

- **Split pad** (one drag: X = angle, Y = power), both axes integer-quantized on release,
  ±1°/±5 nudges for exactness. **Adopted.**
- Sliders (two detented tracks). Works, less elegant. Kept as a toggle.
- Direct aim (drag the field to point the barrel). **Rejected — unrepeatable, and the angle
  collapses to ~0° anywhere at or below the tank's height.**

Haptics are unresolved and are a platform question: `navigator.vibrate` exists on Android
Chromium, not on iOS Safari. If the 10°/50 detent feel is load-bearing, that decides a
platform, not a button style.

## The armory — the economy, out of the match

The buy phase does not fit a two-minute gap, so it lives outside the match. What you carry in is
**stock**: you commit to a loadout *before* you know the map or the wind, and every shot is a
decision you have already paid for.

| item | price | what it buys |
|---|---|---|
| BABY MSL | free · unlimited | 22 dmg, blast 9 |
| MISSILE | 60 | 40 dmg, blast 15 |
| BABY NUKE | 180 | 58 dmg, blast 24 |
| NUKE | 520 | 85 dmg, blast 38 |
| DIGGER | 140 | **0 dmg** — bores a tunnel; hurts by dropping what stands on it |
| ROLLER | 260 | 60 dmg, rolls downhill |
| ARMOR | 200 | absorbs 40 damage before the hull does |

- Start with **1000 CR**. Payout after a match = **damage dealt + 400 for a win**. Selling back is
  a full refund — but only in the armory. **Nothing is purchasable mid-match.** That lock *is* the
  commitment; without it there is no decision, only a shopping trip.
- The computer plays the same economy from its own bankroll, earning from damage it deals, and it
  climbs the price list as it wins. It cannot buy in bulk, so it spends per shot instead: **you get
  to plan, it gets to adapt.** A losing computer is stuck with cheap shells, which is a better
  difficulty curve than a slider.
- **The free shell is what makes bracketing affordable.** Paid shells are for confirmation. That is
  the original's rhythm, and it is where the wind finally costs money: a miss on a 520 CR nuke is a
  miss you paid for.

Screens are strict about which one owns the input: the armory and the match-over card are never
both open.

**The armory scrolls rather than clips.** The grid carries `align-content:center; align-content:safe
center`. Plain centring is a trap with `overflow:auto`: when the rows are taller than the box the
overflow goes *above* the scroll origin, where no scroll offset can reach it, so the top row is
clipped and unreachable. `safe` degrades to `start` on overflow — a tall screen still centres, a
short one starts at the top and scrolls. That distinction only bites in a windowed browser, where
phone toolbars take roughly 100px of a 390px landscape height.

**An empty gun hands you the free shell.** Firing with nothing loaded names what ran out and moves
the selection to BABY MSL immediately. The swap is *not* deferred, and that is deliberate: `wIdx` is
in `SETTLED`, so mutating it from a timer would make `selfCheck` report the board unsettled and stop
autosaving. The message lingers; the state change does not. Since the free shell is infinite, there
is always somewhere safe to land the player.

**The computer keeps its own weapon.** It used to write its pick to `state.wIdx` — the *player's*
selection — so its turn silently replaced the player's loadout choice while the HUD went on
highlighting the old chip. You would plan a NUKE, the computer would fire a MISSILE, and your next
shot would be a MISSILE too, with the HUD still saying NUKE. The interface and the fired weapon
disagreed, which is the same class of defect as the DIGGER's label. The computer now has `ai.wIdx`.

### Open

- **Touch targets**: the card steppers are 32×30px, up from 23×20. Better, still short of the 44px
  ideal — fine for a secondary stepper, not fine as a primary control.
- Half-used armour cannot be sold back (packs are 40 damage), which is deliberate but will read as
  a bug to anyone who tries.

## Falling — dirt is rock, tanks are not

Straight from the 1991 manual, because both halves are counter-intuitive and both are defaults:

> **Suspend Dirt** `0-100`, default `0` — *"This number allows dirt to remain suspended in the air
> some of the time. It is a percentage chance, per shot, that all the dirt on screen will fall."*

> **Tanks Fall** `ON/OFF`, default `ON` — *"If this is turned off, tanks will not fall when the
> ground is shot out from beneath them. Not very realistic, but an option nonetheless."*

Read the first one carefully: the number is the chance dirt **falls**, so at the default of `0`
dirt **never falls**. Terrain left hanging after an explosion stays hanging — that is the
original's normal state, which is why arches, tunnels and overhangs are legitimate geometry and
not a bug waiting to be fixed. The second is the mechanic we were actually missing: tanks obey
gravity, taking damage based on the drop, with Parachutes as the counter.

- **Fall trigger**: any terrain change re-tests the footprint under each tank (5px either side).
  The tank descends to the deepest support in that footprint, so a tank on a crater lip slides in.
- **Damage**: nothing under 10px; then 1 hp per 3px fallen. A 33px drop costs 8 hp.
- **Dug clean through**: if no column in the footprint has ground at all, the tank falls out of
  the world and is destroyed. Sandhog under a tank is now a kill.
- **Dirt does not move.** Deliberately, and per the manual.

The fall is instantaneous in the model and *animated only in the drawing*, via a cosmetic offset
that lives outside `state` — so nothing new is persisted, nothing can be double-applied on
restore, and a page reload mid-fall simply shows the tank already landed. Tank `x`/`y` are derived
from the terrain rather than restored, and are now in the snapshot so the invariant check asserts
that the terrain replay drops the tank exactly where the live board had it.

## Accessories — Parachutes

The original's own table: **"Parachute $10,000, Bundle Size 8, Arms Level 2"** — 1,250 each, and
an *accessory* rather than a weapon. Ours is 80 CR, scaled against the Shield/armour ratio, and
**consumed only when one actually opens**.

The manual's rule, implemented literally:

> *"If you are going to fall, an onboard system computer looks down and estimates how much damage
> you tank will take from the fall. If your parachutes are deployed, and the safety threshold is
> less than the amount of damage you will take, the parachutes activate, and your tank takes no
> damage from the fall (unless it lands on an enemy tank.) If your parachutes are passive, or the
> precomputed damage is less than the safety threshold, then you will fall without parachutes, and
> take damage."*

- Default **deployed**, threshold **5** — the manual's defaults, not invented ones.
- Count and state on the HUD, greyed at zero, one tap to toggle — the original's status bar.
- The threshold dial (1–100) sits on the armory card. The original kept it in the Tank Control
  Panel; a phone has no such panel, and the armory is where accessories are bought.
- **A chute cushions a landing. There is no landing at the bottom of the world** — so a bottomless
  fall is never saved, and never spends a chute.
- The manual's "*unless it lands on an enemy tank*" is unreachable here: tanks never move, so two
  of them are never adjacent. Left unimplemented rather than written as dead code.

Because a chute opens only when the predicted damage clears the threshold, digging under a
defended tank is a war of attrition: each intermediate collapse can burn a chute, and only the
final one puts them in the ground.

## Terrain is 2D by construction

The engine was already 2D: `mask` is a full bitmap, `solid(x,y)` reads it, `redrawTerrain()` paints
it per-pixel, and `carve()` punches discs out of it. The only one-dimensional thing left was
`genTerrain()` — it filled everything below a height *curve*, which by construction cannot produce
an overhang. Terrain is now that heightmap **plus stamped features**: arches with a hanging roof,
enclosed caves, towers, shelves, and rare detached islands. All of it written into the same mask,
so collision and rendering needed no changes at all.

- Features draw from their **own RNG stream** (`frnd = mulberry32(seed ^ 0x9e3779b9)`), so the
  heightmap and the wind are unchanged for a given seed no matter how the feature code evolves.
- `surf` is now **derived from the mask** (`syncSurf()`), never the reverse. With 2D rock the
  topmost solid pixel is the only meaningful surface, and tank placement and targeting must agree
  with collision rather than with a curve that no longer describes the map.
- Features keep out of the outer 16% of the field, so both tanks always have level ground and a
  clear firing arc.

**Verified across 100 seeds:** no entombed tanks, no spawn without ground, byte-identical maps on
regeneration, mean surface relief 72px of a 300px field, and **0/100 maps lacking both a wall and
an overhang** — so every map has real vertical structure, not just the ones that happened to roll
an arch.

## The DIGGER was lying

Two defects, both cases of the label outrunning the code.

1. **Its `tunnels` flag scraped the sky.** It carved the last 22 points of the *approach path* — but
   a projectile stops at first contact, so those points are the air it just flew through. Measured:
   carving the first three of them (x=232, 238, 245) leaves the solid count *unchanged* at 45738.
   A digging warhead now carries on **into the rock along its heading** for `r*2.6` px — 773 rock
   cells removed against 293, and 10 covered tunnel cells against 3.
2. **It advertised `dmg 18` and did direct damage on contact.** The manual is explicit: *"Most of
   these weapons cannot directly harm a tank, though they can cause them to fall and take damage
   that way."* Damage is now 0 and the armory says what the weapon actually does. Its real
   mechanism — removing the ground from under a tank — only became *true* once falling tanks landed.

And per the manual, *"If they hit a tank, they fizzle"* — confirmed: a Digger passing 3px from the
enemy does nothing at all.

The payoff: a 12°/950 Digger bores a 29px lane, and the **identical follow-up shot travels 262px
further and clean off the field** (impact 366 → 628). It makes its own firing lane through rock.
## What is cut

- Buy phase *inside* the match — built as the out-of-match armory instead
- Movement, characters, dialogue — that is Worms, and Worms already exists
- Real-time anything. No timers. No reaction tests.
- Bounced shots, tornadoes — later, and only if they serve wind. (Falling *dirt* is
  not deferred: the manual's default is that it never falls. Falling *tanks* was the real gap.)

## Open — everything not done, in one place

**Opponents and armory have their own design doc:** `DESIGN-opponents-and-armory.md`.
Both named priorities hide a question larger than their name — difficulty is the *opponent*
(the 1991 manual contains no difficulty setting at all; it contains a roster), and "more
armory content" requires a BURIED state that the existing 2D mask already enforces for free.
Its §8 carries the decisions still open. **Slice 1 is built**: the six-rung opponent ladder
(MORON/SHOOTER/TOSSER/SPOILER/CHOOSER/UNKNOWN — chosen in the armory, saved with the match, schema
v9) and the TRACER / SMOKE TRC information weapons. **Slice 2 is built**: the Earth family (Dirt
Clod / Ball / Ton, Liquid Dirt, Riot Charge / Blast / Bomb), the BURIED state with a HULL readout,
and typed terrain ops — a crater and a deposit are different operations now, so the ordered op list
carries a type (schema v10). Every terrain change remains a literal recorded op. **The specials are
built too**: MIRV / Death's Head / Leapfrog / Funky Bomb, one multi-warhead mechanism with different
numbers, children as real projectiles (schema v11, which also brought in-flight ordnance into the
invariant for the first time). **Two regressions reported from play are fixed** — see the README: the armory
charged credits without registering (a hand-written ammo list that outlived the catalogue), and the
in-combat controls were cropped (the chip list outgrowing the HUD). Both were the same shape: a second
structure kept in step with `WEAPONS` by hand. The ladder is measured rather than asserted —
see `spikes/001-touch-controls/ladder.html`, which re-runs the sweep.

Kept here so it does not have to be reconstructed from conversation. Three kinds, deliberately
separated, because "unfinished" and "broken" are not the same claim.

### Defects outstanding — none known

The three that were open are fixed and verified:

| what it was | fix |
|---|---|
| Off-field miss published as a distance | an off-field landing is where the shot *left the world*, not where it would have hit, so printing it as `+207px` handed out a bracketing number that couldn't be bracketed. Now flagged and rendered `OFF · LONG` / `OFF · SHORT`. The AI's bracket memory is clamped so a wild miss is still correctable. |
| Mid-air save vs a physics retune | the save stored `angle/power/t` and the resume **re-fired from the muzzle**, re-simulating the flight that had already happened. Measured: a 5% `VMAX` change moved the landing 40px. The save now carries the shot's real `x, y, vx, vy` and resumes from there. The flown trail restarts — the price of the outcome being reproducible rather than plausible. A physics fingerprint is stored too, so retuning `GRAV`/`WIND_A` (which still affect the *remaining* flight) is announced: `PHYSICS CHANGED · SHOT MAY LAND DIFFERENTLY`. |
| Rotation while backgrounded | nothing re-measured on return, so a rotate-while-locked kept the old layout indefinitely (`fit 1.923, cw 1280` frozen). `visibilitychange → visible` now re-measures and re-renders, and `resize()` ignores non-positive dimensions so a hidden tab reporting 0×0 keeps the last good layout instead of collapsing the field. |

### Not built yet

| what | notes |
|---|---|
| **Smarter opponent** | it brackets its own last landing and jitters. It does not lead the wind, use terrain, or remember more than one shot. This is the biggest lever on whether the game is worth replaying. |
| Rest of the earth-moving family | SANDHOG, LIQUID DIRT (which *adds* dirt — and with falling tanks, burying becomes a mechanic), RIOT BOMBS, DIRT CLOD/BALL/TON. The manual's own tables are the spec. |
| Rest of the specials | MIRV, DEATH'S HEAD (fractal lattice), LEAPFROG, FUNKY BOMB, NAPALM. |
| Rest of the accessories | Guidance (heat/ballistic/horz/vert), Batteries, Shields (3 tiers), Mag Deflector, Super Mag, Auto Defense, Jump Jets. |
| `Suspend Dirt` toggle + Earth Disrupter | the manual's *other* half of the terrain system. Default stays 0. Only interesting as a pair. |
| Armour top-up | half-used plating can't be sold back (packs are 40 damage). Deliberate, reads as a bug. |
| Bigger touch targets | armory steppers are 32×30, short of the 44px ideal. |
| Terrain texture tuning | the 9-row strata (±13) read as strong horizontal bedding once the blit is crisp. Defensible as sedimentary layering, prominent either way. |
| Rougher towers | the tower feature is a plain rectangle by construction; tapering or roughening its sides would stop it reading as a placed block. |
| Sound | sessions are often silent and sometimes in sunlight — deliberately absent so far. |
| PWA / offline / installable | the build shape's "if it graduates" step. Nothing needs a server, so it's packaging only. |

### Cut by design — not backlog

- Buy phase inside a match (it is the out-of-match armory instead)
- Movement, fuel, characters, dialogue — that is Worms, and Worms exists
- Real-time anything; timers; reaction tests
- Bounced shots and tornadoes, until they serve wind

### Platform decision outstanding

**Haptics.** `navigator.vibrate` exists on Android Chromium and not on iOS Safari. If the
10°/50 detent feel is load-bearing, that decides a *platform*, not a button style. Currently the
HUD reports which is available and the game works either way.

## Build shape

Single-file HTML, canvas + DOM HUD, no dependencies, no backend, no build step. Runs from a
phone browser over LAN. If it graduates: PWA, offline, installable. **No server** — there is
nothing here that needs one.
