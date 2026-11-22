# CLAUDE.md

Context and rules for Claude Code working in this repo. **`docs/PLAN.md` is the source of truth**; this file condenses what you need every session. If the two disagree, PLAN.md wins — then fix this file.

## Project

A SunFounder PiCar-X turned into an autonomy learning platform, one demoable milestone at a time (M0 foundation → M9 mapping; see PLAN.md §7).

- **Current status:** see open GitHub milestones and issues (`gh api repos/{owner}/{repo}/milestones`, `gh issue list`); don't infer it from this file. Planned layout is in PLAN.md §5 (`proto/`, `libs/hal/`, `libs/net/`, `picard/`, `tools/picar-cli/`, `dashboard/`, `py/`, `docs/`).
- **Languages:** C++20 (HAL, daemon, CLI), C# Avalonia (dashboard, runs on the laptop), Python (vision/ML from M7).
- **This copy of Claude runs on the Pi itself** (`creampi`), so commands here can touch real hardware. A second copy runs on the laptop for the dashboard.
- This file covers C++ and hardware. C#/Avalonia conventions will go in `dashboard/CLAUDE.md` when dashboard work starts (Claude Code reads it when working in that directory).

## Architecture

```
Laptop: Dashboard (C#) ──UDP+protobuf──► Pi: picard (C++ daemon, systemd) ──I2C/GPIO──► Robot HAT
                       ◄── telemetry ───        ▲
                       ◄── H.264 (MediaMTX) ─   │ UDP
                                       picar --remote / Python / future picard_bridge (ROS 2)
```

- **`libpicar_hal`** — I2C bus, GPIO, MCU protocol, servo/motor/battery/LED/button. Hardware only through interfaces (`II2cBus`, `IGpio`) so everything is testable against mocks.
- **`libpicar_net`** — UDP transport + nanopb messages behind `send(channel, bytes)` / `on_receive`.
- **`picard`** — the daemon and the **only process that touches hardware**. Holds `flock(/run/picard.lock)`. Runs the 50 Hz control loop, command watchdog, safety limits, client table, arbitration, telemetry.
- **`picar`** — CLI. *Direct mode* takes the lock and drives hardware itself (refuses if picard is running). *`--remote` mode* is a UDP client of picard (`picar --remote test watchdog`, etc.).
- **Video** — `rpicam-vid` → MediaMTX (RTSP on LAN, WebRTC remote).
- **Remote access** — Tailscale only. **Never port-forward picard; the protocol has no authentication.**

## Build & test (planned — lands in M0)

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build -j4
ctest --test-dir build --output-on-failure
```

- Unit tests use GoogleTest against mocks and must run on a laptop and in CI with no hardware.
- **Hardware tests are never registered with ctest.** They run only through explicit `picar` commands (e.g. `picar --remote test watchdog`), with user approval.
- Built natively on the Pi with GCC; Pi 4 has 4 GB — keep `-j4` or lower, and drop to `-j2` if the build gets killed for running out of memory (protobuf and template-heavy code are the usual cause).

## Code conventions

**C++ / real-time (picard control thread):**
- Allocate at init, never in the control loop. Use `std::array`, vectors reserved once, ETL fixed-capacity containers, or `std::pmr` over static buffers.
- No hidden allocation in hot paths: no `std::function` captures, `std::string` building, exceptions, or iostreams.
- `mlockall(MCL_CURRENT | MCL_FUTURE)` at startup and pre-touch stack/buffers.
- Control thread: `SCHED_FIFO`, `clock_nanosleep(CLOCK_MONOTONIC, TIMER_ABSTIME)` at 50 Hz.
- Control ↔ network threads talk only through a lock-free SPSC queue.
- Debug builds abort on `operator new` from the control thread.

**Protocol:**
- `.proto` files in `proto/` are the single source of truth for C++ (nanopb), C# and Python.
- One `Envelope { seq, timestamp, oneof payload }`. v1 payloads: `DriveCommand`, `Telemetry`, `Hello`, `ModeRequest`.
- **Send state, not events** — clients send full desired state at 20–50 Hz; picard sends full telemetry at 10–20 Hz.
- Drop out-of-order packets by `seq`. Datagrams **< 1200 bytes**. Large data goes over HTTP or chunked.
- Telemetry is **unicast** to each live client (no broadcast/multicast).
- Client presence (who gets telemetry, ~2 s expiry) and the command watchdog (whether motors may run) are **separate**. Telemetry traffic never keeps the motors alive.
- Arbitration: **safety > manual (gamepad) > autonomy**. Touching the gamepad always takes over.

**General:**
- Per-car calibration (servo trims, limits) lives in a config file, never hard-coded.
- **`picar` direct mode must be safe by construction**, not by Claude remembering flags: every actuator command has a default maximum duration (e.g. 2 s) and clamps throttle to a configurable cap; all motors are zeroed on normal exit, SIGINT/SIGTERM, or any error (RAII guard plus a signal handler).
- SunFounder code is reference only. Nothing of theirs touches I2C at runtime.
- Record decisions as one note per decision in `docs/decisions/`; hardware findings in `docs/hardware.md`; protocol in `docs/protocol.md`.

**TBD in M0 — don't decide these silently.** If a task needs one before it's settled, raise it with the user and record the answer in `docs/decisions/`:
- clang-format style and naming conventions
- whether exceptions are allowed outside the control thread
- the logging library
- error-handling style (`std::expected` vs. error codes)

## Git & workflow

- **Default branch is `master`.** Never push to it; never merge PRs — the user reviews and merges on GitHub.
- One issue per session ("Implement #12"). **Plan mode first**; no code until the plan is approved.
- One short-lived branch per task: `m<N>/<task>` (e.g. `m1/i2c-bus`). Always push with an explicit branch name: `git push -u origin m1/i2c-bus` (bare `git push` and `git push origin HEAD` are denied in settings).
- One PR per task, aiming for **< ~300 changed lines**, small logical commits, **Conventional Commits** (`feat(hal): add I2C bus wrapper`). Use `Fixes #N` in the PR body. Open with `gh pr create`; fill in the PR template's hardware checklist honestly (mark N/A or leave unchecked what wasn't done).
- **Public repo: never commit secrets** (Wi-Fi credentials, Tailscale auth keys, tokens). Keep them in untracked local config listed in `.gitignore`.

## Hardware facts

Full tables in PLAN.md §2. Key facts:

- **Pi 4 (4 GB)**, hostname `creampi`, Raspberry Pi OS **Bookworm 64-bit**, **libgpiod v1** CLI tools (`gpioset`/`gpioget`, not the v2 syntax).
- **SunFounder Robot HAT V4**, MCU **AT32F413 at I2C 0x14** (bus 1, GPIO2/3).
- Car: 2× rear TT motors (**no encoders**), 3 servos (steering, cam pan, cam tilt), Pi camera, ultrasonic, 3-ch grayscale, I2S speaker.
- **Power:** one **5 V / 3 A buck** feeds the Pi, servos, motor driver and MCU. Motors run from the regulated 5 V rail. Motor inrush or a stalled servo can brown out and reboot the Pi. Check the budget before adding hardware.
- **Battery:** 2S 18650, 6.0–8.4 V, cutoff 6.0 V. `Vbat = raw/4095 × 3.3 × 3` (MCU ADC ch 4). LEDs: 2 lit > 7.6 V, 1 lit 7.15–7.6 V, 0 lit < 7.15 V. The power switch is an instant cut — `sudo shutdown -h now` first.

| Function | BCM GPIO | Notes |
|---|---|---|
| I2C SDA/SCL | 2 / 3 | MCU 0x14 |
| MCU reset | 5 | Low 10 ms, high 10 ms. **Don't touch otherwise** |
| Motor 1 / 2 direction | 23 / 24 | Speed: MCU ch 13 (reg 0x2D) / ch 12 (reg 0x2C) |
| USER LED | 26 | Active-high |
| USR / RST button | 25 / 16 | Active-low, need internal pull-up, bounce badly |
| D0–D3 | 17, 4, 27, 22 | |
| SPI | 8, 9, 10, 11 + 6 (BSY/IRQ) | Future IMU |
| UART | 14 / 15 | Future lidar |
| I2S audio | 18, 19, 21 | **Leave alone** |

**MCU protocol:**
- PWM: write 1-byte register + 16-bit value. 0x20–0x2B = pulse width ch 0–11; 0x2C/0x2D = motors; 0x40–0x43 prescaler, 0x44–0x47 period for timers 0–3.
- Timers are shared: T0 = ch 0–3, T1 = 4–7, T2 = 8–11, T3 = 12–13 (motors). Channels on one timer share a frequency.
- `prescaler = 72 MHz / (period + 1) / freq − 1`.
- ADC: write `[reg, 0x00, 0x00]`, read 2 bytes MSB-first, 12-bit.
- If a transfer is interrupted the MCU can desync; recover via the GPIO5 reset. This is why there is exactly one I2C owner.

**Unresolved — do not guess.** Resolve against the `robot-hat` Python source (`i2c.py`, `pwm.py`, `adc.py`, https://github.com/sunfounder/robot-hat) and cite file + line:
1. ADC register numbering (0x17→0x14 descending vs. 0x10 ascending).
2. 16-bit byte order (probably high byte first; SMBus `write_word_data` sends low first).
3. Whether the prescaler register takes `prescaler` or `prescaler − 1`.

**Unverified (M1):** which motor is left/right; servo channels (expected P0 = pan, P1 = tilt, P2 = steering); ultrasonic trig D2/GPIO27, echo D3/GPIO22; grayscale on ADC A0–A2; motor direction polarity (which level on GPIO23/24 means forward, and whether the two motors are mirrored). Treat these as assumptions until a hardware investigation confirms them.

## Safety rules (hard rules for Claude)

The car is real, and the MCU **keeps outputting its last PWM if the Pi hangs**. The Pi hardware watchdog reboots the Pi but does **not** stop the motors.

1. **Never move the car or drive the hardware without the user's approval at that moment.** That covers `picar` actuator commands (drive, steer, pan, tilt, test), running or (re)starting `picard`, raw `i2cset`/`i2ctransfer`/`gpioset`/`pinctrl`, any script that imports robot-hat, SunFounder's example scripts (`sudo python3 …/example/*.py`), the `robot_hat` CLI (e.g. `robot_hat reset_mcu`), `systemctl start/restart/stop` for any service, and `sudo` in general. `.claude/settings.json` forces a prompt for these. Do not try to route around it (wrappers, aliases, `bash -c`, scripts that do the same thing).
2. **Before asking, say what will move, how fast, for how long, and the precondition.** New or changed motion code: **car on a stand, wheels off the ground**. Floor tests: low speed, open space, only after a stand test passed.
3. **Always keep a stop path.** Use short durations (`--for 1s`) and low throttle (≤ 0.3) by default. Know the stop command (`picar stop`, or kill the remote client so the watchdog fires) before starting.
4. **One I2C owner.** Check `systemctl is-active picard` before any direct-mode command. Don't run `i2cdetect`/`i2cget`/`i2cdump`/`i2cset`/`i2ctransfer` on bus 1 without approval, **even when picard isn't running**: probes hit 0x14 too, and the stock firmware expects fixed frames and can desync from unexpected transfers. If the MCU stops responding afterwards, report it; recovery is the GPIO5 reset.
5. **Pins:** only touch GPIO5 with the documented reset sequence. Leave I2S (18/19/21) alone. `gpioget`/`gpiomon` reconfigure a line as input, so never run them on 23/24 (motor direction), 26 (USER LED, once picard drives it) or other driven outputs.
6. **If anything hangs or the Pi reboots mid-test, assume the motors may still be running.** Tell the user immediately; don't silently retry.
7. **Power:** check battery voltage and `vcgencmd get_throttled` before motor tests; below ~7.15 V, ask the user to charge first. The battery reading is itself an I2C read, so it follows rule 4: use `picar battery` (direct mode, after the `systemctl is-active picard` check) or picard's telemetry. Until `picar` exists, ask the user to read the battery LEDs. Never "fix" a stall or brownout by raising power or retrying harder; report it.
8. **Never weaken safety layers** (command watchdog timeout, systemd watchdog, hardware watchdog, speed caps) without an explicit decision note in `docs/decisions/` that the user approved.
9. **Never port-forward or expose picard** beyond the LAN/Tailscale.
10. Don't run Claude with `--dangerously-skip-permissions` on the Pi (bypass mode is disabled in project settings).
