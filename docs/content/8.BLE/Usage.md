## BLE

WiCAN exposes a BLE GATT service for CAN data exchange and device queries. BLE must be enabled on the configuration page before use.

**Note: When BLE is connected, the WiFi configuration access point is disabled. It turns back on when BLE disconnects.**

### GATT Service

| UUID | Description |
|------|-------------|
| 0xFFF0 | Primary service |
| 0xFFF1 | Data characteristic (read/write/notify) |

All CAN traffic and command responses use the FFF1 characteristic. Subscribe to FFF1 notifications to receive data.

### Pairing

BLE connections require secure pairing. The passkey is configured on the device settings page. Writes to any characteristic are rejected until pairing completes.

### Protocol Modes

BLE works with whichever protocol is selected in the CAN configuration:

- **ELM327**: Send AT commands and OBD-II requests as text. Responses arrive as notifications on FFF1.
- **slcan**: Send slcan frames as text.
- **RealDash**: See [RealDash BLE configuration](../4.RealDash/Usage.md).
- **AutoPID**: CAN traffic is handled by the AutoPID engine. Use the JSON command interface below to read cached PID values.

### JSON Command Interface

When the protocol is set to AutoPID, BLE clients can query device state by writing JSON commands to FFF1. Responses are sent back as FFF1 notifications.

Commands are JSON objects with a `cmd` field. Responses may span multiple BLE notifications depending on payload size and MTU. A newline character (`\n`) marks the end of each response, so the client should accumulate notification payloads until `\n` is received.

#### Commands

**Get AutoPID data**

Returns all cached PID values.

- Write: `{"cmd":"get_autopid_data"}`
- Response: JSON object with current PID values, e.g.:
```json
{"VehicleSpeed": 23.0, "EngineSpeed": 1165.0, "BatterySoC": 72.5}
```

**Get ECU status**

- Write: `{"cmd":"get_ecu_status"}`
- Response: `{"ecu_online":true}` or `{"ecu_online":false}`

**Get AutoPID configuration**

Returns parameter names, units, and device classes.

- Write: `{"cmd":"get_config"}`
- Response:
```json
{"VehicleSpeed":{"class":"speed","unit":"km/h"},"BatterySoC":{"class":"battery","unit":"%"}}
```

**Get destination statistics**

Returns success/fail counts per configured destination (MQTT, webhook, etc.).

- Write: `{"cmd":"get_dest_stats"}`
- Response:
```json
{"destinations":[{"success":142,"fail":0},{"success":140,"fail":2}]}
```

**Set RTC time**

Sets the hardware RTC clock. Values are BCD-encoded, matching the WebSocket `set_rtc_time` command format. After setting, the ESP32 system clock is synced to the RTC.

- Write: `{"cmd":"set_rtc_time","hour":32,"min":48,"sec":0,"year":38,"month":3,"day":49,"weekday":1}`
- Response: `{"rtc_set":true}` or `{"rtc_set":false}` on I2C error
- Missing or non-numeric fields: `{"error":"missing fields"}`

BCD encoding: decimal 20 = BCD 0x20 = 32, decimal 30 = BCD 0x30 = 48. The year is offset from 2000 (year 38 = BCD 0x26 = 2026).

**Get RTC time**

Reads the current hardware RTC clock. Values are BCD-encoded.

- Write: `{"cmd":"get_rtc_time"}`
- Response: `{"hour":32,"min":48,"sec":0,"year":38,"month":3,"day":49,"weekday":1}`

**ELM327 passthrough**

Sends a raw ELM327/AT command and returns the response. The command is executed between AutoPID polling cycles — AutoPID is not paused or disrupted. ELM327 settings are restored after each command so AutoPID resumes cleanly. The trailing `\r` is added automatically if omitted.

- Write: `{"cmd":"elm327","data":"09 02"}`
- Response:
```json
{"response":"49 02 01 31 48 47 ...\r\r>"}
```

The device waits up to 6 seconds for AutoPID to yield ELM327 access and another 6 seconds for the ELM327 response. If AutoPID does not yield in time: `{"error":"autopid busy"}`. If the ELM327 does not respond: `{"error":"no response"}`.

If no data is available, or the `cmd` value is not recognized, the response is `{"error":"no data"}`.

Non-JSON writes (AT commands, slcan frames) are passed through to the CAN stack as before. Malformed JSON (missing `cmd` field, invalid syntax) is also passed through. The JSON command interface does not interfere with other protocol modes.

### MTU and Chunking

BLE MTU is negotiated per connection (typically 20-490 bytes). Large responses are split into MTU-sized chunks sent as sequential notifications. The `\n` terminator signals the final chunk.

### Testing with nRF Connect

1. Connect to the WiCAN device and pair.
2. Expand service 0xFFF0.
3. Enable notifications on FFF1 (tap the triple-arrow icon).
4. Write `{"cmd":"get_autopid_data"}` to FFF1 (select UTF-8 text input).
5. Observe the JSON response in the notification log.
