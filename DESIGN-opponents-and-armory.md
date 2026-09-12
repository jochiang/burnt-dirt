# Burnt Dirt — opponents and armory: design

**Status:** DRAFT. Nothing here is built. Decisions are marked *decided*, *proposed* or *open*.

**Grounding:** the 1991 Scorched Earth manual, on disk at `/tmp/se-manual.txt`. Line references are
to that file, and quotations are verbatim. Where I have invented something, it says so.

**Reads with:** `SPEC.md` (the design record) and `spikes/001-touch-controls/README.md` (the
verification record).

---

## 1. Why these two, and why in this order

Two priorities were named: **AI difficulty settings** and **more armory content**. Both are right,
and both hide a design question that isn't obvious from the name.

The armory question hides a *mechanic*: the biggest category of missing content — Earth Producing —
exists to bury tanks, and we have no burial rule. So "more armory content" is not a content
exercise, it's a state-machine exercise (§3).

The AI question hides a *structure*: difficulty is usually built as a scalar error multiplier, which
produces an opponent that is bad at everything uniformly. The manual's own ladder does the opposite
(§2).

**Order: opponents first.** More weapons against a weak opponent only widen the win, and the
computer buys from the same catalog the player does — so new items are only interesting once the
computer can use them.

---

## 2. Opponents

### 2.1 What the manual documents

`/tmp/se-manual.txt:367` — "Available Computer Opponents". Eight entries, quoted:

| Opponent | Manual, verbatim (line) | The competence it encodes |
|---|---|---|
| **Moron** | "you can't get much stupider than this. Morons just pick an angle and power, and shoot." (372) | none — no feedback is used |
| **Shooter** | "can be significantly deadlier than Morons, but only if they have a straight line of fire." (377) | line-of-sight |
| **Poolshark** | "Poolsharks act like Shooters unless you are using rebounding walls. Then they try to rebound shots off of the walls and ceilings" (381) | bank shots |
| **Tosser** | "start out like Morons, but they'll refine their aim to get closer and closer, until they hit." (386) | bracketing |
| **Chooser** | "have all the above methods available to them, and decide which one will be most effective." (392) | method selection |
| **Spoiler** | "Taking into account the wind factor and gravity, they will get a perfect shot almost every time, assuming nothing is in the way." (396) | solves for wind |
| **Cyborg** | "methods similar to the Spoilers, but are much nastier about choosing targets. They will tend to attack tanks who are weakened, winning, or have attacked them in the past." (403) | threat weighting and grudges |
| **Unknown** | "one of the above will be chosen randomly to control the tank, but you will not be notified of what the selection was!" (409) | — |

Two things worth noticing. First, **difficulty is not a dial anywhere in the manual** — the word
"difficulty" appears zero times in 2,724 lines. Choosing an opponent *is* the difficulty setting.
Second, the ladder is ordered by **competence**, not by how much it misses: a Spoiler isn't a Moron
with a smaller error radius, it's an opponent that solves a problem the Moron doesn't know exists.

### 2.2 Where we currently sit

**Our AI is a Tosser.** It fires, measures the miss, and corrects off its own last landing — 
converging to within ±5px by turn 2–3. It does not lead the wind (which is why its error grows with
wind), does not test line of sight, and does not spend deliberately.

That's a useful calibration point: the manual's third rung, out of eight.

### 2.3 Architecture: competences are atoms, opponents are data

The mistake would be to write four AI functions. Instead, write the **competences** once and express
each opponent as a set of flags — so adding an opponent later is a table row, not a code path.

Proposed primitives:

```
solveRandom()      pick angle+power, ignore everything that happens after
hasLineOfFire()    is there a clear lane from my muzzle to the target
bracket()          derive the next shot from where the last one landed   [exists today]
solveForWind()     integrate wind+gravity to compute the shot directly
bankShot()         reflect off ceiling/walls            [N/A today, see 2.4]
chooseMethod()     pick the most effective of the above
weighTargets()     prefer weakened / winning / grudge targets
chooseWeapon()     pick from the catalog by situation   [economy hook exists today]
hideIdentity()     choose one of the above, never reveal which  [Unknown]
```

Each opponent then becomes:

```
Moron        { solveRandom }
Shooter      { hasLineOfFire ? exact : solveRandom }
Tosser       { bracket }                                       ← what we have
Spoiler      { solveForWind }
Chooser      { chooseMethod(solveRandom, hasLineOfFire, bracket, solveForWind) }
Cyborg       { solveForWind, weighTargets, chooseWeapon }
Unknown      { hideIdentity(any of the above) }
```

### 2.4 What transfers to our engine, and what doesn't

*Decided:* Poolshark is **not buildable** and should be named as such rather than faked. We cut
rebounding walls deliberately, and the manual is explicit that Poolshark only differs from a Shooter
*when walls exist*. If walls ever return, `bankShot()` is a single-competence addition.

*Open:* Cyborg's target selection collapses in a 1v1 duel — there are no targets to weigh. The
*grudge* survives cleanly, though: remember who hit you, and spend the good ammo when they're
already hurt. Proposed reading: Cyborg becomes the opponent that **finishes you**, rather than one
that picks whom to attack.

### 2.5 Determinism — the constraint the original never had

The original didn't need reproducible matches. We do: the whole persistence story is seed plus an
ordered list of terrain ops, so a resumed match must be bit-identical to an uninterrupted one.

Consequences:

- Every opponent's decision must be a **pure function of (state, seed)**. We already have the seeded
  stream (`mulberry32`, with a separate `frnd` for terrain features) — no `Math.random` may enter.
- The opponent becomes **part of the match's identity**, so it must be recorded in the save. Save
  schema goes to **v9**: `opponent` joins `SETTLED`, or a resumed match is played by a different AI
  than the one that started it. This is precisely the class of bug the invariant exists to catch.
- **No changing opponent mid-match.** It is a setup choice, which means it belongs in the armory.

A pleasant side effect: because the ladder is deterministic, we can *measure* an opponent's skill
(§6) instead of asserting it.

### 2.6 Where it's chosen

In the armory, with the rest of the setup — consistent with the existing principle that the armory is
the front door and setup controls belong where the intent is. The opponent's name is displayed;
*Unknown* is the exception, and that's the joke.

### 2.7 Proposed ladder for our engine

| Rung | Flags | Expected behaviour |
|---|---|---|
| MORON | `solveRandom` | rarely lands; a beginner's first win |
| SHOOTER | `hasLineOfFire` | deadly across flat ground, helpless when a hill is in the lane |
| TOSSER | `bracket` | **today's AI** — converges in 2–3 shots |
| SPOILER | `solveForWind` | accurate regardless of wind; only terrain stops it |
| CHOOSER | `chooseMethod(...)` | never wastes a shot on a blocked lane |
| CYBORG | `solveForWind + weighTargets + chooseWeapon` | spends its best ammo when you're already hurt |
| UNKNOWN | hidden selection | no read on what you're facing |

`Chooser` is deliberately listed above `Spoiler`: the manual says it picks among methods, and with
four primitives that's a genuinely different behaviour from only ever solving analytically.

---

## 3. Burial — what "more armory content" actually requires

### 3.1 The manual says burial is intended

`/tmp/se-manual.txt:1253` — "Earth Producing Weapons": *"weapons which take some form of compacted
earth that explodes into a much larger amount of dirt. These weapons can be used to build
fortifications, or bury enemy tanks."*

A Ton of Dirt is *"easily capable of burying someone alive"* (1268), and Riot Charges exist because
*"This weapon's primary use is to unbury yourself when you get covered with dirt"* (1211).

So a BURIED state is a designed mechanic, not an emergent accident. I had flagged this as an open
question when Liquid Dirt first came up; the manual closes it.

### 3.2 The insight: burial needs no new rule

We already have 2D terrain: a `Uint8Array` mask that `solid()` consults and `carve()` punches holes
in. A tank whose muzzle is inside rock is **already** unable to land a shot — every shot it fires
hits the dirt around it. Nothing needs to be written to make burial have teeth. The physics we built
for overhangs and arches enforces it for free.

So this is not a new mechanic. It is three small additions to an existing one.

### 3.3 The three additions

1. **`deposit(x, y, r)`** — the mirror of `carve()`. Same code path, `+1` instead of `−1`. Crucially
   it must be recorded the same way, in the same ordered list, because `state.carves` is described
   in-code as *"every terrain-modifying op in order: the entire persistence story"*. So the op tuple
   grows a type, and the save schema goes to **v9**.
2. **A BURIED indicator in the HUD.** If your tank is under dirt, you cannot see it and cannot tell
   why your shots are pointless. This is a *rendering* problem, not a physics one, and it's the part
   that's actually easy to get wrong.
3. **The remedy.** Riot Charge (a wedge carve around your own turret) and Riot Blast (a wider one).
   Both are shaped calls to the existing `carve()`.

### 3.4 Why the categories are self-consistent

Worth stating, because it's what makes the catalog feel designed rather than accumulated:

- **Earth Producing** = defense. Bury yourself, or bury them.
- **Earth Destroying** = counter-defense. Remove the dirt, re-open the lane, drop them by removing
  what they stand on.
- **Riot Bombs** (spherical, *"They do no damage to tanks"*, 1220) = the specific anti-turtle tool
  for someone who buried themselves.
- Neither category damages a tank directly — matching the manual's rule for Earth Destroying
  (*"Most of these weapons cannot directly harm a tank, though they can cause them to fall and take
  damage that way"*, 1204). All damage comes from falling.

### 3.5 Decisions to make

- *Proposed:* dirt landing on a tank does **no** damage. Symmetric with the rule above, and it keeps
  the arithmetic of "damage is a pure function of fall height" intact.
- *Open:* can a buried tank fire at all? Proposed: yes, and it hits its own surroundings — no special
  case, the mask decides. The alternative (an explicit "you may not fire" rule) adds a state the
  physics already expresses.
- *Open:* does being buried protect a tank from enemy fire? Implied yes, since enemy shots hit the
  dirt first — which is exactly why Riot Bombs exist.

---

## 4. The catalog

### 4.1 The manual's taxonomy

*Decided:* use the manual's own categories rather than inventing tab names —
**Standard / Earth Destroying / Earth Producing / Energy** (1202, 1253, 1285) plus **Accessories**
(1312). Our current 8 cards become a real catalog inside that structure.

The complete source tables, for reference:

**Weapons** (1086) — name, cost, bundle, blast radius, arms level:

```
Baby Missile   $400   10   10      0      Riot Charge    $2,000  10   36   2
Missile        $1,875  5   20      0      Riot Blast     $5,000   5   60   3
Baby Nuke      $10,000 3   40      0      Riot Bomb      $5,000   5   30   3
Nuke           $12,000 1   75      1      Heavy Riot B.  $4,750   2   45   3
Leap Frog      $10,000 2   20/25/30 3     Baby Digger    $3,000  10   N/A  0
Funky Bomb     $7,000  2   80      4      Digger         $2,500   5   N/A  0
MIRV           $10,000 3   20      2      Heavy Digger   $6,750   2   N/A  1
Death's Head   $20,000 1   35      4      Baby Sandhog   $10,000 10   N/A  0
Napalm         $10,000 10  N/A     2      Sandhog        $16,750  5   N/A  0
Hot Napalm     $20,000 2   N/A     4      Heavy Sandhog  $25,000  2   N/A  1
Tracer         $10     20  0       0      Dirt Clod      $5,000  10   20   0
Smoke Tracer   $500    10  0       1      Dirt Ball      $5,000   5   35   0
Baby Roller    $5,000  10  10      2      Ton of Dirt    $6,750   2   70   1
Roller         $6,000   5  20      2      Liquid Dirt    $5,000  10   N/A  2
Heavy Roller   $6,750   2  45      3      Dirt Charge    $5,000   5   N/A  1
Plasma Blast   $9,000   5  10-75   3      Earth Disrupter $5,000 10   N/A  0
Laser          $5,000   5  N/A     2
```

**Accessories** (1314):

```
Heat Guidance    $10,000  6   2      Battery         $5,000  10   2
Ballistic Guid.  $10,000  2   2      Mag Deflector   $10,000  2   2
Horz Guidance    $15,000  5   1      Shield          $20,000  3   3
Vert Guidance    $20,000  5   1      Force Shield    $25,000  3   3
Lazy Boy         $20,000  2   3      Heavy Shield    $30,000  2   4
Parachute        $10,000  8   2      Super Mag       $40,000  2   4
                                     Auto Defense     $1,500  1   3
                                     Fuel Tank        $10,000 10   3
                                     Contact Trigger   $1,000 25   3
```

Two rules I'd hold to:

1. **The bigger-boom ladder is already complete.** Missile → Baby Nuke → Nuke is the whole
   "strictly larger" axis, and the manual adds nothing new to it. New content should add new *kinds*
   of solution — information, terrain, defense, guidance — not new rungs.
2. **No item that is merely a second name for an existing item.** Every addition should change what
   is *possible*, not just the numbers.

### 4.2 Price fitting — a real problem

Our economy is in CR, not the manual's dollars, and the existing scale is **designed, not derived**:

```
Missile      $1,875 →  60 CR   (÷31)
Baby Nuke   $10,000 → 180 CR   (÷56)
Nuke        $12,000 → 520 CR   (÷23)
Digger       $2,500 → 140 CR   (÷18)
Roller       $6,000 → 260 CR   (÷23)
Parachute  $1,250ea →  80 CR   (÷16)
Shield      $20,000 → 200 CR  (÷100)
```

No constant, and the top end is compressed on purpose: a match is two to five minutes, a win pays
damage + 400, so a Nuke at 520 CR is roughly one good match. That's the intended feel — you buy
stock *before* you know the map or the wind, and the free unlimited Baby Missile keeps bracketing
affordable.

*Proposed rule for new items:*

1. Preserve the manual's **rank order** — never price an item at or below a strictly better one.
2. Anchor on the pairs that map 1:1 (the four standard weapons, Digger, Roller).
3. Compress the top so the most expensive item lands at **≈1–2 winning payouts**, not ten. A Super
   Mag at $40,000 must not become a 1,000 CR item.
4. Round to numbers a player can do arithmetic with in their head.

*Open:* the fitted numbers for the full catalog. Slice 1's items are fitted below; the rest should be
fitted when they're actually built, not now.

### 4.3 Arms Level — the original's own gate

`/tmp/se-manual.txt:2211`: **Arms Level, range 0–4, default 4** — *"This lets you disallow the use of
certain items from the game. Using an Arms Level of 0 is often useful for beginners, so there aren't
so many options to deal with... The Arms Level also affects available accessories."*

So the original's answer to "the catalog is overwhelming" is a setup filter, not a difficulty dial —
consistent with §2.1.

*Open, two options:*

- **(a) Faithful gate.** Arms Level as an armory setting, default 4 (everything). Simple, and it keeps
  progression exactly where the manual puts it.
- **(b) Win-gated unlock.** Items unlock as you win. This is a real change to the economy's shape, and
  adds meta-progression the game currently doesn't have. It's a bigger decision than it looks and
  shouldn't be smuggled in with a settings row.

I'd take (a) now and treat (b) as its own question.

### 4.4 The armory as a catalog

The armory is the front door and currently shows 8 cards in a 2-row grid that just fits a windowed
phone viewport. A ~49-item catalog needs:

- category tabs or sections (the manual's four, plus accessories);
- a "re-buy last loadout" affordance, because the armory sits *between* matches and shouldn't become
  a second game;
- the fullscreen/field/handedness controls staying where they are;
- scrolling that works at short viewport heights (already fixed once — the `safe center` trap).

---

## 5. Slice 1 — what to build first

**Slice 1: the ladder + information weapons.**

- Competences: `solveRandom`, `hasLineOfFire`, `bracket` (exists), `solveForWind`, `chooseMethod`,
  `hideIdentity`.
- Opponents: Moron, Shooter, Tosser (= today), Spoiler, Chooser, Unknown.
- Opponent picked in the armory; recorded in the save (v9).
- **TRACER** and **SMOKE TRACER**, the first new armory items.

Why these, and not explosives: both ride systems that already work. The tracer path is *already*
simulated and *already* persisted (it's in the save schema today), so Tracers are close to a data
change. And information weapons are the cleanest possible demonstration that the catalog adds *kinds*
of solution — a Tracer deals no damage at all; it buys a fact, at the cost of a turn.

*Proposed prices:* TRACER 10 CR for 5 · SMOKE TRACER 40 CR for 5. The original prices Tracers at
$10 for 20 and Smoke Tracers at $500 for 10 — a 100× premium for the persistent visual, which our
engine preserves exactly (a lingering smoke trail is a *visual memory aid*, which is the whole
point).

**Slice 2: burial.** `deposit()`, the BURIED indicator, Riot Charge and Riot Blast, Dirt Clod / Dirt
Ball / Ton of Dirt / Liquid Dirt. Save schema v9 covers both slices if the carve list gains its type
field once.

**Slice 3: the rest of the catalog**, fitted prices, Arms Level, and the remaining accessories.

---

## 6. Verification plan

The project's standard is that claims get measured, so each of these is a test, not an assertion:

**Is the ladder actually a ladder?** Run 100 seeds per opponent and measure *shots to first hit*.
Expected: Moron high and erratic, Tosser 2–3, Spoiler 1. If Spoiler isn't strictly better than
Tosser across seeds, the competence isn't implemented.

**Does Spoiler actually solve for wind?** Measure error against wind magnitude. Tosser's error should
*grow* with wind (it ignores it); Spoiler's should stay flat. This is the discriminating test — a
Spoiler that merely has a smaller error constant would fail it.

**Does Shooter really need line of sight?** On a map with a wall blocking the lane, Shooter must be
*helpless* — a test whose pass condition is failure to hit. If it still lands shots, it's cheating.

**Does Chooser choose?** Instrument which method it picked per shot; on blocked lanes it must select
`hasLineOfFire` and decline, not waste the shot.

**Does Unknown stay unknown?** The choice must be hidden in the UI and present in the save. Both
halves are required or the resumed match diverges.

**Determinism, throughout.** Every opponent: identical shot sequence across repeat runs of the same
seed; the standard trajectory-hash comparison. Plus the existing `selfCheck` must accept the new
fields and reject a save whose `opponent` doesn't match the running one.

**Burial.** Deposit dirt on a tank → its shots hit its own surroundings; Riot Charge clears it; the
deposit appears in `carves` in order; a resumed match reproduces the same mask hash.

**Tracers.** No damage dealt; path recorded; *and the bracket should use it* — a ranging shot that
doesn't inform the next shot would be a wasted feature.

---

## 7. Files likely to change

- `spikes/001-touch-controls/index.html` — all of it. Competences, the opponent table, `deposit()`,
  the BURIED HUD state, Tracers, the armory IA, save schema v9 and its `SETTLED` list.
- `spikes/001-touch-controls/ladder.html` — *new*. A harness for the 100-seed per-opponent sweep,
  following the existing `harness.html` / `armory.html` pattern, so the measurement can be run on a
  phone as well as the desktop.
- `SPEC.md` — state-machine notes for BURIED, and a pointer to this doc.
- `spikes/001-touch-controls/README.md` — the verification record for whatever gets built.
- `.github/workflows/pages.yml` — only if the new harness should ship to the live site.

---

## 8. Open questions

1. **Cyborg in a duel** — is "finishes the weakened" the right adaptation of "chooses targets"?
2. **Buried tanks and damage** — confirm dirt deposit does no damage (§3.5).
3. **Can a buried tank fire?** Proposed: yes, and it hits its own surroundings. Needs deciding.
4. **Arms Level** — faithful setup gate (a) or win-gated unlocks (b)? (§4.3)
5. **Do opponents differ in skill only, or also in wealth?** Proposed: **skill only**. A poor Moron
   confuses two axes and makes the ladder harder to read. (The AI already has its own bankroll, so
   this is a real choice.)
6. **Full catalog price fitting** — needs the rank-order/compression rule applied item by item.
7. **Poolshark** — recorded as unbuildable without rebounding walls. Revisit only if walls return.
