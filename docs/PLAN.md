# Dead Files — Technical Architecture & Implementation Plan

> A gritty 1990s UK investigative simulation and tactical point-and-click adventure.
> Engine: **Godot 4.x** (pin the latest stable 4.x at project start; nothing here needs anything newer than 4.4).
> Language: **typed GDScript** (rationale in §0.6).
> Target: PC (Steam, GOG), 640×360 internal canvas, integer-scaled.

---

## Contents

0. [Critical Feasibility Audit (Godot 4.x)](#0-critical-feasibility-audit-godot-4x)
1. [Domain Model & Entity Architecture](#1-domain-model--entity-architecture)
2. [State Machine & Viewport Flow](#2-state-machine--viewport-flow)
3. [Core Simulation Subsystems](#3-core-simulation-subsystems)
4. [Modular Implementation Plan (Milestones)](#4-modular-implementation-plan-milestones)
5. [Risk Register](#5-risk-register)
6. [Open Design Questions](#6-open-design-questions)

---

## 0. Critical Feasibility Audit (Godot 4.x)

**Headline verdict:** the *engine* side of this design is very achievable. Static first-person
scenes at 640×360 cost almost nothing to render, and Godot's `Resource` system fits a
simulation built on data. The real dangers are **scope** (eight distinct interfaces, each a
mini-game), **legibility** (a game about reading documents at 640×360), and the **Truth
Engine** (procedural mysteries that are solvable *and* interesting). The plan below is
ordered to deal with those three first.

### 0.1 Resolution & Viewport Pipeline

**What works out of the box**

| Setting | Value | Why |
|---|---|---|
| `display/window/size/viewport_width/height` | `640 × 360` | Base canvas. 16:9 and an exact divisor of 1280×720, 1920×1080 (×3), 2560×1440 (×4) and 3840×2160 (×6). |
| `display/window/stretch/mode` | `viewport` | Renders the whole game at 640×360, then scales up. No sub-pixel bleed by construction. |
| `display/window/stretch/aspect` | `keep` | Letterbox/pillarbox for 16:10 and ultrawide. |
| `display/window/stretch/scale_mode` | `integer` | Crisp integer scaling (available since 4.2). |
| `rendering/textures/canvas_textures/default_texture_filter` | `Nearest` | No bilinear smearing. |
| `rendering/2d/snap/snap_2d_transforms_to_pixel` | `true` | Stops sprites shimmering between pixels during tweens. |
| `rendering/2d/snap/snap_2d_vertices_to_pixel` | `true` | Same, for polygons and lines. |
| Renderer | **Compatibility** (OpenGL 3.3 / GLES3) | Pure 2D, so Forward+ gains nothing. Compatibility runs on more hardware (old laptops, Steam Deck) and starts faster. |

**Critique 1: legibility is the biggest technical-design risk.**
Most of this game is *reading*: chits, autopsy reports, microfiche newspapers, ledgers,
timecards, typed letters. At 640×360, a full A4 page cannot be shown readably. A 5×7 pixel
font gives about 100 characters per line at most, and microfiche newsprint is worse.
Options:

1. *Pure low-res:* documents become "zoomed fragments" you pan across. Authentic, but slow and
   hard on the eyes. **Not recommended** as the only mode.
2. **Hybrid pipeline (recommended):** draw the *world* (desk, rooms, car interior, corkboard
   frame) in a 640×360 `SubViewport`, integer-scaled. Draw **document inspection, the
   transcript and the UI text layer** in a native-resolution `CanvasLayer` above it, with a
   pixel-styled but higher-density font (e.g. a 2× bitmap font drawn at the output scale). The
   document "paper" art stays pixel art, but text is drawn sharp at native scale. *Papers,
   Please* and *Return of the Obra Dinn* both made similar compromises.
3. Add a **legibility setting** ("Authentic / Clear") that switches the document font and turns
   off grain over text. This is also an accessibility requirement (§4, M4).

Because of option 2, the root window uses `stretch/mode = disabled`, and a small
`PixelViewport` script does the integer scaling itself:

```gdscript
# res://core/display/pixel_viewport.gd
class_name PixelViewport extends Control
## Hosts the 640x360 world SubViewport and integer-scales it into the window.

const BASE := Vector2i(640, 360)
@onready var _container: SubViewportContainer = %WorldContainer
@onready var _viewport: SubViewport = %WorldViewport

func _ready() -> void:
	_viewport.size = BASE
	_viewport.canvas_item_default_texture_filter = Viewport.DEFAULT_CANVAS_ITEM_TEXTURE_FILTER_NEAREST
	get_window().size_changed.connect(_relayout)
	_relayout()

func _relayout() -> void:
	var win := get_window().size
	var scale := maxi(1, mini(win.x / BASE.x, win.y / BASE.y))
	_container.stretch_shrink = 1
	_container.scale = Vector2(scale, scale)
	_container.position = (Vector2(win) - Vector2(BASE * scale)) / 2.0
	Events.display_scale_changed.emit(scale)  # UI layer uses this to map world -> screen coords
```

Mouse input reaches the SubViewport through `SubViewportContainer`. Hotspot coordinates
are always authored in 640×360 space.

**Critique 2: shader stacking.** Film grain, CRT/phosphor glow, rain distortion and the
darkroom safelight tint each read the screen texture. Several `BackBufferCopy` passes or
several screen-reading `ColorRect`s cost more than they need to, and they mix badly.
**Use one post-process "uber shader"** on the `SubViewportContainer` (it already samples the
viewport texture, so you don't need `hint_screen_texture`). Effects are switched by uniforms:

```glsl
// res://core/display/post_uber.gdshader  (canvas_item)
shader_type canvas_item;
uniform float grain_amount : hint_range(0.0, 0.2) = 0.04;
uniform float rain_amount  : hint_range(0.0, 1.0) = 0.0;   // windscreen distortion
uniform sampler2D rain_normal : filter_nearest, repeat_enable;
uniform float phosphor_amount : hint_range(0.0, 1.0) = 0.0; // microfiche / terminal glow
uniform vec4 safelight_tint = vec4(1.0);                    // darkroom
uniform float time_quantum = 12.0;                          // grain fps, keeps it "filmic"
// ... single pass: offset UV by rain normal, sample, add bloom-lite, multiply tint, add grain.
```

Decide early whether effects run **at 640×360** (chunky grain, honest to the pixel art) or
**at output resolution** (smooth CRT scanlines). Recommendation: grain and rain at
640×360 (inside the SubViewport, as a final `CanvasLayer` `ColorRect`). The CRT/phosphor
effect, used only for the microfiche reader and terminal, runs at output resolution on the
container, so scanlines can be finer than a game pixel.

**Performance:** a static scene with about 20 layered sprites, one or two particle systems
(rain), and a single post pass is far below any budget, even on integrated GPUs. The only
real risk is **memory from large hand-painted backgrounds**. A 640×360 RGBA layer is about
0.9 MB, so 30 layers per room × 40 room templates is still fine. Use lossless import with
`VRAM Compressed` **off** (compression artifacts ruin pixel art).

**Pitfalls to codify in the style guide:**
- Never put a `Camera2D` with smoothing in the world viewport. Scenes are static; parallax
  "breathing" is done by moving whole layers in integer steps.
- Tweens on world-space positions must round (`position = position.round()`), or rely on
  pixel snapping.
- Rain particles: `GPUParticles2D` is fine, but use the Compatibility-friendly
  `CPUParticles2D` if you target very old GPUs.
- Text inside the low-res world (e.g. a sign on a wall) must use a **bitmap font** with
  antialiasing disabled and `Font.subpixel_positioning = DISABLED`.

### 0.2 State Machine & Node Architecture vs. Resource Bloat

**Recommendation: a hybrid.** That means **one persistent orchestrator (autoloads) plus swapped
"interface scenes" plus all simulation state in plain `Resource`/`RefCounted` data.**

- **Do not** keep every interface loaded in one giant scene under a state machine that toggles
  `visible`. The tape deck, corkboard, archive desk and burglary rooms each carry their own art,
  audio buses and input handling. Keeping them all alive wastes memory, leaks input
  (an invisible `Control` still eats clicks unless you set `mouse_filter`), and makes the
  scene tree very hard to debug.
- **Do not** go to the other extreme of a separate top-level scene per mini-step, swapped with
  `change_scene_to_file()`. That throws away the autoload/orchestrator layering and makes
  shared overlays awkward: the boss key, the clock HUD, the evidence satchel.

**Shape:**

```
Root (Main.tscn, never freed)
├── PixelViewport               ← 640x360 world SubViewport (§0.1)
│   └── WorldViewport
│       └── ActiveInterface     ← exactly ONE interface scene instanced here (swapped)
├── HiResLayer (CanvasLayer)    ← document reader, transcripts, tooltips, dialogue
├── HUDLayer (CanvasLayer)      ← clock, film count, op timer, scrutiny pip
├── OverlayLayer (CanvasLayer)  ← pause, boss-key cover screen, panic prompt
└── TransitionLayer             ← fades, "17:30 — The Flat" title cards

Autoloads (singletons):
  Events          ← typed signal bus (no state)
  Campaign        ← owns CampaignState (the single save root)
  GameClock       ← in-game time, phase windows
  PhaseDirector   ← top-level FSM (4 phases + sub-states)
  InterfaceRouter ← instances/frees interface scenes into ActiveInterface
  Content         ← read-only registry of authored defs (by StringName id)
  Audio           ← buses, directional cue player, ambience beds
```

**Rule:** *interface scenes are views.* They read from `Campaign.state`, emit intents on the
`Events` bus ("player wants to pick lock on socket X"), and redraw when state changes. They
never hold authoritative data. You can then free and re-instance any scene at will, and
save/load becomes "serialize `CampaignState`, rebuild the current interface."

**Resource bloat warning.** What does *not* belong in the scene tree:
- `EvidenceItem`s (hundreds over a campaign). Keep them as data; draw them as pooled
  `EvidenceCard` controls only when visible.
- Cold case boxes, microfiche rolls, chits, archive shelves. These are data. The archive is
  a list view and a shelf view, not 400 `Node2D`s.
- The burglary apartment graph. Rooms are `SceneNodeDef` data. Each room's **art** is a
  `PackedScene` *template* (e.g. `kitchen_galley_a.tscn`) that exposes named `Marker2D`
  anchors. Sockets are attached to those anchors at runtime.

Also: **separate definitions from runtime state.** A `Resource` loaded from `res://` is
*shared and cached*, so changing it at runtime changes it for everyone, and in the editor it
can even be saved back to disk. Authored data goes in `*Def` resources (immutable at
runtime). Mutable data goes in `*State` objects that refer to defs **by id**.

### 0.3 Procedural Generation & Truth Engine Feasibility

The pipeline in the brief (Seed Crime → Historical Distribution → Modern Target → Domestic
Sockets) is the correct *order of generation*. The real risk is not order. It is
**dependency cycles and unreachable clues**. Typical dead ends:

| Dead end | Example | Prevention |
|---|---|---|
| Circular gating | The only clue to the target's address is inside the target's house. | Clue placement respects a **knowledge DAG**: a clue may only be placed at a location whose access requirements are satisfiable by knowledge acquired *strictly earlier*. |
| Clearance lock-out | Motive document is in Vault C (Trust 60) but the campaign can only reach Trust 40 by then. | Generator is given the player's *projected* clearance curve. Validator checks `required_clearance ≤ projected_clearance(day_case_unlocks)`. |
| Time-window impossibility | Alibi timecard is in a safe (+5 min) behind a picked door (+3 min) in a 12-minute window. | Burglary solvability check: *minimum* action-cost path to all mandatory sockets ≤ `window × 0.6` (slack factor, tunable). |
| Single point of failure | The one motive clue is in a box the player misfiled last week, or burned film. | **Redundancy rule:** every anchor has ≥ 2 independent clue paths, at least one of them outside the burglary. |
| Red herring too good | A decoy satisfies all four anchors for an innocent person. | Decoys are placed so they can satisfy **at most 2 anchors** for any one wrong suspect. The validator checks this. |
| Unreadable logic | The generated chain needs a leap no human would make. | Clues come from **authored clue templates** with explicit `requires`/`yields` facts. There is no free-form inference. |

**How to make it feasible:**
1. **Hand-author first.** Build case #1 by hand *in the exact data format the generator will
   emit*. The generator becomes "a machine that writes what we already wrote by hand." This is
   the most important de-risking step in the whole project.
2. **Procedural assembly, not procedural invention.** The generator picks and parameterizes
   authored fragments: crime archetypes, motive archetypes, means archetypes, clue templates,
   room templates, document templates with fill-in slots. Writing quality stays human.
3. **Generate, then prove.** After generation, a **headless solver** (§3.1.4) simulates an
   idealized player over the knowledge DAG and proves all four anchors are reachable. Reject
   and re-roll on failure (rejection sampling, with a max-attempts cap). Fall back to a
   hand-authored case if the cap is hit.
4. **Determinism.** One `RandomNumberGenerator` per generation pass, seeded from
   `hash([campaign_seed, case_index, generator_version])`. **Never** use the global `randi()`,
   `Array.shuffle()` or `Array.pick_random()`. Those use the global RNG and break
   reproducibility. Provide `Rng.shuffle(arr, rng)` and `Rng.pick(arr, rng)` helpers.
5. **Lifecycle placement.** Generation runs on `WorkerThreadPool` between campaign days
   (behind the "Morning Audit → next day" transition) or when the campaign starts, behind a
   loading card. It produces *plain data* (dictionaries / `RefCounted`) and only builds
   `Resource`s on the main thread. Resources are not safe to share across threads while
   they are being mutated.
6. **Fuzz in CI.** A headless Godot run generates 10,000 seeds and asserts 100% solvability,
   zero cycles, and the decoy ceiling. This is cheap and catches regressions that playtesters
   never would.

### 0.4 Save/Load State Complexity

**Recommendations:**

1. **Do NOT save player progress as `.tres`/`.res` through `ResourceSaver`.** It is
   convenient, but loading a resource file can instantiate embedded scripts. A shared or
   downloaded save would then be a code-execution vector, and Steam Workshop or forum save
   sharing will happen. It also couples save files to class/script paths, which change during
   development.
   → Save to **versioned JSON** (readable, diffable, easy to migrate), optionally gzip'd. If
   you prefer binary, use `FileAccess.store_var(value, false)` (`full_objects = false`).
2. **Everything is referenced by stable `StringName` ids**, never by object reference or
   `NodePath`. `EvidenceState.location = {kind: "socket", id: &"case03.flat.bedroom.wardrobe"}`.
   On load, a `Registry` resolves ids to defs. A missing id is a hard, logged error, not a
   silent null.
3. **Store generated content, not just the seed.** Saving `seed + generator_version` is
   tempting, but any generator patch would silently rewrite a player's in-progress case. Save
   the **full generated `CaseSeed` output** (it is small, a few hundred KB at most) *and* the
   seed (for bug reports).
4. **Forensic & tampering data is an append-only event ledger** (event sourcing). Store the
   list of `ForensicEvent` and `CustodyEntry` records. Derived numbers (Task Force progress,
   heat) are **recomputed** on load from the ledger plus rules. That gives no desync, free
   debugging ("why am I at 70% heat?" → replay the ledger), and simple migrations.
5. **Save points at phase boundaries only** (start of Archive Day, start of Off-Shift, start of
   Night Op, start of Morning Audit). There are no mid-burglary saves, which removes a whole
   class of bugs (in-flight timers, mid-animation states, half-resolved complications). Add a
   single **"suspend" slot** written on quit during a burglary and **deleted when loaded**
   (roguelite-style) so players can quit without save-scumming.
6. **Schema versioning.** `{"schema": 7, ...}` at the root, plus a chain of
   `migrate_v6_to_v7(dict) -> dict` functions. Unit-test each migration against a frozen
   fixture save.
7. **Auto-backup** the previous 3 saves per slot. Procedural campaigns are long, and a single
   corrupt write is a refund request.

### 0.5 Systemic Pruning: Where the Fantasy Drowns in Admin

This is the candid part. The brief has **six-plus meters** (Department Trust, Internal
Scrutiny, Social Anchor, Police Heat, Task Force progress, forensic contamination) and **eight
interfaces** (archive desk, microfiche, darkroom, corkboard, tape deck, flat housekeeping,
stakeout, burglary), plus resolution puzzles, blotter, tampering and an interrogation.
Each one is fine alone. Together they risk *Papers, Please* plus *Thief* plus a tax return.

**Recommended cuts / merges for v1:**

| Item | Problem | Recommendation |
|---|---|---|
| **Trust + Scrutiny** | Two meters for one idea ("how much do they watch me?"). | Keep both but give them clear, different jobs: **Trust** is long-term and slow (clearance, audit odds). **Scrutiny** is short-term, resets daily, and drives in-day supervisor visits. Show Scrutiny as *diegetic* signs only: footsteps more often, the supervisor's mug left on your desk. No bar. |
| **Social Anchor** | A third life-sim meter. It adds chores, not detective work. | **Cut the meter.** Replace with a **fixed weekly calendar** of 1–2 obligations (Sunday dinner, Thursday pub). Missing one queues a scripted spare-key visit. Same tension, no meter, much less tuning. |
| **Police Heat + Task Force progress + forensic contamination** | Three overlapping "they're closing in" numbers. | **Merge into one "Task Force Profile"**: an accumulating set of *traits* the police know about the unknown offender (shoe size 9, left-handed tool marks, uses a 35mm camera flash, works with archive access…). Threat comes from *how well the profile matches you*, which is far more readable than a percentage. |
| **Chit processing** | Busywork if the volume is high. | **3–5 chits per day, each ≤ 60 s.** Chits do double duty: a legitimate chit is your *cover* to enter a vault that also holds the cold case box you want. That turns admin into tradecraft. |
| **Darkroom** | Three separate development tasks (crime-scene film, microfiche, own burglary film). | **One develop interaction** (load → time in bath → fix → hang), reused for all three. The darkroom's main value is the **alibi window** and the **solvent-smell cover**. Keep those, and make the minigame short. |
| **Cassette scrubbing** | 60/90-minute tapes in real time are tedious. | Compress: a "60-minute" tape is about 3 real minutes at 1×, with 8× fast-forward (pitched-up audio is a clue) and a **visual waveform** where talk shows as dense bursts. Scrubbing is about *spotting*, not waiting. |
| **Flat housekeeping** | Manually hiding each item before every op is a chore. | **One "secure flat" pass**, a checklist of 3–4 hotspots (cover board, lock trunk, burn notes) with a time cost. Skipping one is a *risk*, not a fail state. |
| **Interrogation mini-game** | A whole dialogue system for a rare event. | **Defer to M4/post-launch.** v1: a scripted interview with 3–4 branching choices that rely on Task Force Profile mismatches. |
| **Lethal strike methods** | Each method needs unique art, animation and rules. | **v1 ships 4 methods** (spiked bottle, medication swap, gas valve, brake line), each a 2–3 step socket interaction reusing the burglary system. More come later as content. |
| **Morning blotter** | Risk of a wall of text. | Show **at most 3 lines per op**, each tied to a concrete `ForensicEvent`. The blotter is *feedback*, not reading homework. |

**A deeper design critique.** The fantasy is *"I know the truth and I'm the only one who
will act on it."* Every system should feed **knowledge** or **risk**. Anything that only feeds
a number should be cut. Test each feature against: *"Does this make the player feel more like
a detective or more like a clerk?"* The answer for the clerk systems should be "a clerk who
is secretly a detective." In other words, the admin *is* the disguise, and it must stay
short enough to feel like one.

**On the "executioner" theme:** a wrong deduction plus a lethal strike means killing an
innocent person. This is the strongest moral lever in the design. Do not shy away from it
(see §3.2), but it puts real pressure on the Truth Engine's honesty. A player who acts on a
generator bug, rather than their own mistake, will feel cheated.

### 0.6 Language Choice: GDScript vs. C#

**Recommend typed GDScript.** The reasons: faster iteration, no .NET runtime to ship, no
export friction, and the best editor integration. Performance is a non-issue here (the
heaviest code is a graph search over maybe 200 nodes, run between days). Use static typing
everywhere (`untyped_declaration` warning set to *Error*) to get most of C#'s safety.
Consider C# **only** if you want existing .NET libraries or strong unit-testing culture. If
so, choose it from day one: mixing the two adds friction at every boundary.

---

## 1. Domain Model & Entity Architecture

### 1.1 Layering

```
┌──────────────────────────────────────────────────────────────────────┐
│ DEFINITIONS (authored or generated, immutable at runtime)  *Def    │
│   CaseSeed, PersonDef, FactDef, EvidenceDef, ClueTemplate,         │
│   DomicileDef, SceneNodeDef, SocketDef, LockDef, TargetRoutine,    │
│   ChitDef, ArchiveLocationDef, LethalMethodDef                     │
├──────────────────────────────────────────────────────────────────────┤
│ RUNTIME STATE (mutable, serialized)                        *State  │
│   CampaignState ─┬─ ArchiveDeskState                               │
│                  ├─ CaseState[]  ── EvidenceState[]                │
│                  ├─ DeductionBoardState                            │
│                  ├─ OperationState (only during Night Op)          │
│                  ├─ ForensicLedger (ForensicEvent[], CustodyEntry[])│
│                  └─ PlayerState (inventory, film, tools, traits)   │
├──────────────────────────────────────────────────────────────────────┤
│ SYSTEMS (stateless logic, pure functions where possible)           │
│   TruthEngine, SolvabilityValidator, AnchorValidator, OpClock,     │
│   ComplicationDirector, ForensicSystem, TamperingSystem, Scrutiny  │
├──────────────────────────────────────────────────────────────────────┤
│ VIEWS (scenes) ArchiveDesk, MicroficheReader, Darkroom, Flat,      │
│   DeductionBoard, TapeDeck, Stakeout, BurglaryRoom, Blotter, ...   │
└──────────────────────────────────────────────────────────────────────┘
```

### 1.2 Core Definitions (GDScript)

> All `*Def` classes extend `Resource` (so they can be authored in the inspector as `.tres`
> under `res://content/`). Generated cases are built from the same classes in code. Runtime
> `*State` classes extend `RefCounted` and implement `to_dict()` / `from_dict()`.

#### `CaseSeed`: the truth matrix

```gdscript
class_name CaseSeed extends Resource
## The complete, authoritative truth of one cold case. Hand-authored or Truth-Engine output.

@export var case_id: StringName
@export var generator_version: int = 0        # 0 = hand-authored
@export var rng_seed: int = 0
@export var crime_year: int                   # e.g. 1979
@export var crime_archetype: StringName       # &"domestic_poisoning", &"workplace_push", ...
@export var victim: PersonDef
@export var perpetrator: PersonDef            # historic identity
@export var perpetrator_modern: PersonDef     # alias / modern identity (may equal historic)
@export var suspects: Array[PersonDef] = []   # red-herring persons (incl. framed patsy)
@export var conflict: StringName              # underlying conflict archetype
@export var means: StringName                 # LethalMeansDef id
@export var motive: StringName                # MotiveDef id
@export var investigative_flaw: StringName    # why it went cold: &"lost_exhibit", &"bent_DI", ...
@export var facts: Array[FactDef] = []        # every true AND false fact in this case
@export var evidence: Array[EvidenceDef] = []
@export var domicile: DomicileDef             # modern target's home
@export var routine: TargetRoutine
@export var anchor_requirements: Dictionary   # AnchorType -> AnchorRequirement (see §3.2)
```

#### `FactDef`: atomic knowledge

```gdscript
class_name FactDef extends Resource
enum Anchor { NONE, IDENTITY, MEANS, MOTIVE, ALIBI }

@export var fact_id: StringName               # &"c03.f.alias_link"
@export var anchor: Anchor = Anchor.NONE
@export var subject_id: StringName            # PersonDef id this fact is about
@export var predicate: StringName             # &"changed_name_to", &"purchased", &"benefits_from"
@export var object_value: String              # "Derek Pryce", "strychnine", "£40,000 policy"
@export var is_true: bool = true              # false = planted lie / honest mistake in record
@export var links: Array[StringName] = []     # for chain facts (identity path edges)
```

#### `EvidenceDef` / `EvidenceState`: the thing you pin on the board

```gdscript
class_name EvidenceDef extends Resource
enum Kind { DOCUMENT, MICROFICHE_FRAME, PHOTO_NEGATIVE, PHOTO_PRINT, TAPE_SEGMENT, OBJECT, BLOTTER_ENTRY }
enum Origin { ARCHIVE_BOX, MICROFICHE, EVIDENCE_VAULT, SOCKET, TAPE, DERIVED }

@export var evidence_id: StringName
@export var kind: Kind
@export var origin: Origin
@export var origin_ref: StringName            # box id / roll+frame id / socket id / tape id
@export var title: String                     # "Prudential policy no. 44817"
@export var body_template: StringName         # document template id (rendered in HiResLayer)
@export var body_params: Dictionary = {}      # slot fills: {name, date, sum, ...}
@export var proves: Array[StringName] = []    # FactDef ids this evidence establishes
@export var art: Texture2D                    # pixel thumbnail / card art
@export var can_remove: bool = true           # false = must be photographed in situ
@export var access_requires: Array[StringName] = []  # fact ids or capability tags needed to reach it
```

```gdscript
class_name EvidenceState extends RefCounted
enum Location { UNDISCOVERED, IN_SITU, PLAYER_SATCHEL, FLAT_HIDDEN, FLAT_EXPOSED,
				ON_BOARD, UNDEVELOPED_FILM, ARCHIVE_SHELF, POLICE_CUSTODY, DESTROYED }

var evidence_id: StringName
var location: Location = Location.UNDISCOVERED
var location_ref: StringName                  # board slot / socket id / vault shelf
var discovered_day: int = -1
var is_copy: bool = false                     # a photograph print of a document, not the original
var tamper_flags: int = 0                     # bitmask: SWAPPED, MISFILED, CONTAMINATED

func to_dict() -> Dictionary: ...
static func from_dict(d: Dictionary) -> EvidenceState: ...
```

**Design note:** photographing a document in a burglary does **not** create the evidence.
It creates a `PHOTO_NEGATIVE` on the film roll, which must be **developed in the darkroom**
before it becomes a pinnable `PHOTO_PRINT`. This ties Phase 3 back to Phase 1 for free, and
makes film rolls a physical liability (undeveloped film in your satchel during a locker
search is bad).

#### `SceneNodeDef`, `SocketDef`, `LockDef`: the burglary graph

```gdscript
class_name DomicileDef extends Resource
@export var domicile_id: StringName
@export var address: String                   # "Flat 4, 22 Carrow Rd, Leyton"
@export var archetype: StringName             # &"council_flat_2bed", &"terrace_2up2down"
@export var nodes: Array[SceneNodeDef] = []
@export var entry_points: Array[StringName] = []  # node ids reachable from street
@export var neighbour_sensitivity: float = 0.3    # noise multiplier
```

```gdscript
class_name SceneNodeDef extends Resource
@export var node_id: StringName               # &"c03.dom.kitchen"
@export var display_name: String              # "Kitchen"
@export var art_template: PackedScene         # kitchen_galley_a.tscn (exposes Marker2D anchors)
@export var exits: Array[ExitDef] = []
@export var sockets: Array[SocketDef] = []
@export var lighting: StringName = &"night_streetlamp"
```

```gdscript
class_name ExitDef extends Resource
@export var to_node: StringName
@export var hotspot_anchor: StringName        # Marker2D/Area2D name inside art template
@export var label: String                     # "Hallway →"
@export var traverse_cost_s: int = 5
@export var noise: float = 0.0                # creaky stair = 0.2
@export var lock: LockDef                     # null = open doorway
```

```gdscript
class_name SocketDef extends Resource
enum Kind { CONTAINER, CONCEALMENT, HIDING, LETHAL }

@export var socket_id: StringName             # &"c03.dom.bedroom.bureau"
@export var kind: Kind
@export var anchor: StringName                # Marker2D in the room art template
@export var display_name: String              # "Antique bureau"
@export var search_cost_s: int = 30
@export var lock: LockDef                     # null = unlocked
@export var concealed_by: StringName          # another socket that must be moved first (painting → safe)
@export var contents: Array[StringName] = []  # EvidenceDef ids
@export var hide_quality: float = 0.0         # HIDING only: 0..1
@export var lethal_method: StringName         # LETHAL only: LethalMethodDef id
```

```gdscript
class_name LockDef extends Resource
enum Type { NONE, LATCH, CYLINDER, MORTICE, PADLOCK, WALL_SAFE, FILING_CABINET }
@export var type: Type
@export var pick_cost_s: int = 180            # silent
@export var force_cost_s: int = 15            # loud
@export var force_noise: float = 0.6
@export var force_trace: StringName = &"tool_mark"   # ForensicEvent kind left behind
@export var requires_tool: StringName = &"pick_set"  # &"stethoscope" for safes, &"pry_bar"
@export var combination_fact: StringName      # safe combo learnable from a clue → cost drops to 20 s
```

#### `TargetRoutine`

```gdscript
class_name TargetRoutine extends Resource
@export var target_id: StringName
@export var entries: Array[RoutineEntry] = []
@export var early_return_base_p: float = 0.08 # per-op base chance
@export var weather_sensitivity: float = 0.5  # rain makes them come home early

class_name RoutineEntry extends Resource       # (separate file in practice)
@export var weekday: int                      # 0 = Mon
@export var depart_min: int                   # minutes past midnight, e.g. 19*60+45
@export var return_min: int
@export var activity: String                  # "Social club, Leyton Rd"
@export var variance_min: int = 10            # ± jitter, revealed by repeated stakeouts
@export var observed: bool = false            # (lives in state in practice; shown here for clarity)
```

#### `ForensicEvent` / `ForensicLedger`: the footprint

```gdscript
class_name ForensicEvent extends RefCounted
enum Kind { TOOL_MARK, LATENT_PRINT, TREAD, DISTURBANCE, WITNESS_SIGHTING, FLASH_SEEN, FIBRE, SOLVENT_ODOUR }

var event_id: StringName
var op_id: StringName
var day: int
var kind: Kind
var node_id: StringName
var severity: float                           # 0..1, chance it is noticed + quality
var traits_revealed: Array[StringName] = []   # &"shoe_uk9", &"pry_bar_19mm", &"left_handed"
var discovered: bool = false                  # set by Morning Audit roll
var bag_id: StringName                        # evidence bag generated when discovered
```

```gdscript
class_name CustodyEntry extends RefCounted
enum Action { INTAKE, LOGGED, SWAPPED, MISFILED, AUDITED, RECOVERED }
var bag_id: StringName
var day: int
var action: Action
var clerk_id: StringName                      # player's id appears here when they touch it
var detail: Dictionary = {}
```

#### `ArchiveDesk` model

```gdscript
class_name ChitDef extends Resource
enum Task { PULL_FILE, ARCHIVE_ITEMS, MICROFICHE_LOOKUP, DEVELOP_FILM, INTAKE_BAG }
@export var chit_id: StringName
@export var requester: StringName             # DS Harlow, Sgt Okafor
@export var task: Task
@export var target_ref: StringName            # vault/box/roll
@export var docket_no: String                 # legit cover number for terminal queries
@export var time_budget_min: int = 45
@export var trust_reward: int = 2
```

```gdscript
class_name ArchiveDeskState extends RefCounted
var day: int
var chit_queue: Array[StringName] = []        # ChitDef ids (procedurally drawn per day)
var active_dockets: Array[String] = []        # dockets that legitimise queries right now
var trust: int = 20                           # long-term 0..100
var scrutiny: float = 0.0                     # in-day 0..1, resets each morning
var terminal_mode: StringName = &"council_tax" # boss-key cover screen
var locked_out_until_min: int = -1
var darkroom_occupied_until_min: int = -1
```

`ArchiveLocationDef` (vault → shelf → box) holds `clearance_required: int` and
`contents: Array[StringName]`. The archive is drawn as a **list/shelf view**, not as nodes per box.

#### `CampaignState`: the save root

```gdscript
class_name CampaignState extends RefCounted
const SCHEMA := 1
var campaign_seed: int
var day: int = 1
var phase: StringName = &"archive_day"
var clock_min: int = 8 * 60 + 30
var player: PlayerState
var desk: ArchiveDeskState
var cases: Dictionary = {}                    # case_id -> CaseState (incl. embedded generated CaseSeed dict)
var board: DeductionBoardState
var ledger: ForensicLedger
var calendar: Array[Dictionary] = []          # obligations
var flags: Dictionary = {}                    # narrative flags

func to_dict() -> Dictionary: ...
static func from_dict(d: Dictionary) -> CampaignState: ...  # runs migrations first
```

### 1.3 Project Layout

```
res://
├── core/                 # autoloads, display pipeline, save system, registry, rng helpers
│   ├── autoload/         events.gd, campaign.gd, game_clock.gd, phase_director.gd, interface_router.gd
│   ├── display/          pixel_viewport.gd, post_uber.gdshader
│   ├── save/             save_service.gd, migrations/
│   └── util/             rng.gd, ids.gd
├── domain/
│   ├── defs/             case_seed.gd, fact_def.gd, evidence_def.gd, socket_def.gd, ...
│   └── state/            campaign_state.gd, evidence_state.gd, operation_state.gd, ...
├── systems/              truth_engine/, solvability/, anchor_validator.gd, op_clock.gd,
│                         complication_director.gd, forensic_system.gd, tampering_system.gd
├── interfaces/           archive_desk/, microfiche/, darkroom/, flat/, deduction_board/,
│                         tape_deck/, stakeout/, burglary/, blotter/, evidence_intake/
├── content/
│   ├── cases/            case_001_hand.tres  (hand-authored, generator-format)
│   ├── templates/        clue_templates/, document_templates/, room_templates/
│   ├── archetypes/       crimes/, motives/, means/, lethal_methods/
│   └── chits/
├── art/ audio/ fonts/
└── tests/                # gdUnit4 or GUT; headless fuzz harness for the Truth Engine
```

---

## 2. State Machine & Viewport Flow

### 2.1 Signal Bus (`Events` autoload)

Signals are typed and grouped by domain. Views emit **intents**. Systems emit **facts**.

```gdscript
# res://core/autoload/events.gd
extends Node
# --- time & phase
signal clock_advanced(old_min: int, new_min: int)
signal phase_entered(phase: StringName)
signal phase_exiting(phase: StringName)
signal day_started(day: int)
# --- archive
signal chit_completed(chit_id: StringName)
signal scrutiny_changed(value: float)
signal supervisor_approaching(direction: float, eta_s: float)  # -1 left .. 1 right
signal boss_key_toggled(covered: bool)
# --- evidence & deduction
signal evidence_discovered(evidence_id: StringName)
signal evidence_moved(evidence_id: StringName, to: int, ref: StringName)
signal board_slot_changed(anchor: int)
signal board_verdict(verdict: Dictionary)
# --- operation
signal op_action_requested(action: StringName, target: StringName)
signal op_time_spent(seconds: int, remaining: int)
signal op_noise(amount: float)
signal complication_triggered(kind: StringName, data: Dictionary)
signal panic_prompt(deadline_actions: int, options: Array)
signal op_ended(outcome: StringName)
# --- audit
signal forensic_event_logged(event_id: StringName)
signal bag_intake(bag_id: StringName)
# --- display
signal display_scale_changed(scale: int)
```

**Rule of thumb:** use the bus for *cross-system* broadcast (HUD, audio, analytics).
Use direct method calls for *request/response* within one system (`OpClock.spend(...)`
returns the remaining time). Don't route everything through the bus. A signal spaghetti of
200 untyped signals is as bad as a god object.

### 2.2 `PhaseDirector`: a hierarchical FSM

Top level is the four phases. Each phase owns a small sub-FSM. States are `RefCounted`
classes with `enter/exit/handle(intent)`, not nodes, so they serialize trivially (only the
state *name* is saved).

```
PhaseDirector
├── ARCHIVE_DAY (08:30–17:00)          save point on enter
│   ├── AtDesk            (default; chits, terminal, boss key active)
│   ├── InVaults          (pull boxes; chit cover check)
│   ├── AtMicrofiche
│   ├── InDarkroom        (alibi window: supervisor patrols suspended for N minutes)
│   └── Covered           (boss-key overlay; transient)
├── OFF_SHIFT (17:30–21:00)            save point on enter
│   ├── Flat              (hub)
│   ├── DeductionBoard
│   ├── TapeDeck
│   ├── Staging           (rucksack loadout, pick op target + entry + night)
│   └── SecureFlat        (housekeeping checklist)
├── NIGHT_OP (21:30–03:00)             save point on enter (suspend-only inside)
│   ├── Stakeout          (dashboard; scrub clock; learn routine)
│   ├── Infiltration      (node-based burglary; OpClock live)
│   ├── Panic             (1-action response window)
│   ├── Resolution        (lethal socket OR stage break-in + leave)
│   └── Exfil / Caught
└── MORNING_AUDIT (03:30–08:30)        save point on enter
    ├── Handover          (phone box tip / typewriter leak, if legal path)
    ├── Blotter           (read overnight entries tied to your ForensicEvents)
    ├── EvidenceIntake    (log / swap / misfile bags that arrived)
    └── Interview         (only if Task Force Profile match ≥ threshold)
```

Not every night is an op night. `Staging` can choose **"stay in"**, which skips `NIGHT_OP`
(the stakeout can also be its own low-risk night). The week has a rhythm: research days,
stakeout nights, op nights.

**Transition skeleton:**

```gdscript
# res://core/autoload/phase_director.gd
extends Node
var _phase: PhaseState
var _phases := {
	&"archive_day": ArchiveDayPhase.new(),
	&"off_shift": OffShiftPhase.new(),
	&"night_op": NightOpPhase.new(),
	&"morning_audit": MorningAuditPhase.new(),
}

func goto(phase_id: StringName, sub: StringName = &"") -> void:
	if _phase:
		Events.phase_exiting.emit(_phase.id)
		_phase.exit()
	_phase = _phases[phase_id]
	Campaign.state.phase = phase_id
	if _phase.is_save_point:
		SaveService.autosave(Campaign.state)       # before enter: state is clean
	_phase.enter(sub)                               # phase asks InterfaceRouter for its scene
	Events.phase_entered.emit(phase_id)

func _unhandled_input(e: InputEvent) -> void:
	if _phase: _phase.handle_input(e)
```

### 2.3 `InterfaceRouter`: scene swapping

```gdscript
extends Node
const SCENES := {
	&"archive_desk": preload("res://interfaces/archive_desk/archive_desk.tscn"),
	&"deduction_board": preload("res://interfaces/deduction_board/deduction_board.tscn"),
	# heavy scenes (burglary rooms) are NOT preloaded; see show_room()
}

func show(id: StringName, params := {}) -> void:
	await Transition.fade_out()
	_free_active()
	var inst: Node = SCENES[id].instantiate()
	if inst.has_method(&"setup"): inst.setup(params)
	_active_slot.add_child(inst)
	await Transition.fade_in()

func show_room(node_def: SceneNodeDef) -> void:
	# Threaded preload of adjacent rooms keeps node hops instant.
	for exit in node_def.exits:
		ResourceLoader.load_threaded_request(Content.room_template_path(exit.to_node))
	...
```

**Overlays vs. swaps:** the document reader, evidence satchel, panic prompt and boss-key
cover are **overlays** on persistent CanvasLayers. They never swap the world scene. The
**corkboard and tape deck are swaps from within the Flat**, with a quick "lean in" transition
of about 150 ms, because they own their own input handling.

### 2.4 Time Model

Two kinds of time, both driven by `GameClock`:

| Phase | Time model | Why |
|---|---|---|
| Archive Day | **Compressed real time**: 1 real second = 1 game minute (≈ 8.5 real minutes per shift), paused in menus and the document reader. | Supervisor footsteps and the boss key need real-time tension. |
| Off-Shift | **Action-cost** (pinning is free; tape listening costs its playback time; staging and securing the flat cost fixed minutes). | Thinking should not be punished. Doing should cost time. |
| Night Op: Stakeout | **Scrubbable**: the player drags the dashboard clock; observed events are revealed. Costs the night's time up to the scrubbed point. | As specified. |
| Night Op: Burglary | **Discrete action-cost** (OpClock, §3.3). There is **no** ambient drain. | Clear, readable, fair. "1-turn" panic prompts only make sense in discrete time. |
| Morning Audit | Event-driven (no clock pressure). | It is a debrief. |

---

## 3. Core Simulation Subsystems

### 3.1 The Truth Engine (Procedural Cold-Case Generator)

#### 3.1.1 Knowledge DAG model

Everything the player can learn is a **knowledge token**: either a `FactDef` id or a
**capability** (`&"knows_address"`, `&"has_clearance_3"`, `&"knows_safe_combo"`,
`&"knows_routine_window"`). Every clue source has `requires` (tokens needed to *reach*
it) and `yields` (tokens it gives).

```
            ┌───────────────┐   yields fact: victim_name, crime_date, autopsy_poison
            │ Archive Box   │──────────────────────────────┐
            │ (clearance 1) │                              ▼
            └───────────────┘                  ┌───────────────────┐
                                               │ Microfiche: Gazette│ requires: crime_date
                                               │ 14 Mar 1979        │ yields: historic_name_photo
                                               └─────────┬─────────┘
                                                         ▼
                      ┌───────────────────────────────────────────────┐
                      │ Microfiche: Deed-poll notices 1981           │ requires: historic_name
                      │ yields: alias_link (IDENTITY chain edge)      │
                      └──────────────────────┬────────────────────────┘
                                             ▼
                      ┌───────────────────────────────────────────────┐
                      │ Terminal: Electoral roll (needs docket cover) │ requires: alias_name
                      │ yields: modern_address  → capability knows_address
                      └──────────────────────┬────────────────────────┘
                                             ▼
                      ┌───────────────────────────────────────────────┐
                      │ Stakeout (2 nights)  requires: knows_address  │
                      │ yields: knows_routine_window                  │
                      └──────────────────────┬────────────────────────┘
                                             ▼
                      ┌───────────────────────────────────────────────┐
                      │ Burglary socket: bureau  requires: window     │
                      │ yields: MEANS receipt photo (needs darkroom)  │
                      └───────────────────────────────────────────────┘
```

#### 3.1.2 Generation stages (forward over the brief's pipeline)

```
Stage 1  SEED CRIME MATRIX
  rng ← seeded(campaign_seed, case_index, GEN_VERSION)
  archetype ← weighted_pick(CrimeArchetypes, filter: era-plausible for crime_year)
  victim, perpetrator ← PersonFactory(archetype.roles, rng)
  suspects ← 2..3 PersonFactory decoys; one is the "patsy" with partial false evidence
  means  ← pick(archetype.allowed_means)      motive ← pick(archetype.allowed_motives)
  flaw   ← pick(InvestigativeFlaws)            # explains why it's cold AND which records are "wrong"
  perpetrator_modern ← AliasFactory(perpetrator, rule: 40% name change, 60% same name/moved)
  facts  ← FactFactory.emit_truth(archetype, means, motive, alias chain)
         + FactFactory.emit_lies(flaw, patsy)  # false alibi statements, mis-dated log, etc.

Stage 2  ANCHOR REQUIREMENTS
  for anchor in [IDENTITY, MEANS, MOTIVE, ALIBI]:
      req[anchor] ← AnchorRequirement from archetype rules
      # e.g. IDENTITY = path(historic_name → alias → modern_address)
      #      ALIBI    = any_of({timecard_falsified}, {witness_A_conflict + witness_B_conflict})

Stage 3  HISTORICAL RECORD DISTRIBUTION
  for each required fact f (and ~30% of lie facts):
      template ← pick ClueTemplate whose yields ∋ f and whose source ∈ {ARCHIVE, MICROFICHE, VAULT, TAPE}
      place at location L where clearance(L) ≤ projected_clearance(case_unlock_day)
      set template.requires from template rules (e.g. microfiche needs a date fact)
  enforce REDUNDANCY: each anchor gets ≥ 2 independent paths; ≥ 1 path outside the burglary

Stage 4  MODERN TARGET GENERATION
  domicile ← DomicileArchetype pick → instantiate room graph from room templates
  occupation, routine ← RoutineFactory(perpetrator_modern, rng)
  guarantee ≥ 1 weekly window ≥ MIN_WINDOW (e.g. 45 min) on an op-able night

Stage 5  DOMESTIC SOCKET PLACEMENT
  for each fact assigned to "modern" sources (typically MEANS receipt, MOTIVE correspondence):
      pick socket from domicile with compatible kind (letters → bureau/drawer; receipt → kitchen tin)
      optional: wrap in lock / concealment (difficulty budget per case tier)
  add HIDING sockets: ≥ 1 per room on the main path; ≥ 2 within 1 hop of every entry node
  add LETHAL sockets for allowed methods given domicile (gas cooker → gas valve; car → brake line)
  scatter mundane filler contents (non-evidence) for search texture

Stage 6  RENDER DOCUMENTS
  for each EvidenceDef: fill document template slots from facts (names, dates, sums)
  # lies render as plausible documents too: that's the point

Stage 7  VALIDATE (§3.1.4) → accept or re-roll (max 50 attempts → fallback authored case)
```

#### 3.1.3 Content scale (to keep generation interesting)

The variety comes from authored content, not from code. Rough v1 targets:

| Pool | v1 count | Notes |
|---|---|---|
| Crime archetypes | 8 | domestic poisoning, staged fall, workplace "accident", arson, hit-and-run, overdose, drowning, gangland |
| Motive archetypes | 10 | insurance, inheritance, affair, blackmail, debt, silence-a-witness, ... |
| Means archetypes | 10 | each with era-plausible paperwork (chemist register, gun cert, garage invoice) |
| Investigative flaws | 8 | bent DI, lost exhibit, coerced confession of patsy, misread pathology, ... |
| Clue templates | ~60 | the real workhorse; each = source + document template + requires/yields |
| Document templates | ~40 | autopsy, MG11 witness statement, timecard, policy, letter, deed-poll notice, gazette column, ... |
| Room templates | ~25 | across 5 domicile archetypes (council flat, terrace, semi, bedsit, flat above shop) |
| Lethal methods | 4 | §0.5 |

#### 3.1.4 Solvability validator

A **headless idealized player**: a fixed-point search over knowledge tokens.

```gdscript
class_name SolvabilityValidator
## Proves a CaseSeed is completable. Pure function; safe to run on a worker thread.

static func validate(case: Dictionary, ctx: ValidationContext) -> ValidationReport:
	var known := ctx.starting_tokens.duplicate()       # e.g. victim name from the case file label
	var sources: Array = case.clue_sources              # [{id, requires:[..], yields:[..], kind, cost}]
	var reached := {}
	var changed := true
	while changed:
		changed = false
		for s in sources:
			if reached.has(s.id): continue
			if not _all_known(s.requires, known): continue
			if s.kind == &"socket" and not _burglary_feasible(s, case, known, ctx): continue
			if s.kind == &"archive" and s.clearance > ctx.projected_clearance(case.unlock_day): continue
			reached[s.id] = true
			for t in s.yields: known[t] = true
			changed = true

	var report := ValidationReport.new()
	for anchor in FactDef.Anchor.values():
		if anchor == FactDef.Anchor.NONE: continue
		var req: AnchorRequirement = case.anchor_requirements[anchor]
		report.anchor_ok[anchor] = req.is_satisfied_by(known)
		report.anchor_paths[anchor] = _count_independent_paths(anchor, sources, ctx)  # ≥ 2 required
	report.decoy_ok = _decoy_ceiling(case, known) <= 2   # no wrong suspect satisfies > 2 anchors
	report.cycle_free = _is_dag(sources)
	return report
```

`_burglary_feasible` runs **Dijkstra over the domicile graph** from each entry point. Edge
weight = `traverse_cost_s + min(pick_cost_s, force_cost_s)` for locked exits. It sums
the costs to reach every *mandatory* socket in that op (plus the socket's search/lock cost),
plus the return path to an exit, and requires `total ≤ window_s × SLACK` (SLACK = 0.6 by
default, ~0.75 on "Hard"). It also asserts that every mandatory socket is ≤ 1 hop from a
HIDING socket.

**CI fuzz test:** `godot --headless -s tests/fuzz_truth_engine.gd -- --seeds 10000`.
It fails the build on any rejection-after-50-rerolls, cycle, or decoy ceiling breach, and
prints the seed for reproduction.

### 3.2 4-Anchor Deduction Matching (the "Roottree Code")

**Board model:** four anchor slots, each holding up to 3 pinned evidence cards, plus a
**subject** field (the person the board is "about").

```gdscript
class_name AnchorRequirement extends Resource
enum Mode { ALL_OF, ANY_OF_SETS, PATH }
@export var anchor: FactDef.Anchor
@export var mode: Mode
@export var fact_sets: Array = []      # ANY_OF_SETS: [[f1], [f2, f3]]; ALL_OF: [[f1, f2]]
@export var path_from: StringName      # PATH (identity): historic identity token
@export var path_to: StringName        #                   modern address token
```

**Validation runs in three tiers:**

| Tier | Check | Feedback to player | Purpose |
|---|---|---|---|
| **1. Category** | Does each pinned card prove ≥ 1 fact whose `anchor` matches the slot? | Immediate. The pin "won't stick" (a soft bounce with the line *"That's not a motive, that's a bus timetable"*). | Teaches the system without giving answers. |
| **2. Coherence** | Do all pinned facts share one subject chain? (IDENTITY establishes `historic ↔ modern` and every other anchor's facts are about a person on that chain.) For PATH mode, do the pinned identity facts form an unbroken chain? | Immediate. Red string goes taut and green when coherent. It visibly snaps when two cards point at different people. | Gives the satisfying "it connects!" moment. |
| **3. Truth** | Are all pinned facts `is_true` **and** do they satisfy `AnchorRequirement` for the *real* perpetrator? | **Hidden.** The board lets you *act* once tiers 1–2 pass on all four anchors. Truth only shows itself through consequences. | Real deduction stakes. A coherent but false board (the patsy frame) is possible. |

```gdscript
class_name AnchorValidator
static func evaluate(board: DeductionBoardState, case: CaseSeed) -> Dictionary:
	var v := {"category": {}, "coherent": false, "actionable": false, "is_true": false}
	var subject_chain := _identity_chain(board.slots[FactDef.Anchor.IDENTITY], case)  # set of person ids
	var all_cat := true
	for anchor in board.slots:
		var facts := _facts_of(board.slots[anchor], case)
		v.category[anchor] = facts.any(func(f): return f.anchor == anchor)
		all_cat = all_cat and v.category[anchor]
	v.coherent = all_cat and not subject_chain.is_empty() \
		and board.all_facts(case).all(func(f): return f.anchor == FactDef.Anchor.NONE or subject_chain.has(f.subject_id))
	v.actionable = v.coherent
	v.is_true = v.coherent and _all_true_and_satisfying(board, case)   # never shown in UI
	return v
```

**Consequences of a false board:**
- *Legal path:* the prosecution collapses, the patsy (or innocent) is charged, and the real
  killer is alerted. Their routine changes and they destroy remaining evidence. The case is
  harder but **still recoverable** (re-open with the remaining clue paths; this is why the
  redundancy rule exists).
- *Lethal path:* an innocent person dies. This is **not recoverable** for that case. It gives a
  permanent narrative scar and a large Task Force Profile jump (two "accidents" connected to
  one cold case is a pattern). This is the game's strongest moment. Signpost it with a
  difficulty option ("Show board truth confidence") for players who want a less brutal game.

### 3.3 Countdown Timer: `OpClock` & Burglary Rules

```gdscript
class_name OperationState extends RefCounted
var op_id: StringName
var domicile_id: StringName
var current_node: StringName
var window_start_s: int            # e.g. 19:45 → seconds since midnight
var window_end_s: int              # target's *actual* return time (hidden; includes variance + complications)
var believed_end_s: int            # what the player observed on stakeouts
var now_s: int
var noise: float = 0.0             # accumulates; decays 0.02 per 30 s of quiet
var film_left: int = 24
var flash_charge: float = 1.0
var opened_sockets: Dictionary = {}    # socket_id -> {forced: bool, disturbed: bool}
var forensic_events: Array[ForensicEvent] = []
var pending_return_s: int = -1     # set by ComplicationDirector
```

**Action cost table** (data-driven: `res://content/op_actions.tres`; values from the brief):

| Action | Cost | Noise | Trace | Notes |
|---|---|---|---|---|
| Move between nodes | `exit.traverse_cost_s` (5–15 s) | `exit.noise` | — | Creaky stairs 0.2 |
| Search container | 30 s | 0.05 | `DISTURBANCE` if not "tidy" | "Tidy search" option: ×2 time, no disturbance |
| Pick cylinder lock | 180 s | 0 | — | Silent |
| Force lock (pry-bar) | 15 s | 0.6 | `TOOL_MARK` (severity 0.8) | Reveals pry-bar width trait |
| Crack wall safe | 300 s | 0.1 | — | 20 s if `combination_fact` known |
| Photograph document | 15 s | 0.15 (wind) | `FLASH_SEEN` chance if room faces street | −1 film, flash recharge 8 s |
| Take item | 5 s | 0 | Item missing (police notice ⇒ blotter) | |
| Hide | 5 s | 0.05 | — | |
| Lethal setup | method-specific (60–240 s) | method-specific | method-specific | §3.5 |

**Per-action loop:**

```gdscript
func perform(action: OpAction) -> void:
	var cost := action.cost_s(op_state)
	op_state.now_s += cost
	op_state.noise = clampf(op_state.noise + action.noise, 0.0, 1.0)
	for trace in action.traces(op_state): ForensicSystem.record(op_state, trace)
	Events.op_time_spent.emit(cost, op_state.window_end_s - op_state.now_s)
	ComplicationDirector.tick(op_state, cost)       # may set pending_return_s, trigger panic
	_check_return()
```

**ComplicationDirector** (evaluated after each action, scaled by elapsed cost so long
actions are riskier):

```
p_complication(per action) = 1 - (1 - base_rate)^(cost_s / 60)
base_rate = routine.early_return_base_p / expected_actions
          + weather.rain_starting ? routine.weather_sensitivity * 0.05 : 0
          + noise * neighbour_sensitivity * 0.04           # neighbour knocks / calls police
Kinds: EARLY_RETURN, NEIGHBOUR_KNOCK, PHONE_RINGS (answerphone reveals info!), DOG, POWER_CUT
EARLY_RETURN sets pending_return_s = now + rand(60..180)
```

**Audio telegraphing → panic prompt:**
- At `pending_return_s - 60`: car door slams outside (stereo-panned to the street side).
- At `pending_return_s - 20`: keys rattle at the entry node.
- At `pending_return_s`: **Panic prompt.** The player gets **one action**: hide in a HIDING
  socket in the current node or 1 hop away, or exit through a non-entry exit (the fire escape).
  Anything else, or a timeout of 6 real seconds (configurable/off in accessibility), means
  **discovered**.
- **While hidden**, the target moves through the graph along a scripted "return path". The
  discovery chance per room visited is `disturbance_visible_in_room × (1 - hide_quality)`.
  After `rand(5..20 min)` (fast-forwarded), they go to sleep or leave again. The player then
  exfiltrates with a severe time-and-noise penalty.

**Running over time with no complication** (now > `window_end_s`) means the target simply
returns, which triggers the same keys-rattle → panic sequence. The stakeout's `variance_min`
data is why careful observation matters.

**Outcomes:** `CLEAN`, `TRACES_LEFT`, `SPOTTED` (witness description → profile traits),
`CAUGHT` (game-over or a scripted "flee" sequence with heavy profile gain; a design choice,
see §6).

### 3.4 Evidence Tampering Ledger & Task Force Profile

**Flow:**

```
Night Op ──► ForensicEvent[] (in OperationState) ──► committed to ForensicLedger at op end
                                   │
Morning Audit roll: each event discovered with p = severity × police_attention(case)
                                   │ discovered
                                   ▼
             Blotter line  +  EvidenceBag generated (arrives at archive intake in 1–3 days)
                                   │
                         Intake chit lands on YOUR desk
                                   │
          ┌────────────────────────┼─────────────────────────┐
          ▼                        ▼                         ▼
     LOG ACCURATELY           SWAP CONTENTS            ADMINISTRATIVE MISFILE
  traits → Profile         needs substitute item      bag → neglected vault
  (no personal risk)       (primer flakes, other      traits withheld; creates
                           shoe cast); traits         "missing exhibit" record
                           replaced with decoys       → future AUDIT risk
                           → AUDIT risk if checked
```

Every action writes a `CustodyEntry` with the player's `clerk_id`.

**Task Force Profile** is the merged "heat" system:

```gdscript
class_name TaskForceProfile
## Derived — recomputed from the ledger on load. Never saved directly.
static func compute(ledger: ForensicLedger, player: PlayerState) -> Dictionary:
	var known_traits := {}                 # trait -> confidence 0..1
	for bag in ledger.bags_logged_accurately():
		for t in bag.traits: known_traits[t] = maxf(known_traits.get(t, 0.0), bag.quality)
	for bag in ledger.bags_swapped():
		for t in bag.decoy_traits: known_traits[t] = maxf(known_traits.get(t, 0.0), bag.quality * 0.8)
	var score := 0.0
	for t in known_traits:
		if player.traits.has(t): score += known_traits[t] * TRAIT_WEIGHT[t]
	score += ledger.pattern_bonus()        # e.g. 2+ "accidents" linked to reopened cold cases
	score += ledger.custody_anomalies(player.clerk_id) * 0.15   # your name on misfiled/swapped bags
	return {"traits": known_traits, "match": clampf(score, 0.0, 1.0)}
```

- `match ≥ 0.4`: the Task Force asks the archive for records. Scrutiny baseline up and more
  supervisor walk-bys.
- `match ≥ 0.7`: **Interview** sub-state in Morning Audit.
- **Audits:** each day, `p_audit = 0.05 + 0.25 × (1 - trust/100) + 0.1 × open_anomalies`.
  An audit picks an archive section. Swapped or misfiled bags in that section may be found
  (`p = 0.5` each) and become strong `custody_anomalies`.

Players can also **change their own traits**: buy new shoes, switch the pry-bar for a
different width, wear gloves (a stops-prints tool slot). This turns the profile into an
active game rather than a passive doom meter.

### 3.5 Resolution Paths

- **Lethal (staged accident):** a `LETHAL` socket in the domicile (only offered once the
  board is actionable). It is a 2–3 step interaction (e.g. *gas valve*: open cooker cover →
  loosen valve with spanner → blow out pilot). Each step has a time cost and its own
  `ForensicEvent` risk (`FIBRE`, `TOOL_MARK`). The coroner's verdict arrives on the blotter
  days later as `accidental` or `suspicious`. The chance of `suspicious` depends on the traces
  left plus a method base rate.
- **Legal (parallel construction):** the player must (a) leave evidence where police will
  find it *legally*, by staging an "unforced" disturbance or leaving the bureau ajar, and
  (b) send a tip. Phone box: choose a box far from home (map pick), with a time cost. Each
  box's distance from the flat feeds a `WITNESS_SIGHTING` chance. Typewriter: choose a
  public machine (library, post office). The typeface itself is a trait. Outcome: prosecution
  starts, and **police attention on that case rises** (raising discovery rates on related
  events). With a configurable chance, a corrupt contact intervenes (a narrative event).

---

## 4. Modular Implementation Plan (Milestones)

**Guiding principle:** build **one hand-authored case end-to-end** before generating any.
Each milestone ends in something *playable* and a list of explicit exit criteria.

### M0: Project Bootstrap (≈ 1–2 weeks)

- [ ] Godot 4.x project, Compatibility renderer, settings from §0.1, `.gitattributes` (LFS for `*.png *.wav *.ogg`), `.gitignore` (`.godot/`, `*.import` per policy).
- [ ] Directory layout §1.3, autoload stubs (`Events`, `Campaign`, `GameClock`, `PhaseDirector`, `InterfaceRouter`, `Content`, `Audio`).
- [ ] `PixelViewport` + `HiResLayer` proof: a pixel-art desk background with a sharp
      document overlay, tested at 720p/1080p/1440p/4K and 16:10/21:9.
- [ ] Post-process uber shader with grain + rain toggles.
- [ ] Test framework (gdUnit4 or GUT) + headless CI job (GitHub Actions with a Godot headless image) that runs tests and exports a Windows/Linux build.
- [ ] Style guide: pixel snapping rules, font sizes, hotspot conventions, naming of ids (`c03.dom.kitchen.bin`).

**Exit criteria:** blank project builds in CI. The pixel and hi-res layers look correct on 4 resolutions.

### M1: Data / Resource Foundation (≈ 3–4 weeks)

- [ ] All `*Def` resources from §1.2 with inspector-friendly exports.
- [ ] All `*State` classes with `to_dict/from_dict` and round-trip unit tests.
- [ ] `Registry`/`Content` id lookup; duplicate-id and dangling-reference checker (editor tool + test).
- [ ] `SaveService`: JSON, schema version, migrations scaffold, 3-deep backups, suspend slot.
- [ ] `GameClock` (both time models), `PhaseDirector` FSM with stub phases and save points.
- [ ] `Rng` helpers (seeded shuffle/pick); lint rule or test that greps for `randi()`/`shuffle()` in `systems/`.
- [ ] Debug console overlay: `goto phase`, `give evidence`, `set trust`, `dump state`.
- [ ] **Hand-author Case 001** (`content/cases/case_001_hand.tres`) in full generator format: facts, evidence, anchor requirements, domicile (4–5 rooms), routine.

**Exit criteria:** you can cycle all four phases with placeholder screens, save and load at each
boundary, and get back an identical `to_dict()`. Case 001 passes the reference checker.

### M2: Archive Desk & Deduction Board (≈ 8–10 weeks)

*Phase 1:*
- [ ] Archive desk scene: chit tray, terminal (council-tax cover / records query), phone.
- [ ] Chit system: daily draw, vault pull flow, docket cover, Trust rewards.
- [ ] Vault/shelf list view with clearance gating; cold case box contents → document reader.
- [ ] Scrutiny (in-day) + supervisor patrol AI: a simple timeline of walk-bys, with frequency driven by scrutiny.
- [ ] **Directional audio cues** + boss key (cover screen swap ≤ 1 frame; caught-while-uncovered check at arrival).
- [ ] Microfiche reader: roll → frame scrub, phosphor shader, lens zoom, "note this" → evidence card.
- [ ] Darkroom: unified develop interaction; alibi window suspends patrols; negatives → prints.

*Phase 2:*
- [ ] Flat hub scene, secure-flat checklist, weekly obligations calendar.
- [ ] Deduction board: drag/pin cards, red string renderer, 3-tier `AnchorValidator`, "act on this board" gate.
- [ ] Tape deck: transport controls, tape counter, waveform strip, 8× FF pitched audio, mark segment → `TAPE_SEGMENT` evidence.
- [ ] Staging screen: rucksack slots (pick set, pry-bar, camera + film, gloves, torch), op night choice.

**Exit criteria:** Case 001 is fully *researchable*. A tester can find, pin and make coherent all
four anchors using only archive, microfiche and tape sources (redundancy path), within about 5 in-game days.

### M3: Node-Based Burglary Vertical Slice (≈ 8–10 weeks)

- [ ] Stakeout scene: windscreen rain shader, binocular view, scrubbable dashboard clock, observed routine log (with variance reveal on repeats).
- [ ] Room template pipeline: art scene with `Marker2D` anchors → `SceneNodeDef` binds sockets/exits at runtime; threaded preload of neighbours.
- [ ] `OpClock`, action table, noise model, film & flash, `OperationState`.
- [ ] Sockets: search, pick, force, safe crack, photograph, take; concealment chains (painting → safe).
- [ ] `ComplicationDirector`, audio telegraphing, panic prompt, hiding + return-path simulation, exfil.
- [ ] `ForensicSystem` → `ForensicLedger`; Morning Audit: blotter, intake bags, Log/Swap/Misfile, audits, `TaskForceProfile`.
- [ ] Resolution: **1 lethal method** (gas valve) + **legal path** (phone box tip); coroner/prosecution follow-ups.
- [ ] Scripted interview (3–4 choices), not the full minigame.

**Exit criteria (the vertical slice):** Case 001 is playable *start to finish*, through either
resolution, in about 2–3 hours. A false-board scenario is possible and handled. Save/load works at
every phase boundary, and the suspend slot works mid-op. **External playtest with 5–10 people
before M4.**

### M4: Procedural Truth Engine & Polish (≈ 12–16 weeks)

*Truth Engine:*
- [ ] Content pools at v1 scale (§3.1.3): authored archetypes, clue templates, document templates, room templates.
- [ ] Stages 1–6 generator; deterministic RNG; worker-thread execution behind day transition.
- [ ] `SolvabilityValidator` (knowledge fixed-point + Dijkstra burglary feasibility + decoy ceiling + DAG check).
- [ ] CI fuzz: 10k seeds, 100% pass; generation time budget < 500 ms per case on min-spec.
- [ ] Campaign structure: case unlock cadence, Trust/clearance curve projection used by the generator, an authored opening case and finale beats wrapped around procedural middle cases.
- [ ] Remaining 3 lethal methods; typewriter leak path.

*Polish:*
- [ ] Audio pass: ambience beds per location, foley for every tactile action (tape transport, film wind, flash whine, lock pins).
- [ ] Accessibility: Clear document font, timer relax / panic timeout off, safelight alternate palette (darkroom red is hard for colour-blind and low-vision players), subtitles for all audio cues **with direction indicators** (the boss key must not be audio-only), full key remapping, controller/Steam Deck pointer support.
- [ ] Interrogation minigame (if budget allows; otherwise post-launch).
- [ ] Steam (GodotSteam: achievements, cloud saves → JSON files make this trivial) & GOG builds; Steam Deck verification pass.
- [ ] Localization hooks (`tr()` on all UI; document templates keyed for translation, even if launch is English-only).

**Exit criteria:** a 10+ case campaign that generates reliably. Feature-complete for beta.

### Suggested sequencing / team-size note

For a solo developer or a team of 2–3, total **~10–13 months** to beta is realistic *with the
pruning in §0.5*. Without pruning, double it. The single biggest schedule risk is art: about
25 room templates plus the desk, flat, car, darkroom and microfiche. Lock the art style (palette,
dithering rules, resolution of detail) in M0 with one finished room before building the rest.

---

## 5. Risk Register

| # | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R1 | Generated cases feel samey / illogical | High | High | Hand-author first; procedural *assembly* of authored fragments; large clue-template pool; playtest generated cases specifically. |
| R2 | Document legibility at 640×360 | High | High | Hybrid hi-res text layer (§0.1); Clear-font option. |
| R3 | Admin systems feel like chores | Medium | High | §0.5 pruning; every chit doubles as cover; hard cap on per-day admin time. |
| R4 | Player acts on a wrong board because of a generator bug | Medium | Very high | Validator + fuzz CI; decoy ceiling; "report this case" debug export with seed + full state. |
| R5 | Save corruption / breaking saves between patches | Medium | High | JSON + schema migrations + fixture tests; store generated output, not just the seed; backups. |
| R6 | Art throughput | High | High | Modular room templates; reuse via palette/prop swaps; lock style in M0. |
| R7 | Real-time boss-key mechanic frustrates players | Medium | Medium | Generous telegraphing; difficulty option; visual + subtitle cues. |
| R8 | Lethal-path content and platform ratings | Low | Medium | Stylized, off-screen deaths; check Steam/GOG content survey early; PEGI/ESRB-style self-rating for store pages. |
| R9 | Godot minor-version upgrades mid-project | Medium | Low | Pin engine version per milestone; upgrade only at milestone boundaries with full test run. |

---

## 6. Open Design Questions

These are decisions for you, the designer. The architecture supports any of the answers, but
each one changes content and tuning:

1. **Is getting caught game-over** (ironman/roguelite feel), or a recoverable setback (flee
   sequence, big profile jump)? The plan assumes *recoverable* for v1, with an optional
   Ironman mode.
2. **Campaign length & structure:** one long run of N procedural cases with an authored
   arc around it, or a "case-of-the-week" endless mode? The plan assumes an authored
   frame (opening + finale) with procedural middle cases.
3. **How visible should board truth be?** The plan hides tier 3 by default, with an
   accessibility/difficulty toggle that shows a confidence hint.
4. **Is the lethal path available from the first case,** or unlocked after the player has
   done a legal resolution? Gating it is a strong narrative and onboarding lever.
5. **Setting specifics:** one fictional city (more freedom), or a real one (e.g. a fictionalized
   Greater Manchester/London borough)? This affects microfiche newspaper names, street maps and
   the authenticity of place names.
