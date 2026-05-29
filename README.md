# Salus iT600 Cloud Integration for Home Assistant

[![PR: Fixes + HVAC Control](https://img.shields.io/badge/PR-Fixes%20%2B%20HVAC%20Control-blue)](https://github.com/vladirocox/salus-it600-cloud/pull/1)
[![hacs_badge](https://img.shields.io/badge/HACS-Custom-orange.svg)](https://github.com/custom-components/hacs)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

> **This is a contribution fork** of [Peterka35/salus-it600-cloud](https://github.com/Peterka35/salus-it600-cloud) with critical bug fixes and new features. Pull request: [#1](https://github.com/vladirocox/salus-it600-cloud/pull/1)

## What This Fork Fixes & Adds

| Issue | Original | This Fork |
|-------|----------|-----------|
| ❌ Thermostat controls not working | Stub `async_set_hvac_mode` that raised `NotImplementedError` | ✅ Working OFF/HEAT via `ep1:sTherS:SetSystemMode` |
| ❌ All shadow property names wrong | Used `ep9:sIT600TH:*` — device never matched these | ✅ Corrected to `ep1:sTherS:*` / `ep1:sComm:*` / `ep2:sOnOffS:*` / `ep1:sPowerS:*` |
| ❌ Current temp always `None` | Wrong key `ep9:sIT600TH:LocalTemperature_x100` | ✅ Reads `ep1:sTherS:LocalTemperature_x100` |
| ❌ Target temp always `None` | Wrong key `ep9:sIT600TH:HeatingSetpoint_x100` | ✅ Reads `ep1:sTherS:HeatingSetpoint_x100` |
| ❌ Switch state reading broken | Wrong key `ep9:sOnOffS:OnOff` | ✅ Reads `ep2:sOnOffS:OnOff` |
| ⚠️ State update delay | 3s HA debounce + 30s poll cycle | ✅ `async_force_refresh()` + MQTT shadow document subscription for near-instant updates |

## Features

✅ **Full device control** — Set temperature, change modes, control switches
✅ **Real-time MQTT updates** — Subscribes to `$aws/things/+/shadow/update/documents` for instant state changes
✅ **Three preset modes** — Schedule, Manual, Away/Frost protection
✅ **OneTouch rules** — Trigger predefined automation rules
✅ **Multiple gateways** — Support for multiple Salus gateways
✅ **Battery monitoring** — Track battery levels on wireless devices

## Installation

Add custom repository in HACS: `https://github.com/vladirocox/salus-it600-cloud`

Or install manually:
```bash
cd /config/custom_components
git clone https://github.com/vladirocox/salus-it600-cloud.git
# Restart Home Assistant
```

## Contributing

Changes submitted as PR [#1](https://github.com/vladirocox/salus-it600-cloud/pull/1) to upstream. All contributions welcome!

## License

MIT License
