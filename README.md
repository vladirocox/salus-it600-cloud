# Salus iT600 Cloud Integration for Home Assistant

[![Upstream PR](https://img.shields.io/badge/Upstream%20PR-%2311-blue)](https://github.com/Peterka35/salus-it600-cloud/pull/11)
[![hacs_badge](https://img.shields.io/badge/HACS-Custom-orange.svg)](https://github.com/custom-components/hacs)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

> **Contribution fork** of [Peterka35/salus-it600-cloud](https://github.com/Peterka35/salus-it600-cloud).
> The IT800TX fixes are proposed upstream in [PR #11](https://github.com/Peterka35/salus-it600-cloud/pull/11).
> That PR ports them onto upstream v1.1.0 and keeps devices reporting the `ep9` properties working.
> This fork's `main` still contains the original IT800TX implementation.

## What This Fork Fixed for the IT800TX

The IT800TX reports its shadow under `ep1:sTherS:*`, `ep1:sComm:*`, `ep2:sOnOffS:*` and `ep1:sPowerS:*`.
The pre-1.1.0 integration read and wrote `ep9:sIT600TH:*`, so an IT800TX showed no temperature or preset and
commands were published to properties the device never applied.

| Issue | Original | This Fork |
|-------|----------|-----------|
| ❌ Thermostat controls not working | Stub `async_set_hvac_mode` that raised `NotImplementedError` | ✅ Working OFF/HEAT via `ep1:sTherS:SetSystemMode` |
| ❌ Shadow property names wrong for the IT800TX | Used `ep9:sIT600TH:*` — the device never reported these | ✅ Corrected to `ep1:sTherS:*` / `ep1:sComm:*` / `ep2:sOnOffS:*` / `ep1:sPowerS:*` |
| ❌ Current temp always `None` | Wrong key `ep9:sIT600TH:LocalTemperature_x100` | ✅ Reads `ep1:sTherS:LocalTemperature_x100` |
| ❌ Target temp always `None` | Wrong key `ep9:sIT600TH:HeatingSetpoint_x100` | ✅ Reads `ep1:sTherS:HeatingSetpoint_x100` |
| ❌ Switch state reading broken | Wrong key `ep9:sOnOffS:OnOff` | ✅ Reads `ep2:sOnOffS:OnOff` |
| ⚠️ State update delay | 3 s HA debounce + 30 s poll cycle | ✅ `async_force_refresh()` + per-device MQTT shadow subscription |

## Features

✅ **Full device control** — Set temperature, change modes, control switches
✅ **Real-time MQTT updates** — Subscribes to each device's `$aws/things/{device_code}/shadow/update/documents`
✅ **Three preset modes** — Schedule, Manual, Away/Frost protection
✅ **OneTouch rules** — Trigger predefined automation rules
✅ **Multiple gateways** — Support for multiple Salus gateways
✅ **Battery monitoring** — Track battery levels on wireless devices

## Installation (this fork)

Add custom repository in HACS: `https://github.com/vladirocox/salus-it600-cloud`

Or install manually:
```bash
cd /config/custom_components
git clone https://github.com/vladirocox/salus-it600-cloud.git
# Restart Home Assistant
```

## Upstream Contribution

The upstream PR is [#11](https://github.com/Peterka35/salus-it600-cloud/pull/11). It contains the IT800TX-specific
changes only:

- per-device property maps instead of hardcoded `ep1` names, so devices reporting `ep9` keep working;
- HVAC mode through `SetSystemMode` for iT800 devices, with the matching hold type for presets;
- tests, documentation and changelog.

The earlier [PR #1](https://github.com/vladirocox/salus-it600-cloud/pull/1) was merged inside this fork only and
never reached upstream. The other changes in this fork (per-device MQTT subscriptions, connection handling,
availability) are already part of upstream v1.1.0 or were specific to the pre-1.1.0 code, so they are not duplicated
in PR #11.

## Contributing

Contributions are welcome. Please open an issue or pull request; upstream fixes should go to
[Peterka35/salus-it600-cloud](https://github.com/Peterka35/salus-it600-cloud).

## License

MIT License
