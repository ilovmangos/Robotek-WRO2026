# 4. Systems Thinking & Engineering Decisions

> Rubric criterion 4: subsystem interaction, constraints and trade-offs, iteration/test cycles, risks and mitigations, "why X instead of Y".

## 4.1 How the subsystems work together

```
            ┌────────────── Raspberry Pi 5 (Ubuntu 24.04 + ROS 2 Jazzy) ───────────────┐
 LiDAR ───► │ /scan (10 Hz) ─► wall geometry, corner detection ─┐                         │
 Camera ──► │ /usb_cam/image_raw ─► HSV block detector ─────────┼─► state machine ─► θ_ref │
 RRC Lite ► │ IMU /imu_raw (50 Hz) ─► heading θ ────────────────┘        │                │
            │                                inner PD loop ◄─────────────┘                │
            └───────────┬─────────────────────────────┬──────────────────────────────────┘
                        ▼ servo command (µs)          ▼ speed command
                   Ackermann servo               encoder motor (RRC Lite closed loop)
```

| Interaction | Dependency / constraint |
|---|---|
| Speed <-> LiDAR | LiDAR scans at 10 Hz: the faster the car, the less it "sees" per metre. Open speed is capped (not 100 %) so corners are detected in time. |
| Speed <-> camera | Pillar colour must be confirmed in 3 frames before the car reaches the pillar -> Obstacle speed is 35 %. |
| LiDAR height <-> chassis/wheels | Walls are 10 cm; wheel diameter sets chassis height, which sets LiDAR height. A wheel change in V6 fixed missed walls. |
| Lighting <-> camera | HSV thresholds depend on light; the LED ring makes them venue-independent. |
| Battery voltage <-> motor speed | Open-loop drivers changed speed with battery level; encoder closed loop removes that dependency. |
| Servo mounting <-> steering geometry <-> software | Mechanical left/right asymmetry is compensated by separate servo gains. |
| Gyro drift <-> lap counting | Bias calibration at start and relative heading at every corner keep laps accurate. |

## 4.2 Constraints and trade-offs

| Decision | Chosen | Rejected | Why (data / test) |
|---|---|---|---|
| Distance sensing | LiDAR STL-19P | Infrared, ultrasonic | IR gave noisy, single-direction values (V2-V3); LiDAR gives wall distance **and** angle in one scan |
| Heading | Gyro integration (IMU in RRC Lite) | LiDAR-only heading | LiDAR 10 Hz is too slow to damp oscillations; gyro at 50 Hz is |
| Computing | Raspberry Pi 5 + ROS 2 | Arduino Nano only | Needed OpenCV + LiDAR processing; ROS 2 splits the code into small testable nodes |
| Colour logic | HSV thresholds | Neural network | Only two colours, fixed lighting; HSV runs in a few ms on the Pi and is easy to tune at the venue |
| Chassis | Laser-cut MDF, single level | Aluminium two-level (V4-V6) | Weight, airflow, and the time needed to modify a part |
| Steering | HiWonder linkage + vertical servo | 3D-printed rack and pinion + stepper | Printed teeth were imprecise and bulky; vertical servo gave wider angle and shorter chassis |
| Motor control | **Encoder motor via RRC Lite** | L298N -> TB6612FNG | L298N stalled below 50 %; TB6612FNG fixed low-speed (35 %) but was open loop; encoder gives repeatable speed with the same hardware family (fewer parts and wires) |
| LiDAR mount | Front, under the chassis plate (V11) | On top (V5-V9), none (V10) | On top the beam passed over 10 cm walls; no LiDAR gave worse corner detection; under the plate improved wall detection most |
| Lighting | Added 16-LED ring | Software white balance | Room light changed detection results between sessions |
| Housing material | PLA -> polycarbonate | PLA only | Crash resistance |

## 4.3 Iteration history with measured effect

| Version | Change | Effect observed |
|---|---|---|
| V1 | Cardboard prototype | Learned Ackermann geometry and component placement |
| V2-V3 | Acrylic chassis, stepper rack-and-pinion, IR sensors, webcam, Arduino Nano + Pi 4 | First autonomous attempts; imprecise steering, noisy sensors |
| V4 | HiWonder kit base, LiDAR, monocular camera, servo steering, start of ROS | Better detection; two motors on one wheel for rule compliance |
| V5 | Two bases joined, LiDAR on top, camera housing | Cleaner layout |
| V6 | Wheels changed to lower the LiDAR; polycarbonate housings | LiDAR sees 10 cm walls; **Open Challenge completed** |
| V7 | New smaller chassis, single motor + gears, single level | First traffic signs detected and avoided |
| V8 | Slimmer front wheels, battery housing, **servo vertical**, tilted LiDAR mount | Wider steering angle; wall detection improved |
| V9 | Pi 5 stacked, MDF base, grooved wheels, spacers | **Lap in 20 s** |
| V10 | LiDAR housing removed (vision tests), identical wheels, gear swap | **< 10 s per lap, 3 laps in 28 s** |
| V11 | LED ring, LiDAR re-mounted at front under chassis, TB6612FNG | Stable colour detection; smoother low-speed control (from 35 %) |
| **V12 (current)** | **External driver removed; encoder motor from HiWonder kit via RRC Lite** | **[COMPLETAR: measured speed repeatability, lap time]** |

## 4.4 Risk register

| # | Risk | Likelihood | Impact | Mitigation | Evidence |
|---|---|---|---|---|---|
| R1 | Colour misdetected under venue lighting | High | Wrong side of pillar (penalty) | LED ring, HSV calibration step at the venue, 3-frame confirmation | `CONFIRM_FRAMES`, section 2.2 |
| R2 | LiDAR sees over walls / noise outside field | Medium | Wrong centering | Mount below 10 cm; median filters; range limits | V6, V11 |
| R3 | Gyro drift | Medium | Wrong turn / lap count | Bias calibration; relative heading at corners | `GYRO_CAL_S` |
| R4 | Sensor stops publishing | Low | Car drives blind | Time-outs and watchdog stop the motor | `IMU_TIMEOUT_S`, `WATCHDOG_S` |
| R5 | Battery low | Medium | Reboot / slow car | Voltage check before each run; encoder closed loop | section 2.1 |
| R6 | Mechanical looseness after crashes | Medium | Veering | Pre-run checklist in [05](05-build-and-reproduce.md#54-pre-run-checklist) | spacers V9 |
| R7 | Parking not implemented / fails | High (today) | Up to 22 pts lost | Implement PARK state; fall back to "stop inside lot" | section 3.5 |
| R8 | Different corridor width (600 vs 1000 mm) | Medium | Wrong centering | Outer loop is relative (D1-D2 = 0), thresholds checked on both widths | **[COMPLETAR: test on 600 mm]** |

## 4.5 Lessons learned
* Change **one** variable per test and log it.
* Most "software" problems in V4-V8 were actually **sensor placement** problems.
* Replace a part only after measuring a failure (e.g. TB6612FNG after seeing the L298N stall at 50 %).
