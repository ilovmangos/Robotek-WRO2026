# 5. Build, Install and Reproduce

> Goal (rubric criterion 5 and rule 7): another team must be able to rebuild and run this vehicle from this repository only.

## 5.1 Repository map

| Path | Content |
|---|---|
| `README.md` | Overview, team, vehicle, summary of every rubric section |
| `docs/` | Detailed documentation (one file per rubric criterion), test log |
| `src/open/`, `src/obstacle/` | ROS 2 Python nodes and pure-logic modules |
| `models/` | STL + PNG preview + README per printed/cut part |
| `schemes/` | Wiring, power, ROS and flow diagrams |
| `t-photos/`, `v-photos/` | Team photos; vehicle photos (6 sides + versions) |
| `videos/` | Links to YouTube videos (one per challenge) |
| `CHANGELOG.md` | Release notes / versions of the car and code |

## 5.2 Bill of materials (summary)

Full BOM with images: [README - BOM](../README.md#bom-bill-of-materials). Core parts: Raspberry Pi 5, HiWonder RRC Lite controller, encoder gear-motor (kit), 15 kg·cm servo, STL-19P LiDAR + adapter, monocular camera, 2S Li-Po, WS2812 16-LED ring, MDF base, printed housings and wheels.

## 5.3 Build and install

### Mechanical
1. Cut `models/vehicle_base` in 3 mm MDF (laser) **[COMPLETAR: thickness]** or print it.
2. Print housings and wheels (PLA, 0.2 mm layer, no supports - see each `models/*/README.md`).
3. Assemble Ackermann, motor/gear train and housings following [README - Assembly Instructions](../README.md#9-assembly-instructions).
4. Wire following [`schemes/`](../schemes) and the pin table in [02](02-power-and-sensor-architecture.md#pin--port-table-to-complete).

### Software (Raspberry Pi 5)
```bash
# 1. Flash Ubuntu 24.04 (Raspberry Pi Imager), enable SSH
# 2. Install ROS 2 Jazzy (deb packages): https://docs.ros.org/en/jazzy/Installation/Ubuntu-Install-Debs.html
sudo apt install ros-jazzy-cv-bridge python3-opencv python3-lgpio
# 3. Vendor workspace (HiWonder ros_robot_controller + msgs) and the STL-19P LiDAR driver
#    [COMPLETAR: exact repo/branch and build commands used on our Pi]
# 4. Clone and run
git clone https://github.com/ilovmangos/Robotek-WRO2026.git
cd Robotek-WRO2026
source /opt/ros/jazzy/setup.bash
source ~/ros2_ws/install/setup.bash           # workspace with the vendor packages
ros2 launch <lidar_package> <lidar_launch>.py &  # [COMPLETAR] publishes /scan
ros2 run usb_cam usb_cam_node_exe &              # publishes /usb_cam/image_raw
cd src/open     && python3 robot_node.py         # Open Challenge
cd src/obstacle && python3 robot_node.py         # Obstacle Challenge
```
Press the RRC Lite user button: 4 s gyro calibration (keep the car still), then it drives. Press again to stop.

> Rule 9.10/9.11: one switch to power on, one button to start - exactly this procedure.

## 5.4 Pre-run checklist
- [ ] Battery >= 8.0 V, cables tight, wheels rotate freely
- [ ] `ros2 topic hz /scan` ~10 Hz, `/ros_robot_controller/imu_raw` ~50 Hz, camera image visible
- [ ] LED ring on; HSV test on a saved frame at the venue
- [ ] Car standing still on the start section (gyro calibration)
- [ ] `config.py` values match the track width (1000 vs 600 mm)

## 5.5 Testing workflow
See [`docs/test-log.md`](test-log.md): every test = date, version, parameter changed, 10 runs per direction, success rate, lap time, notes. Pure logic in `control.py` can be run on a PC without the robot (`python3 -c "import control"` after adding a fake `config`).

## 5.6 Versioning and release notes
* Hardware versions V1-V12 are in the README and [`v-photos/vehicle-versions`](../v-photos/vehicle-versions/README.md).
* Software and documentation changes are in [`CHANGELOG.md`](../CHANGELOG.md). Tag the competition snapshot: `git tag v2026-final` and publish a GitHub Release.

## 5.7 Competition rule checklist for the repository (WRO 2025 General Rules, ch. 7 / App. C)
| Requirement | Where |
|---|---|
| README.md >= 5000 characters, English | `README.md` |
| Photos of vehicle (6 sides) + team photo | `v-photos/`, `t-photos/` |
| YouTube link per challenge, >= 30 s of autonomous driving | `videos/README.md` |
| >= 3 commits (1st >= 2 months, 2nd >= 1 month, 3rd >= 2 weeks before the event; 1st must contain >= 1/5 of final code) | git history |
| Public repository for >= 12 months after the event | GitHub settings |
| Code comments | `src/` |
| Hard copy at the international final | print `README.md` + `docs/` |
