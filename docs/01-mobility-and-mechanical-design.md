# 1. Mobility & Mechanical Design

> Rubric criterion 1. Everything here is also summarised in the main [README](../README.md#6-mobility-management).
> Items marked **[COMPLETAR]** are measurements the team must take and fill in before submission.

## 1.1 Design goals and constraints

| Constraint (WRO 2025 General Rules) | Consequence for the design |
|---|---|
| Rule 9.17: vehicle must not exceed 300 x 200 mm footprint and 300 mm height | Compact single-level layout, 15 x 18 x 16 cm body (V10) |
| Drive wheels must be physically connected (e.g. gearbox); one motor per side is not allowed | One motor + gear train to a single rear axle; steering with an Ackermann linkage on the front wheels |
| One button to turn on, another to start the program | RRC Lite power switch + onboard user button (`/ros_robot_controller/button`) |
| Track border distance is 1000 mm or 600 mm (+-100 mm) in the Open Challenge, always 1000 mm in the Obstacle Challenge; walls 10 cm high | LiDAR must sit at wall height (< 10 cm) and the car must turn inside a 600 mm corridor |
| Parking lot: 20 cm wide, 1.5 x robot length long; parallel parking scores 15 points, partial 7 | Short wheelbase and large steering angle required |
| Rounds last 3 minutes | Speed profile tuned for 3 laps well below that limit (28 s in Open at V10) |

## 1.2 Chassis

* **Base:** laser-cut MDF plate designed in Fusion 360 with custom mounting holes for every component ([`models/vehicle_base`](../models/vehicle_base)).
* **Housings:** 3D-printed (PLA, upgraded to polycarbonate where strength was needed) for camera, Raspberry Pi, batteries and LiDAR ([`models`](../models)).
* **Why MDF instead of the original aluminium two-level chassis (V4-V6):** the aluminium stack was heavy and every change required un-assembling the whole car. MDF is cut in minutes, so we could iterate component positions between test sessions. See the iteration table in [04-engineering-decisions](04-engineering-decisions.md).
* **Why single level (V7):** better airflow for the Raspberry Pi 5 and easier access to cables.

## 1.3 Steering - Ackermann

| Item | Value |
|---|---|
| Mechanism | Ackermann linkage with bearings (HiWonder kit), adapted to our base |
| Actuator | 15 kg·cm digital servo, mounted **vertically** (V8) |
| Servo command | pulse width in µs; software limit 1000-2050 µs, centre 1500 µs ([`src/*/config.py`](../src/)) |
| Software steering limit | +-26 deg (obstacle) / +-29 deg (open) |
| Wheelbase / track width | **[COMPLETAR]** mm / **[COMPLETAR]** mm |
| Measured minimum turning radius | **[COMPLETAR]** m (measure with the car at full lock, centre of the rear axle) |

**Design iterations (evidence of testing):**

1. *Rack-and-pinion with stepper (V2-V3).* 3D-printed teeth were imprecise; enlarging them made the mechanism too big for the chassis.
2. *HiWonder linkage, servo horizontal (V4-V7).* Worked, but the left turn rotated less than the right one, so corners of the field were hard in the Obstacle Challenge.
3. *Servo vertical (V8).* Linkages move freely over a wider angle, we freed space and shortened the chassis. Left/right asymmetry is now compensated in software with separate gains (`SERVO_GAIN_POS` = 550/30 µs per deg, `SERVO_GAIN_NEG` = 500/30 µs per deg).

## 1.4 Drivetrain - torque and speed reasoning

The car is driven by the **encoder gear-motor from the HiWonder kit**, connected to the **RRC Lite** controller (closed-loop speed control using the motor's Hall encoder). This replaces the previous external motor driver (L298N, then TB6612FNG) - see [Changelog](../CHANGELOG.md).

**Why we changed the driver for the kit's encoder channel**

* With the L298N the car stalled below ~50 % duty (voltage drop across the bridge + friction + weight).
* The TB6612FNG fixed the stalling (usable from ~35 %) but was still open-loop: the real speed depended on battery voltage and floor friction.
* With the encoder the RRC Lite runs a PID on motor speed, so the commanded speed is repeatable between runs and while the battery discharges - the main requirement for repeatable lap times.

**Speed estimate (fill in with the real wheel):**

```
v_max [m/s] = (rpm_motor / 60) * (1 / gear_ratio_ext) * pi * D_wheel
```

| Symbol | Meaning | Value |
|---|---|---|
| rpm_motor | no-load speed at 7.4 V (kit motor datasheet; coach's reference car uses 450 rpm) | **[VERIFICAR en la ficha del motor]** |
| gear_ratio_ext | external gear stage motor -> rear axle (see V10: large gear on the motor shaft) | **[COMPLETAR]** (teeth ratio) |
| D_wheel | printed wheel + rubber protector diameter | **[COMPLETAR]** mm |
| v_max | theoretical top speed | **[COMPLETAR]** m/s |
| v_measured | measured with a stopwatch on a 3 m straight, 3 runs | **[COMPLETAR]** m/s |

**Measured performance we already have** (from our test logs, V9-V10): one lap in < 10 s and three laps in 28 s on the Open Challenge track; V9 needed 20 s per lap before the gear change.

**Gear train trade-off (V9 -> V10):** swapping the small and big gear so that the *big gear is on the motor shaft* increased wheel speed (speed-up stage) and cut the 3-lap time. The price is torque at the wheel, which is acceptable because the car is light (**[COMPLETAR]** g) and the track is flat; the cost shows during low-speed manoeuvres, where we use a minimum duty cycle of ~35 %.

**Stability fixes**

* Custom 3D-printed spacers close the gap between the metal axle supports and the chassis -> less vibration and misalignment.
* Grooved wheels + small cylinders between wheel and screws for traction (V9).
* Slimmer front wheels (V8) so they do not stick out of the chassis outline.
* All four wheels identical (V10) to simplify spares and behaviour.
* Centre of gravity kept low: batteries are in a dedicated housing right above the chassis plate, Raspberry Pi above them.

## 1.5 Speed configuration (tested values)

| Challenge | Cruise | Corner | Why |
|---|---|---|---|
| Open | see [`src/open/config.py`](../src/open/config.py) (`MOTOR_CRUISE_PCT`, `MOTOR_TURN_PCT`) | slower | LiDAR (10 Hz) needs time to see the corner wall; 100 % made the car miss corners |
| Obstacle | 35 % | 35 % | camera needs several frames to confirm colour before reaching the pillar |

> Keep this table in sync with `config.py` - the judges can compare.

## 1.6 3D / CAD files and reproducibility

All parts are in [`models/`](../models) (STL + PNG preview + README with material and print settings). Fusion 360 source files: **[COMPLETAR - subir los .f3d/.step a `models/cad/`]**. Assembly order: see [Assembly Instructions](../README.md#9-assembly-instructions) and [05-build-and-reproduce](05-build-and-reproduce.md).
