# SMC truck-volume LiDAR demo — findings & learnings

Live demo: **https://lumigrade.github.io/smc/** (8 vehicles in the dropdown; height-rainbow point clouds, dimension boxes, load vs bed capacity).

This is the engineering record of what we built, what worked, what didn't, and the hard-won
lessons that shape the production design (see **[ARCHITECTURE.md](ARCHITECTURE.md)**).

---

## 1. What it does

Measure how much material a truck carries, from a 3D LiDAR scan at the gate — no scale, no CAD.
Two (or more) SICK picoScan LiDARs on masts scan a truck driving through; the scans fuse into one
point cloud; comparing the **full** bed to the **empty** bed gives the **material volume**, and the
bed's own geometry gives its **max capacity**. Single vehicles (no empty pass) get a **L×W×H**
bounding measurement instead.

The viewer reports **data only** — *Actual load* vs *Bed max load* — and leaves pass/fail to the
SMC supplier contract.

## 2. The capture rig (what produced this data)

- 2× SICK picoScan LiDAR, each on a ~6 m mast either side of the lane, looking **down** at the truck.
- Each capture writes `lidar1.scn` / `lidar2.scn` (raw `PICOSCN3D`: magic + JSON header + UDP scan
  packets + IMU) and, when exported, `lidar1.ply` / `lidar2.ply` (the **accumulated 3D cloud**).
- A custom GUI tool (`main.py`) loads the A/B pair, aligns them (mast-rig or manual), and exports a
  fused PLY.

## 3. The pipeline (per vehicle)

```
fuse A+B  →  isolate the vehicle  →  orient axis-aligned  →
   paired (full+empty):  register full↔empty → classify load → water-tight bed → volume
   single:               L×W×H bounding box
→  export point clouds (.bin) + meta.json  →  web viewer
```

## 4. Key findings & learnings

### 4.1 ⭐ Sweep mode is everything: **vertical, not horizontal**
The single most important lesson. The picoScan is a 2-D line scanner; you build 3-D by sweeping it.
- **Vertical sweep** (the scan fan is vertical, swept across the lane as the truck drives through)
  packs a **dense, solid surface** → fuses cleanly, measures reliably.
- **Horizontal sweep** lays down **sparse horizontal scan-lines that never consolidate** into a
  surface. Proven inside one truck (37H-148.48): its vertical-sweep empty fused to a clean **3.2 m**
  wide bed, its horizontal-sweep full **spread to 7.1 m** of streaks. Same truck, same code.
- **Conclusion baked into the production design: sweep vertically, downward.** Horizontal sweep is
  display-only at best.

### 4.2 Fusion: the brittle path, the robust path, and the real unlock
- The original **2-mast cross-solve** (find each mast pole, solve a 2-point transform) is precise but
  **brittle** — it only worked on the one capture session it was tuned for; new sessions failed
  ("truck off 3–9 m") because the masts sat differently and competing columns confused it.
- A **robust truck-to-truck fusion** (isolate the truck in each sensor, register with a 0–360° yaw
  sweep + ICP, rig-vicinity candidate centres) fused everything *visually* — but the surface was a
  few cm soft, not enough for precise volume.
- **The real unlock was the operator hand-aligning the trucks in the GUI.** Clean fusion in →
  the whole measurement chain fell into place. **Lesson: a rock-solid, repeatable calibration beats
  any clever auto-alignment.** That's exactly why the production design uses *fixed calibration
  anchors* (see ARCHITECTURE.md) — to make that clean alignment automatic and permanent.

### 4.3 Measuring the load
- **Water-tight bed capacity** (flood-fill: "pour water until it spills over the lowest rim"). The
  bed is a **flared trapezoid**, not a box, so L×W×H over-counts ~22%; the flood-fill gets the true
  fillable volume and auto-excludes the cab (the front wall blocks the water).
  - Hardened with a **2-pass gap-close**: occlusion can punch a hole in the rim that drains the basin
    early (read a bed as 0.67 m deep instead of ~1.4 m). Pass 1 floods, finds the basin; raise the
    occluded rim gaps to the wall height; pass 2 re-floods. Clean rims are untouched.
- **Material load** = full top-surface **minus** empty top-surface over the water-tight bed,
  computed from the **same oriented surfaces shown in 3-D**. A *separate* full↔empty registration
  (the first approach) silently **lost the load** on two trucks (read 14 % / 45 % for beds the
  operator knew were ~90 % full). **Lesson: measure from the surfaces you display — one source of
  truth.**
- **Plausibility gate** — never publish an obviously-broken number. A real 8×4 dump bed is
  ~6–7 L × 2.0–3.0 W × 0.9–2.2 H m, ≥ ~10 m³; a load can't exceed ~130 % of the bed. Anything outside
  is shown display-only ("measurement pending validation").

### 4.4 Orientation & cropping
- **Min-area bounding-rectangle** orientation (sweep the angle that minimises the footprint box) is
  robust to a truck entering the gate **leaned** off the lane (we saw +5° to −30° leans). PCA is
  fooled ~8° by the asymmetric cab + heaped material; the min-area rect locks onto the straight sides.
- **Excavators have no long axis** → orient by the dense **undercarriage** (low z-band = tracks), not
  the whole boom-included footprint.
- **Generous crop (r = 7 m cylinder)** so a long bed's tail isn't clipped — clipping the tail also
  broke the bed measurement (the basin spilled into the cab).

### 4.5 Isolating a vehicle from the scene
- **Largest connected above-ground component** = the vehicle; **adaptive density threshold** drops
  sparse scan-streaks; the **crane on a crane-truck survives** (it's attached to the dense body).
- **Mast removal** = drop only **isolated** ground-to-top columns (a free-standing rig mast), *not*
  the dense body. A compact **excavator** stacks tracks under the cab in one ground-to-top column —
  the naïve rule cut the machine; the isolation test keeps it.
- **Ground-plane skirt** (a flat floor layer some fusions include) is removed by a height cut that
  only fires when >12 % of points sit in the low band — so clean trucks (wheels are a small fraction)
  are untouched.

### 4.6 The "top-down illusion"
A truck whose bed looks full from above is often only ~60–75 % full by **volume** — material heaps at
the tailgate and slopes down toward the cab. Footprint-full ≠ depth-full. The viewer's white bed
wireframe makes the fill-vs-rim depth visible so this reads honestly.

### 4.7 Operational gotchas
- **GitHub Pages caches `index.html`.** Only `.bin/.json` cache-bust (a `VER` constant + `?v=N`).
  Bump `VER` every deploy and hand out `…/?v=N`, or stale HTML looks "still broken".
- **ICP volume wobbles ±0.3 m³** run-to-run (multithreaded RANSAC) — seed it, or pin a reference.
- **`.scn` needs the capture tool to make `.ply`** — there's no `.scn` parser in the app; the raw
  capture format must be converted by the recorder/replayer. (Automating SCN→PLY is a production
  must — see ARCHITECTURE.md.)
- SICK **`.pcd` → `.ply`** is a trivial, lossless convert (x/y/z + intensity).

## 5. Results (live demo, VER 28)

| Vehicle | Type | Result |
|---|---|---|
| 37H-115.58 | dump (vertical sweep) | **13.6 m³** load / 73 % of an 18.7 m³ bed — *validated* |
| 37H-148.48 | dump (mixed sweep, hand-aligned) | **14.3 m³ / 91 %** (cap 15.6) |
| 37H-115.53 | dump (horizontal, hand-aligned) | **17.3 m³ / 93 %** (cap 18.6) |
| 37H-142.98 | dump (horizontal, hand-aligned) | **16.7 m³ / 93 %** (cap 18.0) |
| 50E-650.58 | crane truck | **11.4 × 2.8 × 4.0 m** (crane preserved) |
| 76C-102.90 | truck | 7.6 × 2.8 × 3.1 m |
| B62 long pickup | flatbed | 8.0 × 2.5 × 2.9 m |
| Máy xúc | excavator | 9.0 × 3.2 × 4.2 m (body + arm) |

## 6. Bottom line for production
1. **Sweep vertically.** Don't fight horizontal data.
2. **Make the calibration fixed and automatic** (anchors), so every fuse is clean with zero human
   alignment — the thing that made every measurement work here.
3. **Measure from one source of truth** (the displayed surfaces), gate implausible results, and
   surface *data*, not verdicts.

→ The production blueprint that builds on all of this: **[ARCHITECTURE.md](ARCHITECTURE.md)**.
