---
name: prins-fixture
description: Interact with the Ampel Prins switch fixture over TCP using the prins-fixture-mcp server. Use when sending fixture commands, setting LEDs, reading switch presence/version/EEPROM, running button or buzzer tests, managing fixture status, or configuring the MCP server for fixture communication.
---

# Prins Fixture

## Overview

The Prins switch fixture is a legacy hardware test fixture communicating over TCP (ASCII protocol).
It is **not part of the test framework** — it is a standalone legacy system.
Commands are plain ASCII strings; responses start with `SD` or `STR`.

Fixture interaction is done via **prins-fixture-mcp** — a Python MCP server that exposes every command as a tool callable from Claude Code.

## MCP Server (prins-fixture-mcp)

**Location:** `SwitchTests/PrinsFixtureMCP/server.py`

**Install & run:**
```bash
pip install mcp
python PrinsFixtureMCP/server.py
```

**Register in `.claude/settings.json`:**
```json
{
  "mcpServers": {
    "prins-fixture-mcp": {
      "command": "python",
      "args": ["C:/Users/JuliandeBruin/Documents/AmpelTesting/TestDevelopment/PrinsTests/SwitchTests/PrinsFixtureMCP/server.py"]
    }
  }
}
```

Always call `connect_fixture(ip, port)` first; all other tools fail without an active connection.

## TCP Protocol

| Format | Example | Notes |
|---|---|---|
| `PutXxx...` | `PutFixtSt:001` | Write command, response discarded |
| `GetXxx...` | `GetFixtSt` | Read command, returns `SD...` response |
| `DiaXxx...` | `DiaTstBuz00` | Diagnostic command |
| Response prefix `SD` | `SDFixtSt:1` | Normal fixture response |
| Response prefix `STR` | `STRIndLoc:Updating` | Status update response |

## Available MCP Tools

### Connection
- `connect_fixture(ip, port)` — open TCP connection (3 s timeout)
- `disconnect_fixture()` — close connection

### Switch Count
- `set_number_of_switches(n)` — write `PutSwiCnt00XX`
- `get_number_of_switches()` — send `GetSwiCnt`

### LEDs (position indicators)
- `set_all_leds(color)` — all 64 slots, color: `Red/Green/Both/Off`
- `set_led(index, color)` — single slot
- `set_multiple_leds(indices, active_color, inactive_color)` — bulk set

Colors map: `Red=0 Green=1 Both=2 Off=3`

### Fixture Identity & Status
- `get_fixture_id()` — `GetFixtID`
- `get_fixture_status()` — `GetFixtSt` → `SDFixtSt:N`
- `set_fixture_status(status)` — status: `Empty/Assembly/Testing/Packaging/Service`

Status command map:
```
Empty      → PutFixtSt:000
Assembly   → PutFixtSt:001
Testing    → PutFixtSt:002
Packaging  → PutFixtSt:004
Service    → PutFixtSt:008
```

### Switch Diagnostics
- `check_switch_present(index)` — `GetPresenXX`, checks for `"1"` in response
- `check_switch_version(index)` — `GetVerNumXX`, decodes HW rev (bits 7:5) and App rev (bits 4:0); minimum OK = HW=1, App≥10

### EEPROM (PN / SN)
- `program_switch_eeprom(index, pn, sn)` — `PutSwEromXX[32 hex]`
- `get_switch_eeprom(index)` — `GetSwEromXX`, decodes to PN and SN

PN format: `XXX/XXXXXX/L` (12 chars, e.g. `600/025250/A`)
SN format: `A/WW/YY/BPPP` (12 chars, e.g. `A/26/24/1001`)

EEPROM layout (16 bytes, 32 hex chars):
```
[0]  Version (0x02)
[1-3] Reserved (0x000000)
[4]  Batch
[5]  Year
[6]  Week
[7]  Reserved
[8]  Batch number part 1
[9]  Batch number part 2
[10-13] PN numeric fields
[14] PN last digit × 10
[15] PN hw-rev letter (ASCII)
```

### Pass / Fail
- `store_pass_fail(index, passed)` — `PutPasFaiXX[0|1]`
- `get_pass_fail(index)` — `GetPasFaiXX`

### Button Test
- `initialise_button_test()` — `PutButIni`
- `get_button_result()` — `GetButRes`, returns indices where button was NOT pressed (char `'0'`)

### Buzzer Test
- `get_buzzer_amplitude(index)` — `DiaTstBuzXX`
- `test_buzzer(index)` — `GetTstBuzXX`, returns `amplitude,volume`

### LED Test Commands
- `turn_on_test_led(index)` — `PutTstLED00XX`
- `turn_on_test_rgb_led(index, color)` — `PutTstRgb00XY`
- `turn_on_multiple_rgb_leds(color, num_leds)` — `DiaTstRgb00XYYY`
- `turn_on_single_led_on_switch(switch, led, color)` — `PutSglLedXXYZ`
- `turn_on_all_white_leds()` — `PutAllLED`
- `get_daylight_sensor_values()` — `GetTstIlu`

### Escape Hatch
- `send_raw_command(message, expect_response)` — send any raw ASCII string

## Common Tasks

- **Set fixture to assembly mode:** call `set_fixture_status("Assembly")`
- **Check all switches present:** call `check_switch_present` for each index 0–(n-1)
- **Program a switch:** call `program_switch_eeprom(index, pn, sn)` — PN must be 12 chars, SN must be 12 chars
- **Run button test:** `initialise_button_test()` → ask user to press all switches → `get_button_result()`
- **Debug LED positions:** use `set_multiple_leds` with `active_color="Red"` and `inactive_color="Off"` to highlight specific slots
