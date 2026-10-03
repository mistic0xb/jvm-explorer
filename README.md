# JVM 25 Explorer
[![Live Demo](https://img.shields.io/badge/live%20demo-jvm--explorer.mistic.xyz-79c0ff?style=for-the-badge)](https://jvm-explorer.mistic.xyz/)

An interactive, single-file 3D visualization of how a modern JVM (Java 25, G1 collector) manages the heap: allocation, reachability, generations, evacuation, promotion, humongous objects, and mixed collections.

It is a **teaching model**, not a real JVM. Everything is simulated in ~700 lines of JavaScript + Three.js, running entirely in the browser. Open the file, click buttons, watch what happens.

---

## Table of contents

1. [Quick start](#quick-start)
2. [What you are looking at](#what-you-are-looking-at)
3. [The controls](#the-controls)
4. [Camera & navigation](#camera--navigation)
5. [The inspector & hover](#the-inspector--hover)
6. [How the model works (rules of the simulation)](#how-the-model-works-rules-of-the-simulation)
7. [Guided experiments](#guided-experiments)
8. [What is accurate to Java 25](#what-is-accurate-to-java-25)
9. [Where it differs from a real JVM](#where-it-differs-from-a-real-jvm)
10. [Tuning the model](#tuning-the-model)
11. [FAQ / troubleshooting](#faq--troubleshooting)
12. [File structure](#file-structure)

---

## Quick start

1. Save the explorer file as `index.html`.
2. Open it in any modern browser (Chrome, Firefox, Safari, Edge). No server, no build step.
3. Click **new Order graph**, then **Drop local**, then **Young GC**. Watch the log at the bottom.
4. Keep reading this README when you want to understand *why* something happened.

Only external dependency: Three.js r128 from a CDN. Offline? Download `three.min.js` and point the `<script src>` at your local copy.

---

## What you are looking at
### Heap regions (the 32 flat tiles)

The heap is divided into **32 equal-sized regions**, laid out in an 8 × 4 grid. This is the defining idea of G1: the heap is not split into three fixed contiguous spaces, it is a pool of equal regions that are *reassigned* roles as needed.

| Colour | Hex | Role | Meaning |
|---|---|---|---|
| dark gray | `#2d333b` | **free** | Unassigned. Can become eden, survivor, old or humongous. |
| green | `#3fb950` | **eden** | Where brand-new objects are allocated. |
| amber | `#d29922` | **survivor** | Young objects that survived at least one GC. |
| blue | `#388bfd` | **old** | Promoted (long-lived) objects. |
| red | `#f85149` | **humongous** | A single object ≥ half a region, living alone in its region. |

A region flashes white when its role changes — that is your cue that a GC just promoted, evacuated or reclaimed something.

Hover a region to see its ID, role and object count. Click it for a full breakdown in the inspector.

### Objects (the small cubes)

Each cube is a Java object. They drop into eden when allocated and slide to their slot.

| Object | Colour | Size | Notes |
|---|---|---|---|
| `Order` | salmon `#ff7b72` | 32 B | Root of the demo graph. |
| `Customer` | lavender `#d2a8ff` | 24 B | Referenced by `Order`. |
| `Item` | orange `#ffa657` | 24 B | Two of them referenced by `Order`. |
| `byte[]` | pale red `#ffb3ae` | humongous | Skips the young generation. |

A cube is **gray** (`#59616b`) when it is unreachable from any root — i.e. it is garbage that has not been collected yet. This distinction matters: *garbage* and *freed memory* are not the same thing in a JVM.

### References (the lines)

- **Dark-blue → bright-blue lines** are object-to-object references. The dark end is the source, the bright end is the target. They pulse gently to show they are being traversed.
- **Orange lines** go from a stack local (an orange sphere) to the object it points at — these are GC root edges.
- **Purple lines** (toggle: *Klass ptrs*) go from each object to its class box in Metaspace.

### Stack (the orange spheres)

Each orange sphere is one local variable slot on the Java thread stack. It is a **GC root**: anything reachable from here is alive. Up to 10 locals are modelled; if you allocate an 11th, the oldest falls out of scope and is logged.

Click the stack plate for a summary of the current locals.

### Metaspace (the purple plate)

Class metadata — `Order.class`, `Customer.class`, `Item.class`, `byte[].class`. This lives in **native memory, outside the Java heap**, so GC never moves it. Every object carries a *klass pointer* back to its class here; enable *Klass ptrs* to draw those edges.

### The log (bottom panel)

The most useful element on screen. It prints GC-style lines, colour-coded:

| Colour | Line type | Example |
|---|---|---|
| green | allocation | `new Order() -> #3 in eden region 5 (TLAB bump-pointer)` |
| yellow | GC pause | `Pause Young (Normal) (G1 Evacuation Pause) 4->2 regions, 7->3 objects` |
| blue | reference link | `Order#3 field -> Customer#4 (old->young: card table + remembered set)` |
| red | failure | `java.lang.OutOfMemoryError: Java heap space` |

Six lines are kept; older ones scroll off.

---

## The controls

### Allocate

| Button | What it does | What it teaches |
|---|---|---|
| **new Object** | Allocates one random object (Order / Customer / Item) into eden, held by a new local. | TLAB bump-pointer allocation; every object needs a root to stay alive. |
| **new Order graph** | Allocates an `Order`, a `Customer` and two `Item`s, with references Order→Customer and Order→Item, Item. Only the `Order` is held by a local. | Object graphs, transitive reachability, why one root can keep four objects alive. |
| **Humongous byte[]** | Allocates a `byte[]` that is ≥ half a region, straight into a dedicated humongous region. | Humongous allocation path. |
| **Link refs** | Adds one random reference between two reachable non-humongous objects (max 3 outgoing refs each). | Reference graph mutation; produces old→young edges that matter for GC. |
| **Drop local** | Removes a random local from the stack. | An object does not die when its local dies — it dies when *nothing* reaches it. |

### Garbage collector

| Button | What it does | What it teaches |
|---|---|---|
| **Young GC** | Evacuates eden + survivor. Live young objects are copied (age++), dead ones are left behind and their regions become free. | Stop-the-world copying collection; eden is emptied, not "swept". |
| **Mixed GC** | Logs a concurrent mark cycle, then collects young regions **plus** old regions that are under 85% live, plus dead humongous regions. | How old-gen memory is actually reclaimed by G1. |
| **Reset** | Empties the heap, clears locals, resets object IDs. | — |

### Navigate

| Button | What it does |
|---|---|
| **Overview** | Camera preset: whole scene. |
| **Inside heap** | Camera preset: close-up of the heap tiles. |
| **Metaspace** | Camera preset: the class-metadata plate. |
| **Stack** | Camera preset: the thread stack. |
| **Klass ptrs** | Toggle purple object→class lines. |
| **Region IDs** | Toggle `R0`…`R31` labels above each tile. |
| **Auto-orbit** | Slow automatic rotation of the camera. |
| **Zoom + / Zoom −** | Step the camera distance in/out. |

---

## Camera & navigation

| Input | Action |
|---|---|
| Left-drag | Orbit around the current target |
| Right-drag **or** Shift + drag | Pan the target |
| Mouse wheel / trackpad scroll | Zoom (2 … 75 units) |
| Click (without dragging) | Select an object / region / stack / Metaspace |
| Hover | Tooltip with a one-line summary |

Camera moves are smoothly interpolated, so clicking a preset glides rather than jumps.

---

## The inspector & hover

**Hover** shows a compact tooltip: name, region role, age, reference count (for objects) or ID, role and object count (for regions).

**Click** opens the right-hand Inspector panel with full detail:

- For an **object**: class, size, region + role, age vs. tenuring threshold, reachability, its reference list, and a note about object headers (12 B default, 8 B with `-XX:+UseCompactObjectHeaders`, JEP 519, which is a product feature in Java 25).
- For a **region**: role, object count / capacity, live count, contents, and a note explaining what that role means.
- For the **stack**: how many locals are live and what they point at.
- For **Metaspace**: location (native memory), loaded classes, and why klass pointers exist.

Selected objects glow with a pulsing emissive halo; selected regions pulse their tile.

---

## How the model works (rules of the simulation)

These are the exact rules the code implements. Knowing them makes the experiments predictable.

### Heap geometry

- 32 regions in an 8 × 4 grid.
- Each region holds at most **4** normal objects (`CAP = 4`), or exactly 1 humongous object.
- Region roles are dynamic. `setType()` changes a region's colour and flashes it.

### Allocation

1. Find an eden region with a free slot. If none, promote a free region to eden.
2. If no free region exists, run a Young GC and try again.
3. If still nothing, log `java.lang.OutOfMemoryError: Java heap space`.
4. The new object drops in from above (`position.y += 6`, then lerps down) — a visual metaphor for a bump-pointer allocation.

Locals are capped at 10; allocating past that drops the oldest local (FIFO) and logs it.

### Reachability

```
reachable = closure(roots)   // DFS over o.refs, skipping dead objects
```

Computed every `refresh()`. An object is drawn gray if it is not in the closure, and its references stop pulsing.

### Young GC

1. Find all eden + survivor regions (`from`).
2. Compute `youngLive()`: young objects reachable from roots, **plus** young objects reachable from any non-young object. This second part is the model's version of the **card table / remembered set** — an old object's references are treated as roots even if the old object itself is garbage.
3. For each object in `from`:
   - not live → `kill()` (float up, fade out, region slot freed)
   - live and in eden/survivor → `age++`, then:
     - `age >= TENURE` (3) → copy to an old region
     - otherwise → copy to a survivor region
4. Each `from` region: if it still holds objects it becomes `old`, otherwise it becomes `free`.
5. If a destination region could not be found, log `To-space exhausted: N objects pinned (evacuation failure)`.

Note the key detail: **dead objects are never individually freed.** The whole region is simply abandoned and its role reset. That is how a copying collector works.

### Mixed GC

1. Compute full reachability.
2. Candidate old regions: those with at least one object and a live fraction **< 85%**.
3. Humongous candidates: humongous regions whose single object is unreachable.
4. Log a fake concurrent-mark sequence, then collect `young ∪ candidates ∪ humongousCandidates` using the same `collect()` routine.

If no old region qualifies, the log tells you that mixed GC degenerates into a young GC — which is exactly what happens in a real JVM when every old region is mostly live.

### Aging and promotion

`TENURE = 3` in this model. Every successful evacuation of a young object increments its age. At age 3 it is promoted to old and stays there forever (until it becomes unreachable and some mixed GC picks its region).

HotSpot adapts the tenuring threshold dynamically, with a maximum of 15. This model fixes it at 3 so you can see promotion happen quickly.

---

## Guided experiments

Each experiment lists the clicks and what to look for.

### 1. Allocation and reachability

> **new Order graph** → click each cube.

You get `Order#1`, `Customer#2`, `Item#3`, `Item#4`. Only `Order#1` has an orange root line; the others are alive because `Order` references them. Click `Customer#2` — the inspector says *Reachable: yes*.

**Lesson:** liveness is transitive. One root keeps a whole graph alive.

### 2. Garbage ≠ freed memory

> **new Order graph** → **Drop local**

All four cubes turn gray. They are now unreachable, but they are still sitting in eden, still taking up space, still holding their slots. Nothing has been freed.

**Lesson:** garbage collection is lazy. An object becomes garbage the instant it is unreachable, but its memory is only reclaimed at the next GC that covers its region.

### 3. Aging and promotion

> **new Object** (a few times) → **Young GC** ×3

After the first Young GC, survivors move to an amber survivor region and their age becomes 1. Second GC: age 2, still survivor. Third GC: age 3 → promoted to a blue old region.

Click a promoted object: the inspector shows `Age 3 / 3`, region role `old`.

**Lesson:** objects that survive repeatedly are assumed to be long-lived and are moved out of the young generation so future young GCs can ignore them.

### 4. Old-to-young references (remembered set)

This is the subtle one, and the model gets it right.

> 1. **new Object** a few times, then **Young GC** until at least one object is promoted to old (blue region).
> 2. **new Object** to create a fresh young object.
> 3. **Link refs** repeatedly until the log prints `(old->young: card table + remembered set)`.
> 4. **Drop local** until the log says all locals are gone (or just drop a few).
> 5. **Young GC**.

The young object **survives** even though the old object pointing at it is itself garbage.

**Lesson:** a young GC does not scan the whole old generation. It uses the card table + remembered set to treat old-object references as roots. A dead old object can therefore keep a young object alive until a mixed GC removes both.

### 5. Humongous objects

> **Humongous byte[]** → click it → **Drop local** → **Young GC** → **Mixed GC**

The `byte[]` gets its own red region immediately (no eden stop). Dropping the local makes it garbage, but Young GC ignores it — humongous regions are not part of the young generation. Only the Mixed GC reclaims it, and the log reports it under *humongous reclaimed*.

**Lesson:** humongous objects are never copied (copying something ≥ half a region would be wasteful), so they take a different path: allocate → mark → reclaim.

### 6. Concurrent mark and sparse old regions

> Fill some old regions (repeat experiment 3), drop locals so a region is mostly garbage, then **Mixed GC**.

The log first prints the mark cycle (`initial mark, concurrent mark, remark, cleanup`), then collects the old regions below 85% live. If every old region is dense with live objects, the log tells you the mixed GC degrades into a young GC.

**Lesson:** G1 collects the *cheapest* old regions first. That is the "garbage-first" in G1.

### 7. Running out of heap

> Spam **new Object** and **Humongous byte[]** without collecting.

Eventually the log shows `java.lang.OutOfMemoryError: Java heap space` (no free region and Young GC could not free one). If you arrange a situation where live objects cannot be evacuated into enough space, you will see `To-space exhausted: N objects pinned (evacuation failure)`.

**Lesson:** GC is not magic. If live data does not fit, allocation fails.

---

## What is accurate to Java 25

- **G1 is the default collector** for server-class machines, and the heap is region-based — this is the entire premise of the scene.
- **Regions are dynamically assigned** to eden / survivor / old / humongous; there are no fixed contiguous generations.
- **Allocation is a pointer bump** in a thread-local allocation buffer (TLAB), which is why allocation is cheap and why the log says so.
- **Liveness = reachability from GC roots** (locals, statics, JNI handles, threads). The stack spheres model locals.
- **Young GC is a stop-the-world copying collection.** Dead objects are abandoned, live ones are copied, eden is emptied.
- **Aging on copy, then promotion** to old after a threshold.
- **Card table + remembered set** for old→young references; a dead old object can keep a young object alive.
- **Humongous objects** (≥ half a region) skip the young generation, are never copied, and are reclaimed only after marking.
- **Mixed GC** = concurrent mark, then young regions + old regions below a liveness threshold.
- **Metaspace is native memory**, off-heap, holding class metadata, reached via klass pointers.
- **Compact object headers** (JEP 519) are mentioned as a Java 25 product feature: 8 B headers instead of 12 B with `-XX:+UseCompactObjectHeaders`.

---

## Where it differs from a real JVM

Be honest about this when you use the model to teach.

| Model | Real HotSpot |
|---|---|
| 32 regions, each holding 4 objects | Thousands of regions, sized 1–32 MB depending on heap |
| Fixed region capacity | Variable per object |
| Tenuring threshold fixed at 3 | Adapted by the JVM each GC, max 15 (`-XX:MaxTenuringThreshold`) |
| Objects have uniform, tiny sizes | Real headers, alignment, compressed oops, etc. |
| Marking is only logged, not animated | Concurrent mark actually runs alongside the app |
| No SATB write barrier, no marking stack | Full SATB + concurrent marking machinery |
| No Full GC fallback | G1 falls back to a Full GC (serial, stop-the-world) on evacuation failure |
| No class unloading, no string dedup, no reference processing | All present in real G1 |
| References never create cycles that matter for GC | Cycle collection is the whole point of tracing GC |
| "Card table + remembered set" is a simplification | Real RSets are per-region hash structures with refinement |
| Objects are moved with a simple lerp | Evacuation is a real memory copy with forwarding pointers |

None of these gaps change the *shape* of the story. They change the numbers and the edge cases.

---

## Tuning the model

Everything lives at the top of the `<script>` block:

```js
const REG_C = 8, REG_R = 4,   // 8 × 4 = 32 regions
      SP = 3.4,               // spacing between region centres
      CAP = 4,                // max objects per non-humongous region
      TENURE = 3,             // promotions after this many survivals
      ZSTACK = 9.6;           // z position of the thread stack

const COL = { free:0x2d333b, eden:0x3fb950, surv:0xd29922,
              old:0x388bfd, humo:0xf85149 };

const CL = {
  Order:    { size:32, col:0xff7b72, z:-3 },
  Customer: { size:24, col:0xd2a8ff, z:-1 },
  Item:     { size:24, col:0xffa657, z: 1 },
  'byte[]': { size:0,  col:0xffb3ae, z: 3 }
};
```

Common tweaks:

| Goal | Change |
|---|---|
| Make promotion visible faster | `TENURE = 2` |
| Make regions hold more objects | `CAP = 6` (slots are laid out in a 2×2 grid, so >4 overlaps — increase `SP` too, or accept overlap) |
| Make the heap smaller / run out faster | `REG_C = 6, REG_R = 3` |
| Bigger heap | `REG_C = 10, REG_R = 6` |
| Different palette | edit `COL` and `CL[*].col` |
| Add a class | add an entry to `CL` — Metaspace box, label and `new Object` pool pick it up automatically |

Other useful constants inside the code:

- `mkLines(900, …)`, `mkLines(24, …)`, `mkLines(320, …)` — max reference / root / klass segments drawn per frame. Raise them if you build very large graphs.
- `logs.length > 6` — number of visible log lines.
- `autoOrbit` speed: `cur.th += dt * 0.13`.
- Camera presets in the `VIEWS` object.

---

## FAQ / troubleshooting

**The page is blank.**
Check the browser console. If Three.js failed to load from the CDN, download `three.min.js` (r128) and change the `<script src>` to a local path.

**Nothing happens when I click a button.**
The first thing to click is **new Order graph** — most other actions need objects to exist.

**The cubes overlap.**
Objects per region are placed in a fixed 2×2 pattern (`SL`), so the model assumes `CAP ≤ 4`. If you raise `CAP`, either raise `SP` or edit `SL`.

**"To-space exhausted" keeps appearing.**
You have too many live objects and not enough free regions to evacuate into. Run a **Mixed GC** to reclaim sparse old regions, or **Reset**.

**The reference lines disappear.**
They are dimmed, not removed, when an object is unreachable — that is intentional so you can see garbage. If lines genuinely vanish, you may have exceeded the `mkLines` maxima; raise the first argument.

**Log shows an OOM but objects still look alive.**
OOM is logged when allocation fails, but the existing heap is unchanged. Run a GC and look at the log to see what was actually reclaimed.

**Why does a young object survive even though its only local is gone?**
An old object still points at it. See experiment 4. This is the remembered-set behaviour, and it is a real source of "why isn't this garbage?" confusion in production.

**Why did mixed GC behave like a young GC?**
Because every old region was ≥ 85% live. Collecting them would not be worth it; G1 skips them. The log says so explicitly.

---

## File structure

```
index.html
├── <style>                 — all CSS (panels, buttons, log, tooltip, responsive)
├── <canvas id="c">         — the 3D viewport
├── #ui                     — left control panel
├── #info                   — right inspector
├── #log                    — bottom log
├── #tip                    — hover tooltip
└── <script>                — the whole simulation
    ├── constants           — geometry, colours, class table
    ├── renderer / scene    — Three.js setup, lights
    ├── makeLabel()         — crisp canvas-text sprites with outlines
    ├── regions             — the 32 tiles + role changes
    ├── stack / metaspace   — roots and class metadata
    ├── mkLines / seg       — batched line buffers
    ├── reach()             — reachability closure
    ├── allocation          — newObj, newOrder, newPlain, newHumo, link
    ├── GC                  — youngLive, collect, youngGC, mixedGC
    ├── UI                  — buttons, HUD, inspector, log
    ├── input               — orbit / pan / zoom / pick / hover
    └── frame()             — animation loop
```

~700 lines, no build step, no framework. Take it, fork it, change the constants, use it in a talk.

---

## Credits & licence
Uses [Three.js](https://threejs.org/) r128 (MIT).
The simulation logic is original; the JVM semantics it illustrates belong to the OpenJDK G1 collector.