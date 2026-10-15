# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.1.0] - 2026-10-15

### Fixed
- 🐛 **Critical: wildcard MQTT subscription broke all control**
  - The integration subscribed to `$aws/things/+/shadow/update/documents` on connect.
    AWS IoT's policy only grants each device access to its own shadow, so AWS accepted
    the CONNACK (rc=0) and then immediately closed the socket (~1.3s).
  - Symptoms: every `climate.set_temperature` / `set_hvac_mode` / `set_preset_mode`
    failed with `Message publish failed: The client is not currently connected`, and the
    log filled with ~1,500 `MQTT disconnected (rc=7)` warnings.
  - Now subscribes per device (`$aws/things/{device_code}/shadow/update/documents`)
    using the known device codes, so the connection stays up.
  - Verified by isolating four connection scenarios: no-subscribe, wildcard subscribe,
    exact-topic subscribe and `clean_session=False`. Only the wildcard was killed.
- 🐛 **Connection state assumed success instead of waiting for CONNACK**
  - The old code did `await asyncio.sleep(2)` and then set `_mqtt_connected = True`
    unconditionally, so a failed connect was reported as connected and every service
    call retried against a dead client.
  - Now waits for the real CONNACK with a timeout and only reports connected on rc=0.
- 🐛 **MQTT client leak / reconnect storm**
  - A new `uuid4()` client ID was minted per attempt and old clients were never
    stopped with `loop_stop()`, leaking paho network threads.
  - Now reuses one stable client ID per connection and tears down properly via
    `_dispose_mqtt_client()` on every error path.
- 🐛 **Concurrent service calls each spawned their own MQTT client**
  - Added an `asyncio.Lock` so parallel `climate.*` calls serialise their connect.
- 🐛 **API timeouts were silently swallowed**
  - `asyncio.TimeoutError` is a builtin `TimeoutError`, not an `aiohttp.ClientError`,
    so it bypassed the error handlers and was caught by a broad `except Exception`,
    logging with an empty message and returning partial/empty data.
  - Now handled explicitly and raised as `SalusCloudConnectionError`.
- 🐛 **Coordinator reported success with zero devices**
  - A failed poll returned an empty device list without raising, so entities kept
    reporting `available` while every property was `None`.
  - An empty result after previously having devices now raises `UpdateFailed`.

### Changed
- MQTT connect now refreshes AWS credentials once and retries before backing off
  exponentially (5s → 300s), instead of hammering AWS IoT.
- API request timeout raised from 10s to 30s to tolerate the Salus cloud's
  intermittent 502/504 responses.
- `climate` entities now expose `available`, reporting unavailable when the
  coordinator has no fresh data rather than silently showing unknown.
- Per-request and per-device logging downgraded from INFO to DEBUG to stop
  "logging too frequently" warnings.

## [1.0.3] - 2025-11-12

### Fixed
- 🐛 **Critical: AWS IoT credentials expiration**
  - Fixed issue where credentials expired after ~1 hour causing all control operations to fail
  - Error symptoms: "WebSocket handshake error, connection not upgraded" and MQTT rc=7 (not authorized)
  - Added `_aws_credentials_expiry` field to track credential expiration time
  - Parse expiration timestamp from AWS GetCredentialsForIdentity response
  - Automatically refresh credentials 5 minutes before expiration
  - Disconnect and reconnect MQTT client when credentials are refreshed
  - Improved error handling for MQTT disconnections during publish operations
  - Added retry logic for publish operations after reconnection

### Changed
- AWS IoT credentials now automatically renewed before expiration
- MQTT client properly disconnects before credential refresh
- Better connection state management (check both flag and actual connection)

### Technical Details
- Credentials from Cognito Identity Pool expire after ~1 hour
- `_ensure_mqtt_connected()` now checks expiration before every connection
- Fallback expiration: 1 hour if timestamp not provided by AWS
- This ensures uninterrupted device control functionality 24/7

## [1.0.2] - 2025-10-31

### Fixed
- 🐛 **HVAC action state detection**
  - Fixed typo in `hvac_action` property causing incorrect state display
  - Changed `ep9:sIT600_TH:RunningState` to `ep9:sIT600TH:RunningState`
  - Now correctly shows "heating" or "idle" based on actual thermostat state
  - Running state attribute was showing correctly, but hvac_action wasn't
- 📝 **Logging improvements**
  - Changed all INFO logs to DEBUG level
  - Reduced log verbosity during normal operation
  - Only WARNING and ERROR messages shown by default
  - Debug information (API requests, MQTT operations, device states) now at DEBUG level
  - Applied to gateway.py, climate.py, config_flow.py, and button.py

### Changed
- Cleaner logs with less noise during normal operation
- Better log level classification (INFO → DEBUG for routine operations)

## [1.0.1] - 2025-10-31

### Fixed
- 🐛 **MQTT WebSocket connection to AWS IoT**
  - Fixed "WebSocket handshake error, connection not upgraded"
  - Added proper connection synchronization using `asyncio.Event`
  - Improved connection timeout handling (10 seconds)
  - Added connection result verification (rc code check)
  - Better TLS/SSL configuration with cert verification
- 🔧 **Device index detection**
  - Fixed hardcoded device index "11" issue
  - Device index now automatically detected from shadow data
  - Stored as `_shadow_device_index` during device initialization
  - Fallback to "11" if detection fails
- 📝 **Debug logging improvements**
  - Extensive MQTT operation logging
  - Device index information in logs
  - WebSocket URL details (first 100 chars)
  - Error tracebacks for better debugging
  - Proper log level mapping for paho-mqtt

### Changed
- Gateway functions now accept optional `device_index` parameter:
  - `set_temperature(device_code, temperature, device_index=None)`
  - `set_hold_mode(device_code, mode, device_index=None)`
  - `set_switch_state(device_code, state, device_index=None)`
- MQTT connection now waits for actual connection before proceeding
- Better error messages with more context

### Technical Details
- Added `_mqtt_connect_event` and `_mqtt_connect_rc` for connection tracking
- Improved `_ensure_mqtt_connected()` with proper async/sync coordination
- Enhanced `update_device_shadow()` with automatic device index discovery
- Better exception handling with full tracebacks

## [1.0.0] - 2025-01-XX

### Added
- ✨ **Full device control** via AWS IoT MQTT
  - Set thermostat target temperature
  - Change thermostat preset modes (Schedule/Manual/Away)
  - Control switches and relays (turn on/off)
  - Trigger OneTouch automation rules
- 🌡️ **Climate entities** with full functionality
  - Current and target temperature display
  - Three preset modes: Schedule, Manual, Away/Frost
  - HVAC action display (heating/idle/off)
  - Extra state attributes (hold_type, system_mode, battery_voltage, running_state)
- 🔌 **Switch entities** with full control
  - RS600, SPE600, SR600 models
  - Turn on/off functionality
  - State feedback
- 📊 **Sensor entities**
  - Temperature sensors
  - Humidity sensors
  - Battery voltage sensors
- 🚪 **Binary sensor entities**
  - Door/window sensors (WLS models)
  - Motion and occupancy sensors
- 🎯 **Button entities** for OneTouch rules
  - Trigger predefined automation rules
  - Auto-filter system rules
  - Show only user-created rules
- 🔐 **AWS Cognito authentication**
  - SRP (Secure Remote Password) authentication
  - Automatic token refresh
  - Session management
- 📡 **AWS IoT MQTT** communication
  - Real-time device shadow updates
  - WebSocket connection over TLS
  - Automatic reconnection
- 🏷️ **Gateway-specific entity naming**
  - Entity IDs include gateway name
  - Prevents conflicts with multiple gateways
  - Format: `{gateway_slug}_{device_slug}`
- ⚙️ **Configuration flow**
  - User-friendly setup via UI
  - Email/password authentication
  - Automatic device discovery
- 🔄 **Data coordinator**
  - 30-second polling interval
  - Efficient state updates
  - Error handling and retry logic
- 📝 **Comprehensive logging**
  - Debug mode support
  - MQTT connection logging
  - API request/response logging

### Technical Details
- **Authentication**: AWS Cognito with pycognito library
- **MQTT Client**: paho-mqtt for AWS IoT
- **API Base**: service-api.eu.premium.salusconnect.io
- **Region**: EU (eu-central-1)
- **Polling**: 30 seconds
- **Device Shadow**: AWS IoT standard pattern

### Known Limitations
- Only EU region supported (eu-central-1)
- Requires internet connection (cloud-only)
- No local API support
- 30-second polling delay for non-MQTT updates

### Requirements
- Home Assistant 2023.1+
- Python 3.11+
- Active Salus Smart Home account
- Internet connection

## [0.1.0] - 2024-XX-XX (Development)

### Added
- Initial development version
- Read-only implementation
- Basic authentication
- Device discovery
- Entity creation (read-only)

---

## Planned Features

### Future Releases

#### v1.1.0
- [ ] WebSocket support for real-time updates
- [ ] Reduce polling interval when WebSocket active
- [ ] Add diagnostic sensors (signal strength, last seen)
- [ ] Improve error messages and recovery

#### v1.2.0
- [ ] Schedule/program viewing
- [ ] Schedule editing support
- [ ] Manual schedule override
- [ ] Vacation mode support

#### v1.3.0
- [ ] US region support
- [ ] Multi-region account support
- [ ] Automatic region detection

#### v2.0.0
- [ ] Local API support (if available)
- [ ] Hybrid local/cloud mode
- [ ] Offline operation support

---

## Migration Guides

### Upgrading from pre-1.0 (read-only) versions

**Entity ID Changes**:
Old versions used simple entity IDs like `climate.termostat_pokoj`.
Version 1.0+ uses gateway-specific IDs like `climate.apartman_245_5_termostat_pokoj`.

**Steps**:
1. Note down your current entity IDs
2. Update the integration
3. Remove and re-add the integration
4. Update automations and scripts with new entity IDs

**Unique ID Format**:
- Old: `salus_it600_cloud_{device_id}`
- New: `salus_it600_cloud_{gateway_id}_{device_id}`

This allows multiple gateways without conflicts.
