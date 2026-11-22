## Summary

<!-- What does this PR do and why? -->

Fixes #
Milestone: <!-- M0–M9 / side quest -->

## Changes

-

## Testing

- [ ] CI green (build + unit tests against mocks)
- [ ] Unit tests added or updated for new behavior

## Hardware test checklist

<!-- Check only what was actually done. Leave unchecked rather than guess. -->

- [ ] **N/A**: no hardware-facing change (skip the rest of this section)
- [ ] Single I2C owner confirmed (picard stopped for direct mode, or test ran through picard)
- [ ] Battery before test: ___ V; `vcgencmd get_throttled` after test: `0x___`
- [ ] **Tested on the car, wheels off the ground** (on a stand)
- [ ] Floor test at low speed in open space (only if motion behavior changed)
- [ ] Stop path verified: `picar stop` works; killing the client stops the motors within 250 ms (`picar --remote test watchdog`)
- [ ] Servo limits respected; calibration file unchanged, or updated and committed
- [ ] No brownout, Pi reboot or unexpected motion during the test

Observations:

<!-- What you saw: directions, speeds, timings, anything surprising. -->

## Review checklist

- [ ] Conventional Commits; ~300 changed lines or fewer (or justified below)
- [ ] No secrets (Wi-Fi, Tailscale keys, tokens)
- [ ] No allocation, exceptions, iostreams or `std::function` captures in the control loop
- [ ] `.proto` changes are compatible, and all clients (C++, C#, Python) are updated
- [ ] Docs updated where relevant (`docs/hardware.md`, `docs/protocol.md`, `docs/decisions/`, `CLAUDE.md`)
