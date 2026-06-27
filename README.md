# SMC — Truck Material Volume by 3D LiDAR

**Live demo → https://lumigrade.github.io/smc/**

Smart Material Control measures how much material a truck carries at the site gate — automatically,
from SICK picoScan LiDAR scanners fused into a single 3D model. No scale, no CAD: the LiDAR point
cloud is a real measurement. Comparing the **full** vs **empty** bed gives the **material load**; the
bed's own geometry gives its **max capacity**. Single vehicles get an **L × W × H** bounding measurement.

The viewer (this page) has a **dropdown of 8 vehicles** — 4 measured dump trucks, 3 single trucks, and
an excavator — with height-rainbow point clouds, dimension boxes, and *Actual load vs Bed max load*
(reported as **data**; pass/fail is per the SMC supplier contract).

## Documentation

- **[FINDINGS.md](FINDINGS.md)** — the full engineering record: what we built, what worked, what
  didn't, and the hard-won lessons (sweep mode, fusion, water-tight bed, load measurement, isolation).
- **[ARCHITECTURE.md](ARCHITECTURE.md)** — the production blueprint: an automated, multi-gate,
  vertical-sweep system with fixed calibration anchors, SCN→PLY→dashboard, scaling to 3 interconnected
  sites with minimal human touch.
- **[BUILD_PLAN.md](BUILD_PLAN.md)** — the concrete Phase-1 build (one gate, end to end): bill of
  materials, the SCN→PLY service, the edge pipeline, the empty-baseline *fingerprint* + monthly
  re-validation, the full-platform dashboard schema, and a timeline.

## Repo

- `index.html` — the Three.js viewer (single file).
- `clouds/<vehicle>/{structure,material,empty}.bin` + `meta.json` — per-vehicle point clouds + results.
- `clouds/manifest.json` — the dropdown list.

Lumigrade · Sun Group.
