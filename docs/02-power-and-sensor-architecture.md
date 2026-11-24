# 2. Power & Sensor Architecture

> Rubric criterion 2. **[COMPLETAR]** = to be measured/filled in by the team.

## 2.1 Power architecture

```
 Li-Po 7.4 V (2S) ──► RRC Lite controller ──┬─► 5 V rail ─► Raspberry Pi 5 (USB-C / GPIO power)
                          │  (power switch,  ├─► servo (Ackermann)
                          │   battery monitor)└─► USB sensors (LiDAR adapter, camera) via the Pi
                          └─► motor port ──► encoder gear-motor (7.4 V, closed-loop PID on encoder)
                          └─► WS2812 ring (5 V, **[COMPLETAR: pin/source]**)
```

* **One battery, two protected paths:** the electronics rail is regulated by the RRC Lite; the motor port is driven from battery voltage, so motor current spikes do not directly load the 5 V rail of the Raspberry Pi.
* The RRC Lite reports battery voltage (topic `/rrc/battery`), which we read before every run: >= 8.0 V = charged 2S pack, otherwise we change battery.

> **Consistency note for judges:** the previous repository mentioned an external L298N / TB6612FNG motor driver. The current car **does not use an external driver**; it uses the encoder motor channel of the HiWonder RRC Lite.

### Power budget

Fill the "Measured" column with a USB power meter / bench supply. "Datasheet" values are only a starting point.

| Consumer | Voltage | Typical (datasheet / estimate) | Peak | Measured |
|---|---|---|---|---|
| Raspberry Pi 5 (ROS 2 + OpenCV) | 5 V | ~5 W (coach's reference: 4.6 W with camera on, 3.8 W off) | up to ~12 W with USB peripherals | **[COMPLETAR]** |
| STL-19P LiDAR | 5 V | **[COMPLETAR]** | | **[COMPLETAR]** |
| Camera (USB / CSI) | 5 V | **[COMPLETAR]** | | **[COMPLETAR]** |
| 15 kg·cm servo | 5 V | idle low, high when stalled | **[COMPLETAR]** A at stall | **[COMPLETAR]** |
| WS2812 16-LED ring | 5 V | up to 16 x 60 mA = 0.96 A at full white | | **[COMPLETAR]** (we run it at reduced brightness: **[COMPLETAR]** %) |
| Drive motor | 7.4 V | **[COMPLETAR]** A running | **[COMPLETAR]** A stall | **[COMPLETAR]** |

**Battery runtime estimate:** `t [h] = C [Ah] x 0.8 / I_total [A]`, with C = **[COMPLETAR: 2000 or 5000 mAh - the old README listed both; keep the one actually installed]**. A full run (open or obstacle, 3 laps + parking) lasts < 2 min, so a single charge gives **[COMPLETAR]** runs.

### Failure points and mitigation

| Risk | Effect | Mitigation |
|---|---|---|
| Battery voltage sag under motor load | Pi brown-out / reboot | Separate motor path; check `/rrc/battery` >= 8.0 V before starting; swap packs between runs |
| Loose USB/servo connector after a crash | Sensor dropout mid-run | Strain-relief on cables, zip-ties; **sensor time-outs** in code (`IMU_TIMEOUT_S` = 0.20 s, `LIDAR_TIMEOUT_S` = 0.35 s) stop the car if data stop arriving |
| Motor command lost | Car keeps driving | Watchdog (`WATCHDOG_S` = 0.05 s) cuts motor and centres the servo |
| Pi thermal throttling | Lower frame rate | Single-level layout for airflow (V7); **[COMPLETAR]** heatsink/fan |
| Over-discharge of Li-Po | Battery damage | Low-voltage alarm from RRC Lite buzzer |

## 2.2 Sensors - selection and trade-offs

| Sensor | Role | Why this one | Alternatives we tested and why we left them |
|---|---|---|---|
| **STL-19P ToF LiDAR** | Wall distance, wall angle, corner detection | 360 deg, 5-13 Hz, 0.03-12 m, ROS 2 driver provided by the manufacturer | **Infrared (V2-V3):** single direction, noisy under room light, short range; **ultrasonic:** wide beam, ghost echoes |
| **Camera** (HiWonder monocular, 2-DOF) | Red/green pillar detection (HSV) | Colour is the only information that tells pillar sides apart | Webcam (V2-V3): auto-exposure and focus changes; replaced by the kit camera |
| **IMU (inside RRC Lite)** | Heading by integrating the gyro (`/ros_robot_controller/imu_raw`, ~50 Hz) | Fast heading loop independent of the slower LiDAR (10 Hz); gives lap count and corner completion | Compass: disturbed by motors/steel; LiDAR-only heading: too slow and noisy |
| **Motor encoder (Hall)** | Closed-loop speed in the RRC Lite | Repeatable speed regardless of battery level | External motor driver without feedback (L298N, TB6612FNG) |
| **16-LED WS2812 ring** | Constant illumination for colour detection | Room lights changed HSV values between venues | Software white balance only: not enough |

### Placement justified with field geometry

* **LiDAR height.** Walls are **10 cm** high. In V4-V5 the LiDAR was above the walls, so it "saw over" them and detected the room. In V6 we changed the wheels to lower the LiDAR; in V8 we added a small tilted mount; in V10 we removed the housing and tested vision only; in **V11 it is mounted at the front, right under the chassis plate**, which improved wall detection the most. Final height above the floor: **[COMPLETAR]** cm (must be < 10 cm minus beam half-width).
* **LiDAR rays used.** Front (180 deg), left (240 deg), right (120 deg) windows of +-10 deg (median to reject noise), plus a 60 deg and a 30 deg ray on the followed wall; two points define a line and give wall **distance and angle** even when the car is not parallel.
* **Camera.** Tilted down towards the floor, behind the LiDAR housing, so pillars (10 cm tall, 5 x 5 cm) stay in the frame while the car approaches. At the front it stuck out of the chassis and had a limited view.
* **Why the LiDAR window is 10 deg:** on the 1 m wide track at 0.18 m from the wall, a +-10 deg window covers about **[COMPLETAR]** cm of wall, enough samples at 10 Hz for a stable median.

### Calibration procedure

1. **Gyroscope bias** (every run): robot still for 4 s (`GYRO_CAL_S`), average `gz` -> subtract from every sample. Without it a 0.5 deg/s bias would add 15 deg of error over 30 s.
2. **Servo centre and gains:** command 1500 µs, measure straight-line drift over 3 m, adjust `SERVO_CENTER_US`; measure the angle reached at +-X µs to compute `SERVO_GAIN_POS/NEG` (left/right are different, see [mechanical design](01-mobility-and-mechanical-design.md#13-steering---ackermann)).
3. **Gyro sign / servo sign:** if the car steers the wrong way, flip `GYRO_SIGN` or `SERVO_SIGN` before touching gains (documented at the top of `config.py`).
4. **Camera HSV ranges:** for every venue run `color_vision.py` on a saved frame with the LED ring on; adjust `RED_*` / `GREEN_*` in `config.py`. Current values: red H 0-13 and 169-180, S >= 80; green H 53-89, S >= 25.
5. **LiDAR orientation:** verify with `ros2 topic echo /scan` that index 180 deg is the front.

## 2.3 Wiring diagrams

| Diagram | File |
|---|---|
| Visual wiring (realistic images) | [`schemes/Visual Wiring Diagram.png`](../schemes/Visual%20Wiring%20Diagram.png) |
| Electrical wiring (pin level) | [`schemes/Electrical Wiring Diagram.png`](../schemes/Electrical%20Wiring%20Diagram.png) |
| Power distribution | [`schemes/Power Distribution Diagram.png`](../schemes/Power%20Distribution%20Diagram.png) |
| ROS node diagram | [`schemes/ROS Node Diagram (1).png`](../schemes/ROS%20Node%20Diagram%20(1).png) |

> **[COMPLETAR - IMPORTANTE]** These diagrams were drawn for the car with the external driver. They must be **redrawn** for the encoder motor on the RRC Lite (remove driver, add encoder lines) and a pin table added below.

### Pin / port table (to complete)

| Device | Connected to | Signal |
|---|---|---|
| Drive motor + encoder | RRC Lite motor port **[COMPLETAR: M1-M4]** | motor power + Hall A/B |
| Steering servo | RRC Lite servo port **[COMPLETAR]** | PWM 1000-2050 µs |
| LiDAR | Raspberry Pi USB (through UART-USB adapter) | `/scan` |
| Camera | Raspberry Pi USB **[COMPLETAR: or CSI]** | `/usb_cam/image_raw` |
| RRC Lite | Raspberry Pi USB | IMU, button, buzzer, motor/servo commands |
| WS2812 ring | **[COMPLETAR]** | 5 V, GND, data |
