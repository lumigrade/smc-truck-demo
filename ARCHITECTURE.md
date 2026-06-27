# Production architecture — automated multi-gate truck-volume scanning

Goal: turn the proven demo into a **hands-off** system — trucks drive through a gate, a dashboard
shows the load. Scale to **3 gates at separate sites**, interconnected, with minimal human touch.

Grounded in the lessons of **[FINDINGS.md](FINDINGS.md)**: *sweep vertically*, *make calibration
fixed & automatic*, *measure from one source of truth*.

---

## 0. The shift from the demo

| | Demo (today) | Production (this design) |
|---|---|---|
| Sweep | mixed (we learned: vertical only) | **vertical, downward** |
| Alignment | manual in a GUI | **automatic, from fixed anchors** |
| Trigger | operator | **LiDAR auto-detects a truck** |
| SCN→PLY | GUI export | **headless service** |
| Output | hand-deployed web page | **live dashboard, auto-published** |
| Scale | one bench | **3 edge gates → 1 central dashboard** |

## 1. Per-gate hardware

```
            ┌──────── rigid portal frame (fixed geometry) ────────┐
            │                                                      │
        [S1]│  vertical                              vertical  │[S2]
         \  │   fan ↓↓↓                              ↓↓↓ fan   │  /
          \ │      ┌───────────── lane ─────────────┐         │ /
   anchor ▣ │      │   →  truck drives through  →    │ ▣ anchor│
          ▣ └──────┴─────────────────────────────────┴─────────┘ ▣
       (≥3 fixed blocky calibration anchors, surveyed once, in every sensor's view)
```

- **2 sensors** (one each side, looking down) = the proven layout. **4 sensors** (2 per side, or 4
  corners) when you need to kill cab-shadow / heaped-load occlusion and want redundancy. *Start with
  2, wire the portal for 4.*
- **picoScan100, vertical sweep down.** Confirm the chosen model's range covers the worst-case
  diagonal (mount height + half lane width to the far wheel) with margin; vertical fan across the lane.
- **Rigid portal**, not independent masts — fixed sensor geometry is what makes calibration stable.
- An **edge PC** (industrial mini-PC / Jetson / NUC) bolted to the portal does all local compute.
- *(Optional)* an **ANPR camera** for the plate, and a **light-barrier/loop** as a backup trigger.

## 2. Calibration — fixed anchors (the core idea)

This replaces the brittle mast-detection / manual alignment that limited the demo.

- Place **≥3 fixed, blocky, distinctive targets** (corner-cube or L-shaped blocks, ~0.5–1 m, matte,
  **non-collinear, spread in 3D**) where each sensor sees **≥2–3** of them — on the portal legs /
  ground posts, **outside the truck path**.
- **Survey their positions once** (total station) into the gate's coordinate frame.
- Every scan, each sensor **detects its anchors** (static, known shape & position) and solves its
  **pose → gate-frame transform** (3D point-set / PnP-style fit). Because the anchors never move, this
  is rock-solid and repeatable — *the clean alignment that made the demo work, now automatic and
  permanent*.
- Two modes: **solve-once at install** (fast; re-verify daily — a growing anchor residual flags a
  knocked sensor) or **solve-every-scan** (absorbs vibration/drift). Recommend solve-once + daily check.
- Bonus: the anchors + empty gate define the **static background**. Capture it once → **subtract it**
  from every scan to isolate the truck instantly (no blob-finding needed).

## 3. The automated pipeline (per pass)

```
 truck enters → LiDAR auto-trigger
   → each sensor records SCN (timestamped, gate-id)
   → SCN→PLY  (headless deskew+accumulate, the picoScan capture math)
   → FUSE     (apply each sensor's fixed gate-frame transform — no alignment step)
   → ISOLATE  (subtract static background → the truck)
   → ORIENT   (min-area-rect → axis-aligned, handles a leaned entry)
   → MEASURE  (water-tight bed capacity + load, or L×W×H)
   → ID       (plate via ANPR / RFID)
   → PUBLISH  {gate, time, plate, load m³, fill %, bed L×W×H, cloud, photo} → dashboard
```

All steps run **unattended on the edge node**. The two pieces to build new: the **headless SCN→PLY
service** and the **auto-trigger** (both are well-defined; the demo already proves everything
downstream).

## 4. Getting the load without a manual "empty" pass (decide this)

A single gate usually sees a truck **once** (loaded). Three ways to still get the load:

- **A · two-pass** — empty in / full out (or entry + exit gates); pair by plate + time. Cleanest *if*
  the site gives a free empty pass.
- **B · empty-baseline library** — store each truck's empty-bed shape the first time it passes empty
  (keyed by plate); subtract thereafter. Self-building, shared across gates via the central registry.
- **C · single-pass** — detect the bed (rim + floor + walls) from the loaded scan + a truck-model bed
  template; load = material above the floor, capacity = bed volume. **Most hands-off**; needs the bed
  walls well-sampled (vertical sweep + 4 sensors help).

Recommendation: **target C, fall back to B** (auto-built), use A where the layout offers it.

## 5. Multi-gate topology (3 sites)

```
   GATE 1 (site A)        GATE 2 (site B)        GATE 3 (site C)
   ┌──────────────┐       ┌──────────────┐       ┌──────────────┐
   │ sensors+portal│      │ sensors+portal│      │ sensors+portal│
   │ edge PC:      │      │ edge PC:      │      │ edge PC:      │
   │  capture→     │      │  capture→     │      │  capture→     │
   │  SCN→PLY→     │      │  SCN→PLY→     │      │  SCN→PLY→     │
   │  measure      │      │  measure      │      │  measure      │
   └──────┬───────┘       └──────┬───────┘       └──────┬───────┘
          │ results + small cloud + thumbnail (over 4G/5G/fibre)         │
          └───────────────┬──────┴───────────────┬──────┴──────────────┘
                          ▼                       ▼
                  ┌────────────────────────────────────┐
                  │      CENTRAL CLOUD / DASHBOARD       │
                  │  • DB: passes, per-truck history     │
                  │  • truck registry + empty baselines  │  ← shared across all gates
                  │  • 3D web viewer (this repo, scaled) │
                  │  • alerts / exception queue          │
                  └────────────────────────────────────┘
```

- **Edge does the heavy lifting** (raw clouds never cross the WAN — only results + a downsampled cloud
  + a thumbnail go up). Keeps bandwidth tiny and the dashboard fast.
- **Gates are independent** (one going down doesn't affect the others) but **report to one dashboard**.
- The **central registry** holds plates + empty baselines, so a truck's baseline learned at gate 1 is
  used at gate 3.
- **Resilience**: each edge buffers locally on a WAN drop and syncs on reconnect; all gates NTP-synced.

## 6. Minimizing human touch

| Step | How it's hands-off |
|---|---|
| Alignment | **fixed anchors** → never align by hand again |
| Start a scan | **LiDAR vehicle-detection** auto-trigger |
| SCN→PLY | headless service on the edge |
| Truck ID | ANPR camera (or RFID) |
| Measure + publish | unattended pipeline → dashboard |
| **Only human:** | install once (mount, place + survey anchors, capture background); then review only the **flagged exceptions** the dashboard queues |

## 7. Phased rollout

1. **Phase 1 — one gate, end to end.** Vertical sweep, 2 sensors, fixed anchors, single-pass
   measurement, headless SCN→PLY→dashboard. Prove the automation on one site.
2. **Phase 2 — harden.** 4 sensors if coverage needs it; auto empty-baseline library; ANPR plates;
   exception/review dashboard; daily calibration self-check.
3. **Phase 3 — replicate to 3 gates + central registry/dashboard.** One truck's data follows it
   across sites.

## 8. Decisions I need from you

1. **Sensors per gate:** 2 (proven, cheaper) or 4 (better coverage/redundancy)?
2. **Load method:** single-pass (C), empty-baseline (B), or two-pass (A) — i.e. *do trucks ever pass
   empty at a gate?*
3. **Plate capture:** ANPR camera, RFID tags, or manual entry?
4. **Anchors:** purpose-built blocks, or can we use existing fixed site features?
5. **Dashboard:** extend this web viewer + a small backend, or a full platform (auth, reports, export)?

Answer these and I'll turn this into a concrete Phase-1 build plan (bill of materials, the SCN→PLY
service spec, the edge pipeline, and the dashboard schema).
