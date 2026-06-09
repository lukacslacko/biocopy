# BioCopy

A web toy where biomolecules float in a soup and **assemble themselves by running
little programs**. Instead of behavior being hardcoded, every ball's *kind*, its
*bond slots*, and the *actions* it takes are specified in a small text DSL (see
[`copy.prog`](copy.prog)). The headline behavior is **templated reproduction**: a
`Polymerase` ball swims to a template chain, reads it monomer by monomer, grabs raw
material, and builds a copy — which is itself a valid template, so copies beget
copies. It's a fork of the ball-and-spring look & feel from
[`knot`](https://github.com/lukacslacko/knot), retuned for molecular self-assembly
(no gravity, no level-of-detail — balls never appear or vanish into nothing).

## The model

- **Balls.** Every ball has:
  - a **kind** — a name like `Blank`, `Cap`, `Info_1`, `Polymerase`. The kind
    determines the ball's color (a deterministic hash of the name) and, if the kind
    has a program, its behavior.
  - **named bond slots** — e.g. a `Polymerase` has `original`, `material`, `copy`;
    a chain monomer has `next` and `prev`. Bonds are **directed**: a slot points
    *from* this ball *to* another. A back-link is a separate bond in the other
    direction (that's why building a doubly linked chain sets both `next` and `prev`).
  - a **state** register (a name you can set and test), for programs that need a mode.
  - a **program** — a list of actions the ball runs top to bottom, looping via labels.
- **Molecules** are whatever the bond graph connects — no longer just chains.
- **Physics each frame** (a small relaxation sim, several substeps per frame):
  - **Springs** hold every bond at a rest length (`bond length × interact radius`).
  - **Collisions** push apart any two balls closer than `2 · interact radius`,
    *except* directly bonded pairs (this is the old "ignore K neighbors" with **K
    fixed at 1**).
  - **Jiggle** is an optional thermal kick that keeps the soup mixing (off by default).
  - **Soft boundary** instead of gravity: a ball past radius `R` feels a slight
    spring back toward the center. Everything just floats — there is no "down".
- **Swim-to-bind.** When a program binds a slot to a ball it isn't yet touching, the
  ball **swims** to it — a steady seek thrust folded into the same physics, so springs
  and collisions still apply, the soup can shove it off course, and anything it's
  already holding trails along behind it. On contact, the bond forms.

## The DSL

A program is a list of **ball definitions** and **scenarios**. Block comments
`/* … */` and line comments `//` are ignored. See [`copy.prog`](copy.prog) for a
complete, commented example (a `Polymerase`, a `Monomerase`, and two scenarios).

### Ball definitions

```c
ball Blank {};                              // a kind with no slots and no program

ball Info: Chain {                          // `Info` inherits Chain's slots/program
    next: Info | End;                       // a bond slot; the type list is advisory
    prev: Cap | Info;                       //   (mismatches are logged, not enforced)
};

ball Polymerase {
    original: Chain;
    material: Blank;
    copy: Chain;
    {                                       // the action block = this kind's program
        start:                              // a label
        original = Cap;                     // find a nearby Cap, swim to it, bind it
        material = Blank;
        material.kind = original.kind;       // change material's kind to original's
        copy = material;
        ...
        if (copy.kind != End) go clone;     // conditional jump
        ...
        go start;
    }
};
```

- **Inheritance** (`ball X: Y { … }`) merges Y's slots and program; X's own slots add
  to / override them, and X's own action block (if present) replaces Y's.
- A bond slot is `name: KindA | KindB | …;` (an optional `bond` keyword is accepted:
  `bond chain: End;`). The kind list is recorded for logging but not enforced.

### Actions

| Action | Meaning |
| --- | --- |
| `slot = Kind` | Find a nearby ball of `Kind`, swim to it, bind it into `slot`. |
| `slot = path` | Bind `slot` to a specific ball reached by a bond path (e.g. `original = original.next`). |
| `owner.slot = …` | Same, but set a slot on another ball (`copy.next = material`). |
| `slot = release` | Release whatever is bound in `slot`. |
| `target.kind = NewKind` / `= other.kind` | Change a ball's kind (a literal, or copied from another ball). |
| `target.state = NewState` / `= other.state` | Change a ball's state. |
| `label:` … `go label` | Define a jump target / jump to it. |
| `sleep 5s` / `sleep 200ms` | Pause this program. |
| `if (a == b) action` / `if (a != b) action` | Run `action` only if the comparison holds (compares `.kind` or `.state`). |

**Paths** start at one of *this ball's* slots and follow bonds: `original` is the
ball bound in the `original` slot, `original.next` follows that ball's `next` slot,
and a trailing `.kind`/`.state` reads (or, on the left, writes) that property.

### Scenarios

A scenario is an imperative builder, run once when you start it, with local variables:

```c
scenario Test {
    Blank * 1000;                 // create 1000 free Blanks
    Polymerase;                   // create one Polymerase
    chain = Cap;                  // create a Cap, remember it as `chain`
    chain.next = Info_1;          // create an Info_1 and bond chain.next to it
    chain = chain.next;           // walk the variable forward
    chain.next = Info_2;
    ...
    chain.next = End;
};
```

| Statement | Meaning |
| --- | --- |
| `Kind * N;` | Create `N` free balls of `Kind`. |
| `Kind;` | Create one. |
| `var = Kind;` | Create one and bind it to a variable. |
| `var.slot = Kind;` | Create one, bond `var.slot` to it. |
| `var = var.path;` | Move a variable along bonds. |
| `var.slot = otherVar;` | Bond a slot to an already-created ball (e.g. `chain.next.prev = chain` for a back-link). |

In a scenario a kind name **creates** a ball; in a program a kind name **finds** a
nearby one — that's the only semantic difference between the two.

> Because bonds are directed, a chain the `Monomerase` will walk *backwards* must be
> doubly linked. The `TestWithMonomerase` scenario sets each `prev` with
> `chain.next.prev = chain`, mirroring how the `Polymerase` sets both `copy.next` and
> `material.prev` on every monomer it copies.

## Controls

The panel loads `copy.prog`, lists its scenarios, and shows live stats:

| Control | Effect |
| --- | --- |
| **Scenario** | Pick a scenario from the loaded program; selecting one (re)spawns it. |
| **Run scenario** | Respawn the selected scenario. |
| **Reload program** | Re-fetch and re-parse `copy.prog` (edit the file, click Reload). |
| **Legend** | One row per kind with its color and a live ball count. |
| **Render radius** | Drawn ball size only — cosmetic. |
| **Interact radius** | Drives the sim: collision distance is `2 · interact`. |
| **Bond length** | Bonded-center rest length (× interact radius). |
| **Boundary radius / push** | Radius `R` of the soft containment sphere and how hard balls past it are nudged back. |
| **Spring / Collision / Damping / Jiggle** | Relaxation tuning. |
| **Sim substeps / frame** | Physics iterations per rendered frame. |
| **Pause / Shadows / Bonds / Boundary** | Freeze; toggle self-shadows; draw bond lines; show the boundary sphere. |

The **balls / molecules / bonds** counters (molecules = connected components of the
bond graph) sit in the bottom-left, next to a **GPU**/**CPU** indicator.

**Mouse:** drag empty space to orbit · scroll to zoom · right-drag to pan.

Swim speed, target-picking bias, and bind distance are currently code constants;
they'll move into the programs themselves later.

## Run it

A single self-contained file — no build step. Three.js loads from a CDN, and the page
fetches `copy.prog` at startup, so serve it over a static server (with an internet
connection on first load):

```sh
python3 -m http.server 8000
# then open http://localhost:8000/
```

Any static server works (`npx serve`, etc.). A WebGPU-capable browser is recommended
for large scenarios — the physics then runs on the GPU. (Opening the file directly
over `file://` also works: `fetch` is blocked there, so it falls back to an embedded
copy of the program.)

## Implementation notes

Everything lives in [`index.html`](index.html).

- **Parser.** A small tokenizer + recursive-descent parser compiles `copy.prog` into a
  kind table (names, merged-with-inheritance bond slots, and a per-kind instruction
  list) and a set of scenario builders. Bond-slot *names* are global, so changing a
  ball's kind doesn't disturb its bonds.
- **Two views of a bond.** The VM navigates **directed** slots (`bond[]`), while the
  physics uses an **undirected adjacency** (`nbr[]`). A physical bond a–b exists if
  *either* ball references the other, so a one-directional bond (enzyme→substrate)
  still springs symmetrically — and collisions skip directly-bonded pairs.
- **The VM** runs each ball whose kind has a program: it executes actions until it hits
  a blocking one (a swim-to-bind, or a sleep), nudging a `seekTarget` the physics reads
  as a thrust. Binding/releasing edit the directed slots and recompute the two balls'
  adjacency; changing a kind just recolors and (re)starts a program if the new kind has
  one.
- **Rendering** uses one `THREE.InstancedMesh` (a single draw call), colored per ball
  by `hash(kindName) → HSL`, lit by a key/fill/rim rig with ACES tone mapping.
- **GPU acceleration (WebGPU).** When available, the whole per-substep physics —
  springs along the adjacency, a uniform-grid collision broad phase (atomic binning)
  that skips bonded pairs, the seek thrust, jiggle, the soft boundary, and the
  integrate — runs in a compute shader. Positions are read back each frame to drive
  rendering and the VM; the VM (on the CPU) re-uploads only the changed adjacency and
  seek targets. If WebGPU is missing or fails it falls back to identical CPU physics;
  the stats line shows which path is running.
- A `window.__dbg` handle (positions, kinds, bonds, counts, a single-frame `tick()`)
  is exposed for automated testing; it has no effect on behavior.

## License

[MIT](LICENSE)
