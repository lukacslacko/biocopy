# BioCopy

A web toy that simulates biomolecules floating in a soup and **self-assembling**:
monomers jiggle around, bump into each other, and polymerize head-to-tail into
longer chains. It's a fork of the ball-and-spring look & feel from
[`knot`](https://github.com/lukacslacko/knot), retuned for molecular self-assembly
(no gravity, no level-of-detail — molecules don't appear or vanish into nothing).

## The model

- **Monomers.** Each monomer is a short chain of **5 balls**: `head – body – body –
  body – tail`. The head and tail are the *reactive ends*; the three body balls
  carry the monomer's identity. There are **four monomer types**, each with its own
  body color. All heads share one color, all tails share another — so the palette is
  one head color + one tail color + four body colors.
- **Soup.** "Fresh soup" scatters `N` random monomers (random type, random position
  and orientation) inside the boundary sphere, all initially unbound.
- **Physics each frame** (a small relaxation sim, several substeps per frame):
  - **Springs** hold every bond at a rest length (`bond length × interact radius`),
    so a monomer keeps its shape and bonded balls sit as a smooth bead-worm.
  - **Collisions** push apart any two balls closer than `2 · interact radius`,
    *except* the `K` nearest neighbors along the chain. `K` is set to **1** by
    default (only directly-bonded balls are exempt), which keeps chains extended
    instead of collapsing.
  - **Jiggle** is a thermal kick on every ball — this is what keeps the soup mixing
    so reactive ends keep finding each other. Turn it to 0 to freeze the dance.
  - **Soft boundary** instead of gravity: a ball that wanders past radius `R`
    (default 20) feels a very slight spring back toward the center. Everything just
    floats — there is no floor and no "down".
- **Polymerization.** Once per frame, every **unbound head** touching an **unbound
  tail** binds with a small, **tunable probability** (the "Bind probability / frame"
  slider). Binding joins a tail's outgoing bond to a head (`head ← tail`), extending
  the chain. A head only binds a tail from a *different* molecule, so chains stay
  open rather than closing into rings. Free ends glow brighter and bulge slightly,
  so you can always see which sites are still reactive.
- **Body affinity (chemistry).** A signed slider makes body balls of *different*
  monomers interact at short range: at a positive setting **like** bodies (same
  type/color) gently **attract** while **unlike** bodies **repel** when very close;
  a negative setting reverses it; zero is off. It's evaluated in the same short-range
  neighbor scan that drives collisions (no global all-pairs cost) and never acts
  within a single monomer (same-chain is fine) — so you can coax the soup into
  clusters and structure. This and the bind probability both live under the
  **Chemistry** panel.

The **molecules** counter (number of open chains) ticks down and the **bonds**
counter ticks up as the soup assembles.

## Controls

| Control | Effect |
| --- | --- |
| **Monomers (N)** | How many monomers a fresh soup contains (respawns). |
| **Bind probability / frame** | Chance a touching unbound head+tail pair binds, per frame. 0 = no polymerization. *(Chemistry panel.)* |
| **Body affinity (like ↔ unlike)** | Short-range force between bodies of different monomers: + = like-attract/unlike-repel, − = reversed, 0 = off. *(Chemistry panel.)* |
| **Fresh soup** | Scatter a new random soup. |
| **Render radius** | Drawn ball size only — cosmetic, no effect on the sim. |
| **Interact radius** | Drives the sim: collision distance is `2 · interact`. |
| **Bond length** | Bonded-center rest length (× interact radius); smaller = tighter worm. |
| **Boundary radius / push** | Radius `R` of the soft containment sphere and how hard balls past it are nudged back. |
| **Ignore K neighbors** | Chain neighbors exempt from collision (default 1 = directly bonded only). |
| **Spring / Collision / Damping / Jiggle** | Relaxation tuning. |
| **Sim substeps / frame** | Physics iterations per rendered frame. |
| **Pause / Shadows / Bonds / Boundary** | Freeze; toggle self-shadows; draw bond lines; show the boundary sphere. |

**Mouse:** drag empty space to orbit · scroll to zoom · right-drag to pan ·
**drag a ball** to move its molecule · **click a head or tail** to cut it free from
its neighbor. A click only cuts the *inter-monomer* bond at that end — a head or
tail is never severed from its own body, and no ball is ever deleted.

## Run it

A single self-contained file — no build step. Three.js loads from a CDN, so you just
need a static server (and an internet connection on first load):

```sh
python3 -m http.server 8000
# then open http://localhost:8000/
```

Any static server works (`npx serve`, etc.).

## Implementation notes

- **Rendering** uses one `THREE.InstancedMesh`, so all balls draw in a single call,
  lit by a key/fill/rim rig with ACES tone mapping against a grey backdrop.
- **Connectivity** is stored as per-ball `next`/`prev` links (a linear chain), so
  binding and cutting are just pointer edits and any ball index stays valid for the
  life of a soup. Springs follow bonds; collisions use a spatial-hash broad phase.
  Body affinity rides that same broad phase — when it's on, the grid cell is widened
  to the affinity range so one neighbor scan feeds both collisions and affinity — and
  a per-ball monomer id lets it cheaply skip same-monomer pairs.
- A `window.__dbg` handle (positions, counts, simulated cut/drag) is exposed for
  automated testing; it has no effect on behavior.

Everything lives in [`index.html`](index.html).

## License

[MIT](LICENSE)
