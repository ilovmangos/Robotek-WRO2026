# Changelog

All notable changes to the vehicle, code and documentation. Format inspired by Keep a Changelog.

## [Unreleased]
### Added
- Parallel parking state for the Obstacle Challenge (planned).
### Changed
- Motor command path: from `/motor_vel` + external driver to RRC Lite encoder motor (code update pending).

## [2026-11] - Documentation restructure for WRO 2026
### Changed
- README reorganised to follow the WRO 2026 Future Engineers documentation rubric.
- New README sections: mobility torque/speed reasoning, power budget and calibration, software architecture, systems thinking, build & reproduce; `docs/test-log.md`.
### Fixed
- Removed links to the previous (2025) repository; inconsistent battery capacity and motor-driver descriptions; broken table-of-contents anchors.

## Vehicle V12 (current)
- **Removed the external motor driver (TB6612FNG).** The drive motor is now the encoder motor of the HiWonder kit, controlled in closed loop by the RRC Lite.
- Why: repeatable speed independent of battery level, fewer wires and parts.

## Vehicle V11
- 16-LED WS2812 ring for constant lighting.
- LiDAR moved to the front, under the chassis plate (new housing).
- L298N replaced by TB6612FNG (usable from 35 % duty cycle, controlled braking/reverse).

## Vehicle V10
- LiDAR housing removed for vision tests, identical printed wheels, gear swap (big gear on motor). 3 laps in 28 s.

## Vehicle V9
- Raspberry Pi 5 stacked above batteries, MDF base, grooved wheels, spacers. 20 s per lap.

## Vehicle V1-V8
See the version table in the [README](README.md#versions-of-the-car).
