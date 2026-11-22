# PiCarX Project Plan

*Last updated: 2026-11-22*

## 1. Goal

Turn a SunFounder PiCar-X into a platform for learning and building autonomy, one feature at a time. Write the low-level layers (motor control, sensors, eventually the camera) in C++. Use C# for the operator dashboard and Python for vision and ML experiments. Every milestone ends with something that can be demonstrated.

**Long-term goal:** autonomous driving (line and lane following, then odometry, localization and mapping).
**Along the way:** teleoperation (video, telemetry, Xbox controller), added hardware (SPI IMU, encoders, lidar), and possibly custom firmware for the Robot HAT's MCU.

---

## 2. Hardware reference

### Platform
- **Raspberry Pi 4, 4 GB**, hostname `creampi`, user `dmoore`
- Raspberry Pi OS **Bookworm, 64-bit** (`aarch64`), **libgpiod v1** command-line tools
- **SunFounder Robot HAT V4**
- On the car:
  - 2× TT drive motors (rear, **no encoders**)
  - 3× servos (steering, camera pan, camera tilt)
  - Pi camera on the pan/tilt mount
  - Ultrasonic module
  - 3-channel grayscale (line/cliff) module
  - Speaker (I2S)

### Power system
| Item | Spec |
|---|---|
| Battery | 2S 18650 pack, 2000 mAh, 6.0–8.4 V (7.4 V nominal), XH2.54 3-pin with balance tap; over-discharge cutoff at 6.0 V |
| Regulator | **A single 5 V / 3 A buck converter** feeds the Pi (through the GPIO header), the servos, the motor driver and the MCU |
| Motor driver | 5 V, 1.8 A × 2. **The motors run from the regulated 5 V rail, not from the battery directly** |
| Charging | USB-C, 5 V/2 A (10 W), with cell balancing. Charges with the switch OFF. Can drive while charging, but the battery makes up any load above 10 W |
| Power switch | ON = 5 V output enabled. OFF = output cut (<0.5 mA draw). Switching off is an instant power cut, so run `sudo shutdown -h now` first |
| Battery sensing | MCU ADC channel 4 through a 20k/10k divider: `Vbat = raw/4095 × 3.3 × 3` |

**Power risk:** the Pi, motors and servos all share one 3 A budget. Motor inrush or a stalled servo can drop the voltage and reboot the Pi. Check the budget before adding hardware (a lidar especially).

### LEDs
| LED | Meaning |
|---|---|
| CHG (red) | Solid = charging; off = full; slow blink after ~4 h = reminder to unplug; blinking while charging = fault |
| PWR | 5 V output is on |
| Battery ×2 | 2 lit = >7.6 V, 1 lit = 7.15–7.6 V, 0 lit = <7.15 V (rough, and sags under load) |
| USER | GPIO26; ours to use from code |

### Pin map (BCM GPIO numbers)
| Function | Pi GPIO | Notes |
|---|---|---|
| I2C SDA/SCL | 2 / 3 | MCU at 0x14; also the I2C header and Qwiic port |
| MCU reset | 5 | Hold low 10 ms, then high 10 ms. **Don't touch otherwise** |
| Motor 1 direction | 23 | Speed = MCU PWM channel 13 (register 0x2D) |
| Motor 2 direction | 24 | Speed = MCU PWM channel 12 (register 0x2C) |
| USER LED | 26 | Active-high |
| USR button | 25 | **Active-low; needs the Pi's internal pull-up**; bounces badly |
| RST button | 16 | Same as USR |
| D0–D3 headers | 17, 4, 27, 22 | Wired straight to the Pi |
| SPI | 8 (CE0), 9, 10, 11 + **6 (BSY)** | GPIO6 is free to use as an interrupt line (e.g. the IMU's) |
| UART | 14 / 15 | Future lidar |
| I2S audio | 18, 19, 21 | Used by the audio driver; leave alone |
| Board ID | 12, 13 | |

**To verify in M1:**
- Which motor is left and which is right.
- Servo channels: SunFounder's code uses P0 = camera pan, P1 = camera tilt, P2 = steering (check against `picarx.py`).
- Ultrasonic pins: trigger D2 (GPIO27), echo D3 (GPIO22) (check against `picarx.py`).
- Grayscale on ADC A0–A2.

### On-board MCU (AT32F413, I2C address 0x14)
**PWM** (1-byte register address followed by a 16-bit value):

| Registers | Function |
|---|---|
| 0x20–0x2B | Pulse width, channels 0–11 |
| 0x2C / 0x2D | Pulse width, motor 2 / motor 1 (channels 12 / 13) |
| 0x40–0x43 | Prescaler, timers 0–3 |
| 0x44–0x47 | Period, timers 0–3 |

- **Timers are shared:** timer 0 runs channels 0–3, timer 1 runs 4–7, timer 2 runs 8–11, timer 3 runs 12–13 (the motors). All channels on a timer share its frequency.
- `prescaler = 72 MHz / (period + 1) / freq − 1`

**ADC:**
- Write 3 bytes `[reg, 0x00, 0x00]`, then read 2 bytes (MSB first). Values are 12-bit (0–4095). Channel 4 is the battery.

**Known doc conflicts. Resolve these against the `robot-hat` Python source (`i2c.py`, `pwm.py`, `adc.py`) before trusting either doc:**
1. ADC register numbering: one doc page says ch0 = 0x17 down to ch3 = 0x14 (battery = 0x13); another says ch0 = 0x10, ch1 = 0x11, and so on.
2. Byte order: the doc example uses SMBus `write_word_data`, which sends the low byte first, but an unofficial Rust port swaps the bytes before sending. The MCU **probably expects the high byte first.**
3. The Rust port writes `prescaler − 1`.

**Firmware fragility:** if a transfer is interrupted partway, the MCU can get out of sync. Recover by resetting it (GPIO5). This is why only one process may own the I2C bus.

**Custom firmware:** possible (Cortex-M4, `arm-none-eabi-gcc`, SWD), but there's no schematic, no published source, and no stock firmware image to restore. Treat it as a side quest.

---

## 3. Architecture

```
 Laptop (Windows/Ubuntu)                              Pi (Raspberry Pi OS, native)
┌──────────────────────────────┐   UDP + protobuf   ┌───────────────────────────────────────┐
│ Dashboard (C# / Avalonia)    │◄──────────────────►│ picard (C++ daemon, systemd)          │
│  • Xbox controller (SDL)     │  commands 20–50 Hz │  • ONLY owner of hardware             │
│  • telemetry plots + HUD     │  telemetry 10–20 Hz│  • 50 Hz SCHED_FIFO control loop      │
│  • video panel               │◄─── H.264 ─────────│  • command watchdog, safety limits    │
└──────────────────────────────┘  (MediaMTX)        │  • client table, command arbitration  │
                                                    └───────────────────────────────────────┘
 picar (C++ CLI): direct hardware mode, or --remote UDP client of picard
 Python: vision/ML experiments (M7+), same .proto files
 Future: ROS 2 in Docker → picard_bridge, just another UDP client
```

### Components
- **`libpicar_hal`** (C++): I2C bus, GPIO, MCU protocol, servo/motor/battery/LED/button classes. Hardware is reached through interfaces (`II2cBus`, `IGpio`) so it can be tested against mocks.
- **`libpicar_net`** (C++): UDP transport + nanopb messages. The transport sits behind an interface (`send(channel, bytes)` / `on_receive`) so ZeroMQ could be added later.
- **`picard`**: the daemon. It's the only process that touches the hardware. It:
  - runs the control loop, watchdog, safety limits and mode/command arbitration
  - publishes telemetry
  - takes `flock(/run/picard.lock)`
- **`picar`**: the CLI, a separate executable linking both libraries.
  - **Direct mode** takes the lock and drives the hardware itself; it refuses if picard is running.
  - **`--remote` mode** is a UDP client of picard (drive, watch telemetry, scripted tests such as `picar --remote test watchdog`).
- **Dashboard**: C# Avalonia, runs on Windows and Ubuntu. Xbox controller via SDL. Video via MediaMTX.
- **Video**: `rpicam-vid` → MediaMTX (RTSP on the LAN, WebRTC for remote). Our own libcamera capture is a later side quest.

### Messaging protocol (raw UDP + protobuf)
- **One `Envelope` message**: `seq`, `timestamp`, `oneof payload`. v1 payloads: `DriveCommand`, `Telemetry`, `Hello`, `ModeRequest`.
- **Send state, not events.** Clients send their full desired state (mode, throttle, steering, pan, tilt) at 20–50 Hz, so a lost packet doesn't matter: the next one replaces it. picard broadcasts full telemetry at 10–20 Hz.
- **Drop out-of-order packets** by `seq`. Keep each datagram **under 1200 bytes**. Large data (maps, files) goes over HTTP or in chunks.
- **Client table**:
  - Any packet from a client refreshes its entry. Listen-only clients send a `Hello` about once a second.
  - Entries expire after ~2 s.
  - Telemetry goes to each live client by **unicast** (not broadcast or multicast, which Wi-Fi handles poorly).
- **Command arbitration**: safety > manual (gamepad) > autonomy. Touching the gamepad always takes over.
- **Keep two concepts separate**: client presence decides who receives telemetry; the command watchdog decides whether the motors are allowed to run. Telemetry traffic never keeps the motors alive.
- **Serialization**: nanopb (allocation-free C) in picard; Google.Protobuf in C# and Python. One set of `.proto` files in `proto/` is the source of truth.

### Real-time discipline (picard)
- **Allocate at init, never in the control loop.**
  - Use `std::array`, vectors reserved once, ETL fixed-capacity containers (`etl::vector<T,N>`, `etl::circular_buffer`), or `std::pmr` over static buffers.
  - Avoid hidden allocations in hot paths: `std::function` captures, `std::string` building, exceptions, iostreams.
- `mlockall(MCL_CURRENT | MCL_FUTURE)`, and touch the stack and buffers once at startup. systemd needs `LimitMEMLOCK=infinity`.
- The control thread runs `SCHED_FIFO` and sleeps with `clock_nanosleep(CLOCK_MONOTONIC, TIMER_ABSTIME)` for a 50 Hz period. systemd needs `LimitRTPRIO` or `CAP_SYS_NICE`.
- Control and network threads exchange data through a lock-free SPSC queue (`etl::queue_spsc_atomic` or hand-rolled).
- Debug builds: `operator new` aborts if it's called from the control thread. Use `heaptrack` for profiling.
- Measure jitter with `cyclictest` (`rt-tests`). The stock kernel is fine for 50 Hz; PREEMPT_RT is optional later.

### Safety layers
| Layer | Catches | Action |
|---|---|---|
| picard command watchdog | client gone or Wi-Fi drop | motors to 0 within 250 ms (500 ms in the remote profile) |
| systemd service watchdog (`WatchdogSec` + `sd_notify`) | picard hung or crashed | `ExecStopPost` runs `picar stop`, then restarts picard |
| Pi hardware watchdog (`dtparam=watchdog=on`, `RuntimeWatchdogSec=15`) | kernel hang | reboots the Pi. **Does NOT stop the motors** |
| MCU-side watchdog (future custom AT32 firmware) | everything above | MCU zeroes the motors if I2C goes quiet for ~300 ms |

**Known gap:** the MCU keeps outputting its last PWM values if the Pi hangs. Until custom firmware fixes that, test new code with the wheels off the ground, and early driving at low speed in open space.

### Future: ROS 2 integration
- Raspberry Pi OS (Debian) is Tier 3 for ROS 2, so run **ROS 2 Jazzy in an Ubuntu Docker container** (`--net host --ipc host`), only once we need SLAM or Nav2.
- **`picard_bridge`** (rclcpp) is a normal UDP client:
  - telemetry → `/odom`, `/imu`, `/range`, …
  - `/cmd_vel` → `DriveCommand{source=AUTONOMY}`
  - pose, path and map metadata go back to the dashboard in our protocol
- Hardware and the camera stay native on the host. **The Ubuntu container can't use the Pi camera without building Raspberry Pi's own libcamera fork.**
- `foxglove_bridge` gives full ROS visualization in a browser. The laptop runs ROS natively (Ubuntu 24.04) for RViz.

### Remote access (outside the LAN)
- **Tailscale** (WireGuard VPN overlay) on the Pi and laptop. The desktop app works unchanged. **Never port-forward picard: the protocol has no authentication.**
- A remote profile: longer watchdog timeout, speed cap, link latency shown in the HUD. WebRTC video via MediaMTX.
- Optional later: a web/phone client through a `picard-web` gateway (WebSocket ↔ UDP), with the browser Gamepad API and WebRTC video.

---

## 4. Decisions

| Area | Decision |
|---|---|
| Approach | Own stack first (picard + own messaging); ROS 2 in Docker later, only if needed |
| OS | Stay on Raspberry Pi OS Bookworm 64-bit |
| Languages | C++20 (picard, picar, HAL); C# Avalonia (dashboard); Python (vision/ML from M7) |
| Build | CMake + GCC, built natively on the Pi; cross-compile from Ubuntu later if builds get slow |
| Tests | GoogleTest; HAL behind interfaces, so tests run on the laptop and in CI |
| Messaging | Raw UDP + protobuf (nanopb in C++) |
| Video | `rpicam-vid` + MediaMTX first; own libcamera capture later |
| Gamepad | Xbox controller on the laptop via SDL, sent over UDP |
| SunFounder code | Reference only. Disable its autostart services; nothing of theirs touches I2C at runtime |
| Python client | Not needed early; `picar --remote` covers testing until Python is needed for vision |
| Remote access | Tailscale + desktop app; web client optional |
| Repo hosting | **Public** GitHub repo, so branch protection on `main` is free |

---

## 5. Repository layout

```
picarx/
  proto/              # .proto files (single source of truth)
  libs/hal/           # libpicar_hal
  libs/net/           # libpicar_net
  picard/             # daemon: control/, safety/, clients/, main.cpp, picard.service
  tools/picar-cli/    # picar CLI
  dashboard/          # C# Avalonia app
  py/                 # vision/ML experiments (later)
  docs/               # PLAN.md, hardware.md, protocol.md, decisions/ (one note per decision)
  .github/            # CI workflows, PR template, issue templates
  CLAUDE.md           # instructions and context for Claude Code
```

---

## 6. Workflow

- **GitHub, public repo. GitHub flow**:
  - `main` protected by branch protection (or a ruleset): require a pull request, require CI to pass, no force-pushes. Allow yourself to bypass only for emergencies.
  - one short-lived branch per task (`m1/i2c-bus`)
  - **one PR per task, aiming for under ~300 changed lines**, with small logical commits inside it (Conventional Commits: `feat(hal): add I2C bus wrapper`)
- **Because the repo is public, never commit secrets**: Wi-Fi credentials, Tailscale auth keys, tokens. Keep them in untracked local config (listed in `.gitignore`).
- **CI (GitHub Actions)**: build and unit tests (against mocks) on every PR. Hardware testing is manual on the Pi, recorded in a PR-template checklist ("tested on car, wheels off the ground").
- **Tracking**: GitHub Issues + one GitHub Milestone per milestone + a Projects board. `Fixes #N` in a PR closes the issue. No Jira.
- **With Claude Code**:
  - Runs on the Pi (native builds and hardware tests); a second copy runs on the laptop for the C# dashboard.
  - One issue per session ("Implement #12"). **Plan first** (plan mode), and approve the plan before any code is written.
  - Claude creates the branch, commits in small steps and opens the PR with `gh pr create`. Review on GitHub.
  - Optionally, have Claude review the PR in a fresh session first.
  - `.claude/settings.json` permissions: anything that moves the car (`picar drive`, starting or restarting picard) **always asks for approval**; deny `git push` to `main`.
  - `CLAUDE.md` holds the architecture, conventions (no allocation in loops, single hardware owner, protocol rules), hardware facts and safety rules.

---

## 7. Milestones

Each milestone ends with a demo. The dashboard grows with every milestone after M3.

| # | Milestone | Key tasks | Done when |
|---|---|---|---|
| **M0** | Foundation | Repo, CMake, GoogleTest, formatter config, CI; branch protection on `main`; disable SunFounder autostart services; logging; picard systemd unit (hello); enable the hardware watchdog | Builds on the Pi; CI green; `systemctl status picard` shows it running |
| **M1** | HAL + `picar` CLI | I2C wrapper + MCU reset; **confirm the MCU protocol against the robot-hat source**; PWM/timer setup; servos with a calibration file; motors (direction + PWM) with differential turning; battery, LED, debounced buttons | `picar steer -10`, `picar drive 0.3 --for 1s`, `picar pan 30`, `picar battery` work (car on a stand) |
| **M2** | picard + protocol v1 | `.proto` v1; UDP transport; SCHED_FIFO control thread + network thread + SPSC queue; client table; command watchdog; systemd service watchdog; lock file; `picar --remote` | Drive through picard with `picar --remote`; killing the client stops the motors within 250 ms (`picar --remote test watchdog`) |
| **M3** | Dashboard v1 | Avalonia app: connection status, live telemetry plots, keyboard + Xbox driving | Drive from the laptop while watching live data |
| **M4** | Sensors | Ultrasonic using libgpiod edge timestamps; grayscale ×3; both in telemetry | Values show live in the dashboard |
| **M5** | Video + teleop polish | MediaMTX stream; video panel; HUD overlay (battery, distance, mode) | Drive around the house using only the video feed |
| *M5b* | *Remote access (optional)* | *Tailscale; remote profile; WebRTC video* | *Drive from outside the LAN* |
| **M6** | Reactive autonomy | Modes MANUAL/ASSIST/AUTO; arbitration; obstacle stop; cliff detection; grayscale line following with PID | Laps a tape loop and stops for a hand |
| **M7** | Recording + vision | Recorder (telemetry, commands, frames); Python bindings; lane following prototyped in Python/OpenCV, then moved to C++; optional learned steering trained on teleop recordings | Camera-only lap of a lane track |
| **M8** | Odometry hardware | SPI IMU (ICM-42688-P, interrupt on GPIO6); wheel encoders; EKF (Eigen); heading hold | A 1 m square ends within a few cm of the start |
| **M9** | Localization + mapping | AprilTag localization → lidar on UART + BreezySLAM, or ROS 2 container + slam_toolbox/Nav2 | Map of a room; navigate to a goal |

**Side quests (any time):**
- Own libcamera capture and encoding
- Custom AT32 firmware: MCU-side watchdog, encoder counting, closed-loop speed control, more robust I2C protocol with a checksum
- Linux kernel driver for the HAT (C practice)
- SPI display (status or a face)
- Web/phone client

**Why this order:**
- The watchdog (M2) exists before the car first drives over Wi-Fi.
- The dashboard (M3) comes early so every later feature can be seen.
- Teleop and recording come before autonomy, because they're the debugging and data-collection tools for it.

---

## 8. Open items and risks
- Resolve the MCU protocol conflicts (ADC numbering, byte order, prescaler offset). This is the first task in M1.
- Confirm the left/right motor mapping, servo channels and ultrasonic pins.
- The shared 5 V / 3 A power budget limits added hardware. Watch for brownouts.
- The MCU keeps its last PWM output if the Pi hangs (see Safety layers).
- Servo calibration constants differ from car to car. Keep them in a per-car config file.

## 9. References
- Robot HAT V4 docs: https://docs.sunfounder.com/projects/robot-hat-v4/en/latest/
- robot-hat Python library (the real protocol reference): https://github.com/sunfounder/robot-hat
- PiCar-X docs: https://docs.sunfounder.com/projects/picar-x-v20/en/latest/
- Custom C++/ROS 2 PiCar-X driver (lessons on servo constants and differential turning): https://jchisholm204.github.io/posts/picarx
- ROS 2 on Raspberry Pi: https://docs.ros.org/en/jazzy/How-To-Guides/Installing-on-Raspberry-Pi.html
- BreezySLAM: https://github.com/simondlevy/BreezySLAM
- PythonRobotics (reference algorithms): https://github.com/AtsushiSakai/PythonRobotics
