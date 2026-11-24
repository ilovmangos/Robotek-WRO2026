# Test log

One row per test session. Fill in **after every session**; this is the evidence for "testing and iterations" (rubric criteria 1-4).

| Date | Car version | Challenge | Parameter(s) changed | Runs (CW / CCW) | Success (3 laps, no wall touch) | Avg 3-lap time (s) | Notes / what we learned |
|---|---|---|---|---|---|---|---|
| (past) | V9 | Open | wheels grooved, spacers | - | - | ~60 (20 s/lap) | first stable laps |
| (past) | V10 | Open | gears swapped (big gear on motor shaft) | - | - | 28 | < 10 s per lap |
| **[COMPLETAR]** | V11 | Obstacle | LED ring + LiDAR at front | | | | colour detection stable |
| **[COMPLETAR]** | V12 | Open | encoder motor, speed via RRC Lite | | | | |
| **[COMPLETAR]** | V12 | Obstacle | | | | | |

## Metrics definitions
* **Success**: three laps completed, stops in the starting section, no wall pushed (Open: no touching outer wall).
* **Lap time**: stopwatch or ROS bag timestamps.
* **Wall touches / pillar displacements**: counted by an observer.
* **Parking** (when implemented): parked fully / partly / failed.
