# Rubric checklist and open items

Based on the *Future Engineers 2026 Documentation Rubric* (5 criteria x 6 points = 30) and the WRO 2026 Game Rules (Appendix C). Final: **8 December 2026**.

## Commit deadlines (2026 Game Rules, ch. 7)
| Commit | Deadline | Status |
|---|---|---|
| 1st (>= 1/5 of final code) | 2 months before (8 Oct) | earlier history already exists in this repository |
| 2nd | 1 month before (8 Nov) | earlier history exists |
| **3rd** | **2 weeks before = 24 Nov** | **this restructure is the 3rd commit. Per the 2026 rules it is mainly used to score the documentation; later changes may not be considered, so make sure everything important is on `main` at this point** |

> Also in 2026: the repository link must be sent **3 weeks before the competition** (date announced by the organizers) - confirm with the coach that it was sent. The documentation is judged with the **Engineering Journal (PDF) + GitHub**, and the hard copy is also required.

## What earns the top score (6) per criterion, and where it is covered
| # | Criterion | Needs for 6 points | Covered in | Still missing |
|---|---|---|---|---|
| 1 | Mobility & mechanical | torque/speed reasoning, trade-offs, why components, tests | docs/01 | wheel diameter, gear ratio, motor datasheet values, measured speed, turning radius, vehicle mass |
| 2 | Power & sensors | power budget, sensor trade-offs, placement from field geometry, calibration, failure points, iterations | docs/02 | **measured** currents, battery capacity, redrawn wiring diagram without driver, pin table |
| 3 | Software & obstacle strategy | state machine with rationale, algorithm justification, edge cases, tuning process, metrics | docs/03, src | parking, test results table, flowcharts updated, motor path |
| 4 | Systems thinking | constraints, trade-offs, iterations, risks, "why X not Y" with data | docs/04 | real numbers in the V12 row and R8 |
| 5 | Reproducibility & GitHub | README >= 5000 chars, >= 3 commits, CAD/wiring/code, testing workflow, versioning/release notes | README, docs/05, CHANGELOG | CAD source files (.f3d/.step), vendor workspace instructions, tag + GitHub Release |

## Action list (priority order)
1. **Code:** switch `send_motor()` to the RRC Lite motor command and delete `motor_driver.py` (or move it to `src/legacy/`). Test on the car.
2. **Parking:** implement at least a fixed-sequence parallel parking (7-15 pts), then document it and update the flowchart.
3. **Measure and fill every `[COMPLETAR]`** (search the repo: `grep -rn "COMPLETAR" .`).
4. **Redraw** wiring and power diagrams for the car without the external driver; export PNG into `schemes/`.
5. **New photos** of the current car (6 sides + 3D renders) and V12 in the version table; new videos of the current car.
6. **Upload CAD sources** (`models/cad/`) and the vendor package instructions.
7. Fill [`test-log.md`](test-log.md) with real runs (10 per direction).
8. Reconcile numbers: Open speed (README vs `config.py`), battery capacity (2000 vs 5000 mAh), car dimensions.
9. Create tag `v2026-final` + GitHub Release; keep the repository public for 12 months.
10. Print README + docs for the hard copy.

## Mistakes seen in other repositories that cost points (avoid)
* Power section without measured numbers or battery runtime (best teams show a runtime calculation and safety measures).
* Photos without all six sides or without labels; team photo missing.
* Videos under 30 s of autonomous driving, or not showing the final car.
* Obstacle section describing code but not the *decision logic* (flow diagram, tuning notes).
* Diagrams that do not match the final car (our own case: driver still in the diagrams).
* Many large binaries in `git` history, README images hosted outside the repo that can break.
