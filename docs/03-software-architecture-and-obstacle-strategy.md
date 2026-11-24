# 3. Software Architecture & Obstacle Strategy

> Rubric criterion 3. Code lives in [`src/open`](../src/open) and [`src/obstacle`](../src/obstacle). Comments in the code are written in Spanish; this document is the English reference.

## 3.1 Software stack

| Layer | Technology |
|---|---|
| OS | Ubuntu 24.04 on Raspberry Pi 5 |
| Middleware | ROS 2 Jazzy (nodes + topics) |
| Language / libs | Python 3, `rclpy`, OpenCV + `cv_bridge`, `lgpio` (legacy driver only) |
| Vendor package | HiWonder `ros_robot_controller` / `ros_robot_controller_msgs` (IMU, servo, button, buzzer, motor) |
| LiDAR driver | manufacturer ROS 2 package for the STL-19P (`/scan`) |

## 3.2 Module structure

Both challenges share the same layered design. **`control.py` has no ROS imports**: it is pure logic, so it can be unit-tested on a laptop without the robot.

| Module | Open | Obstacle | Responsibility |
|---|:-:|:-:|---|
| `config.py` | yes | yes | One dataclass with every tunable value (gains, thresholds, HSV ranges, speeds, servo mapping) |
| `control.py` | yes | yes | `InnerLoop` (gyro heading -> steering), `OuterLoop` (Open: LiDAR centering), `estimate_wall` / `wall_follow_offset` (Obstacle), `Corner`, `detect_corner`, `start_turn`, `turn_complete`, `LapCounter`, `delta_to_us` (angle -> servo µs) |
| `color_vision.py` | - | yes | HSV red / green block detector (`detect_block`) |
| `robot_node.py` | yes | yes | ROS 2 node: subscriptions, state machine, publishers, watchdog |
| `motor_driver.py` | legacy | legacy | **Obsolete.** Drove an external TB6612FNG. The car now commands the kit's encoder motor through the RRC Lite (see [changelog](../CHANGELOG.md)) |

### ROS 2 graph

| Node / source | Topic | Type | Direction |
|---|---|---|---|
| LiDAR driver | `/scan` | `sensor_msgs/LaserScan` | -> robot node (~10 Hz) |
| RRC Lite | `/ros_robot_controller/imu_raw` | `sensor_msgs/Imu` | -> robot node (~50 Hz) |
| RRC Lite | `/ros_robot_controller/button` | `ButtonState` | -> robot node (start/stop) |
| Camera driver | `/usb_cam/image_raw` | `sensor_msgs/Image` | -> robot node (Obstacle) |
| Robot node | `/ros_robot_controller/acker_servo/set_state` | `SetAckerServoState` | -> servo |
| Robot node | `/ros_robot_controller/set_buzzer` | `BuzzerState` | -> buzzer feedback |
| Robot node | `/motor_vel` | `std_msgs/Float32` (0-100 %) | -> motor |

> **[COMPLETAR - motor command path]** `/motor_vel` is read by the old `motor_driver.py`. With the encoder motor, the robot node must publish speed to the RRC Lite motor topic instead (the coach's reference car uses `/rrc/set_motor`, with a sign convention where negative rps is forward). Update `send_motor()` in both `robot_node.py` files, delete the legacy driver, and update this table. Diagram: [`schemes/ROS Node Diagram (1).png`](../schemes/ROS%20Node%20Diagram%20(1).png).

## 3.3 Control architecture - why cascade LiDAR + IMU

```
 LiDAR (10 Hz) ──► WHERE to go: target heading theta_ref  ──┐
                                                            ▼
 IMU gyro (50 Hz) ─► integrate -> theta ─► PD on (theta_ref - theta) ─► steering angle ─► servo µs
```

| Choice | Reason (data) |
|---|---|
| **Inner loop on gyro heading** | LiDAR updates at only ~10 Hz; at 0.5 m/s that is 5 cm between updates, too slow to steer smoothly. The gyro gives 50 Hz heading. The D term uses the gyro rate directly (no numerical derivative, no noise amplification): `delta = KP_IN * (theta_ref - theta) - KD_IN * gz`. |
| **Outer loop on LiDAR** | The gyro drifts; LiDAR gives absolute position relative to walls and corrects the reference heading. |
| **Gyro bias calibration at start (4 s)** | Removes constant drift before integrating. |
| **Reference clamp** (`THETA_REF_MAX_DEG` 9 deg Open, `FOLLOW_MAX_DEG` 7 deg Obstacle) | Never ask for a large heading change from a noisy LiDAR reading. |
| **Median over +-10 deg windows** (`window_median`) | Rejects spikes and single-ray dropouts. |
| **Servo command passed through an `atan` shaped distance term** (Obstacle) | Distance error is converted to a heading with `atan(LATERAL_GAIN * err)`, so a large error cannot demand an unbounded angle. |

### Parameters (current values; see `config.py`)

| Parameter | Open | Obstacle |
|---|---|---|
| Inner loop `KP_IN` / `KD_IN` | 1.6 / 0.28 | 1.8 / 0.22 |
| Outer loop `KP_OUT` / `KD_OUT` | 27 / 4.5 | (wall follower: `LATERAL_GAIN` 1.5, target 0.18 m) |
| Steering limit | 29 deg | 26 deg |
| Corner trigger (front wall) | 0.80 m | 0.65 m |
| Side opening threshold | 1.8 m | 1.2 m |
| Motor cruise / turn | 75 % / 42 % (**the README says 70 % - reconcile**) | 35 % / 35 % |

## 3.4 State machine

```
IDLE --(button)--> CALIBRATING --(4 s, car still)--> FOLLOWING <--> TURNING --(3 laps)--> DONE
  ▲                                                                                        │
  └──────────────────────────────(button pressed while driving = emergency stop)───────────┘
```

| State | Entry | What happens | Exit |
|---|---|---|---|
| IDLE | power-up or stop | motor 0, servo centred | button |
| CALIBRATING | button | average gyro Z for `GYRO_CAL_S` = 4 s -> bias; reset heading, laps, corners | 4 s elapsed |
| FOLLOWING (Open: STRAIGHT) | calibration done | outer loop (Open) or wall follower + colour logic (Obstacle) sets `theta_ref`; inner loop steers | corner detected |
| TURNING | `detect_corner` true | `theta_ref` = base heading +-90 deg; slower speed; inner loop performs the turn | `turn_complete` (heading reached and `gz` < 20 deg/s) or timeout 3.3 s |
| DONE | `LapCounter` reaches `MAX_LAPS` = 3 | motor 0, servo centred, buzzer | button |

**Rationale.** A state machine keeps the corner behaviour from fighting the straight-line controller: while TURNING the wall-follower output is ignored, so it cannot fight the 90 deg heading change. A **cool-down** of 1.5 s after each corner (`CORNER_COOLDOWN_S`) prevents the same corner from being detected twice.

### Lap counting
Corners are 90 deg each; integrated yaw reaches 360 deg after 4 corners = 1 lap. `LAP_MARGIN_DEG` = 10 deg tolerates integration error. Each corner re-bases the reference heading (`heading_base_deg`), so the controller always works with a small relative angle. **[VERIFICAR con el código si el conteo de vueltas se resetea en cada esquina]**.

### Corner detection (edge cases handled)
* Corner = **front wall closer than `CORNER_FRONT_M` AND one side open** (`SIDE_OPEN_M`). The open side also chooses the **turn direction**, so the robot works clockwise and counter-clockwise without a setting.
* Ignored while already turning, during the cool-down, and if the heading is already too far from the base heading (`CORNER_MAX_THETA_REL_DEG`).
* **Timeouts:** corner timeout 3.3 s; IMU 0.20 s and LiDAR 0.35 s data time-outs; 50 ms watchdog stops the motor.
* Invalid wall estimates (angle > 55 deg, distance > 0.9 m, rays out of range) return "no wall" and the follower outputs 0 instead of a wrong correction.

## 3.5 Obstacle Challenge strategy

1. **Wall to follow chosen by colour.** The camera finds the nearest block (`detect_block`): **red -> follow the right wall, green -> follow the left wall**. This gives the correct passing side (red pillar on the right, green on the left - rule 9.19) with a very simple rule. The colour must be seen in `CONFIRM_FRAMES` = 3 consecutive frames (avoids false triggers).
2. **Wall geometry from two LiDAR rays** (60 deg and 30 deg from the front on the followed side): the two hit points define a line -> perpendicular distance and angle, valid even when the car is not parallel.
3. **Distance target 0.18 m** from the followed wall. Heading request = wall angle + `atan(1.5 * (d - 0.18))`, clamped to +-7 deg.
4. **Pillar nudge.** While a block is near (`base_norm` > 0.5), an extra heading (max 12 deg) steers away from the side of the image where it appears.
5. **Corners and laps** identical to the Open Challenge.

**HSV colour detection** ([`color_vision.py`](../src/obstacle/color_vision.py)): red uses two hue ranges (0-13 and 169-180, S >= 80, V >= 50) because red wraps around the hue circle; green uses H 53-89. Mask is cleaned with a 7 px morphological kernel (2 iterations); a blob must have area >= 650 px, fill >= 0.42 and width/height <= 2.6 (shape of a 5 x 5 x 10 cm pillar, rejects walls and reflections). A WS2812 ring keeps illumination constant.

### Known limitations / pending work (be honest in the journal)
| Item | Status | Action |
|---|---|---|
| **Parallel parking (15 pts)** | **Not in the current obstacle state machine** - it ends in DONE after 3 laps | Implement PARK state (coach's step-8 approach: align with the lot, reverse/arc manoeuvre, replan if not parked). Even the simple "parked partly" gives 7 pts |
| Start from the parking lot (+7 pts) | not implemented | optional; **2026: only awarded if the car also completes at least one full lap** |
| Motor path | still `/motor_vel` + legacy driver | switch to RRC Lite encoder motor command |
| Cases the wall-follow logic cannot cover (e.g. pillar next to a corner, pillar in the inner lane after the corner) | **[COMPLETAR - list failures seen in tests]** | compare with the coach's "seat memory" idea |

## 3.6 Testing and tuning process, and metrics

| Step | Method | Metric recorded |
|---|---|---|
| 1. Sensor sanity | `ros2 topic echo /scan`, `/ros_robot_controller/imu_raw`, `rqt_image_view` | correct axes/signs, rates (10 Hz / 50 Hz) |
| 2. Sign check | small steer step, see which way the car goes | flip `GYRO_SIGN` / `SERVO_SIGN` first |
| 3. Inner loop | hold a heading for 5 m | straight-line deviation (cm) |
| 4. Outer loop / wall follow | run 1 lap on a mock field | mean distance error to target, oscillation amplitude |
| 5. Full runs | 10 runs per direction (CW and CCW) | **success rate** (3 laps without wall touch), **lap time**, **number of wall touches** |
| 6. Regression | change one parameter at a time, record in the test log | see [`docs/test-log.md`](test-log.md) |

Results we already have: V9 lap 20 s -> V10 28 s for 3 laps (< 10 s per lap). Full table of runs: **[COMPLETAR in `docs/test-log.md`]**.

## 3.7 Algorithm flowcharts

* Open: [`schemes/Open Challenge Flowchart.png`](../schemes/Open%20Challenge%20Flowchart.png)
* Obstacle: [`schemes/Obstacle Challenge Flowchart.png`](../schemes/Obstacle%20Challenge%20Flowchart.png)

> The state machine in 3.4 must match these flowcharts - re-export them if the PARK state is added.

## 3.8 How to read the code (60-second tour)

1. `config.py` - all numbers.
2. `control.py` - read `InnerLoop.delta` and `detect_corner` first.
3. `robot_node.py` - `scan_callback` (outer loop and corner), `imu_callback` (the only place that writes the servo), `watchdog_callback` (safety).
