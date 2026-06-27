# Phase-1 build plan — one gate, end to end

Turns **[ARCHITECTURE.md](ARCHITECTURE.md)** (and its locked decisions) into a concrete build: prove
the full **truck → dashboard** automation on a single gate, then replicate to gates 2 and 3.

**Locked decisions:** 4 sensors/gate · vertical sweep down · empty-baseline *fingerprint* with monthly
re-validation · front ANPR camera + CPL lens (plate + driver) · full-platform dashboard.

---

## 1. Bill of materials

### Per gate
| Item | Qty | Notes |
|---|---|---|
| Rigid steel portal / gantry | 1 | Lane width + height clearance; rated for sensors + camera + wind |
| SICK picoScan100 LiDAR | **4** | Vertical sweep down; 2 per side or 4 corners (set by coverage check) |
| Sensor brackets (fixed, rigid) | 4 | No flex — sensor geometry must stay fixed |
| Calibration anchor blocks | 3–4 | Corner-cube / L-shape, matte, ~0.6–1 m, fixed mounts, non-collinear |
| Front ANPR camera + **CPL lens** | 1 | Global-shutter, IR-capable; CPL cuts windshield glare → reads plate **and** driver |
| Illuminator (IR + white) | 1 | Driver-through-windshield needs light, day & night |
| Edge PC (industrial, fanless) | 1 | ~8-core / 32 GB; GPU optional (ICP/ML); IP65 enclosure |
| PoE switch + 4G/5G router (or fibre) | 1 | Sensors + camera + WAN uplink |
| UPS + power distribution | 1 | Ride-through + clean shutdown |
| Inductive loop / light barrier *(optional)* | 1 | Backup trigger |

### Central (shared by all gates)
- Cloud app server + **PostgreSQL** + **object store** (point clouds + plate/driver images) + the web app.
- **Total station** (shared tool) to survey anchor positions once per gate.

## 2. Sensor layout & calibration (4 sensors)
- 4 sensors → top + both sides + reduced front/rear shadow. Run a quick coverage check (mount
  height/angle) so the **bed walls + heaped top** are well sampled (needed for the fingerprint and the
  single-pass fallback).
- **Anchors:** ≥3 per sensor, non-collinear, spread in depth. Survey once into the gate frame. Solve
  each sensor→gate transform at install; **daily self-check** on the anchor residual flags a knock.
- **Background:** capture the empty gate once → static background for instant truck isolation.

## 3. SCN→PLY service (headless)
The one genuinely new component; everything downstream is already proven by the demo.
- **Parse `PICOSCN3D`:** magic + version, JSON header (sensor cfg, mount rpy, ports), then the scan
  packets (per-scan 2-D ranges + scan/encoder angle + timestamp) and the IMU stream.
- **Deskew + accumulate:** project each scan-line to 3-D from the sweep angle + sensor mount, time-
  aligned with the IMU — i.e. the picoScan capture math the GUI tool already does, ported to a headless
  library/service.
- **Output** a PLY per sensor (x/y/z + intensity), then apply the calibration transform and concat all
  4 → the **gate-frame fused cloud**.
- **Acceptance:** byte/geometry parity vs the GUI tool's PLY on a set of captures.

## 4. Edge pipeline (per pass — state machine)
```
idle → DETECT (LiDAR points enter the lane above background)
     → RECORD scn (all 4 sensors)
     → SCN→PLY → FUSE (apply anchor transforms) → SUBTRACT background → ISOLATE truck
     → ORIENT (min-area-rect)
     → ANPR (plate) + DRIVER capture
     → REGISTRY lookup by plate:
          empty pass  → store/refresh fingerprint
          loaded pass → subtract fingerprint → load
     → MEASURE (load m³, fill %, bed L×W×H)  + CONFIDENCE + plausibility GATE
     → PUBLISH {pass record + downsampled cloud + plate/driver images} → idle
```
Buffer locally on a WAN drop; sync on reconnect. NTP-synced across gates.

## 5. Empty-baseline + monthly re-validation (the fingerprint)
- **Fingerprint** = empty bed surface (rim/floor/walls) + silhouette + bed L×W×H, keyed by plate, in
  the central registry (shared by all gates).
- **Loaded pass** → look up by plate → subtract → load. Per-pass sanity: the bed walls visible *below*
  the material must match the fingerprint → mismatch flags a wrong plate or a modified truck.
- **Monthly:** schedule each truck for an empty re-scan (or catch a natural empty pass); diff against
  the stored fingerprint. If the bed changed > threshold (raised sides / extension / new bed) →
  **alert + update the fingerprint fleet-wide** and mark that truck's recent loads for review.
- **No fingerprint yet?** Fall back to the single-pass truck-model template so the first loaded pass
  still returns a result.

## 6. Dashboard — full platform
**Data model (Postgres):**
`Gate` · `Sensor` · `Calibration`(version, anchor_residual) · `Truck`(plate, fingerprint_ref, model,
registered_at, last_validated_at, status) · `Pass`(gate, ts, plate, driver_img, load_m3, fill_pct,
bed_dims, cloud_url, confidence, flags) · `Driver`(opt.) · `User`/`Role` · `Alert`.

**Features:** live pass feed; per-truck & per-gate history + trends; the **3-D viewer per pass** (this
repo's viewer, generalized); **exception/review queue** (gated/low-confidence results); the **monthly
re-validation workflow** with before/after bed diffs; reports & exports (CSV/PDF); auth + roles
(operator / admin / auditor); multi-gate / multi-site.

> **Privacy/compliance — flag now:** the CPL-lens driver images are **personal data**. Define lawful
> basis, on-site signage/consent, access control, and a retention policy before capturing faces.
> Keep driver images access-restricted and separable from the load data.

## 7. Indicative timeline
| Wk | Work |
|---|---|
| 1–2 | Portal + 4-sensor mount; survey anchors; capture background (one gate) |
| 2–4 | SCN→PLY headless service + parity test; calibration solver |
| 3–5 | Edge pipeline + auto-trigger + ANPR/driver-camera integration |
| 4–6 | Empty-baseline fingerprint + monthly re-validation logic |
| 5–8 | Dashboard MVP: live feed, history, 3-D viewer, exception queue, auth/roles |
| 8 | Phase-1 acceptance on one gate → then replicate to gates 2 & 3 |

## 8. Open technical items
- Confirm picoScan100 **range / FOV / scan-rate** vs lane geometry + truck speed (point density at
  drive-through speed).
- ANPR engine (commercial vs open ALPR) + day/night driver capture (CPL + IR).
- Cloud vendor + **data residency** for clouds and driver images.
- Threshold tuning for the monthly fingerprint-diff (what counts as "modified").
