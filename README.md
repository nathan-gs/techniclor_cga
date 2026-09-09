
This project provides several **sensor entities** for a Technicolor CGA gateway in Home Assistant. It reads system and DHCP information, lists connected hosts, and offers a **delta sensor** to detect missing/inactive devices. Thanks to `device_info`, all entities are grouped under **one device** in Home Assistant's device and integrations registry.

## Features

- **System status** (e.g., `CMStatus`) including pass-through of additional system attributes
- **DOCSIS RF sensors**: downstream/upstream power, downstream SNR, and corrected/uncorrectable codeword counters (from the `levels()` tables)
- **WAN / LAN interface sensors**: WAN link status, WAN packet/byte/error counters, and a LAN-ports summary (from `dig_interface()`)
- **DHCP sensors** for all DHCP keys returned by the gateway
- **Host list** with the number of currently detected devices (`hostTbl`)
- **Missing devices / Delta sensor**: shows devices that disappeared or are inactive
- **Clean device grouping** via `device_info` (identifiers = `(DOMAIN, host)`, manufacturer, name, `configuration_url`); model/firmware are added when available
- **Automatic polling** every 5 minutes

## Directory structure 
```
custom_components/technicolor_cga/
├─ __init__.py
├─ config_flow.py
├─ manifest.json
├─ const.py
├─ technicolor_cga.py
└─ sensor.py
```

## Installation

> The integration uses **Config Entries** (UI-based setup).

### HACS
v0.9.1: is installable over HACS custom repo

### Manual
1. Copy this folder to `config/custom_components/technicolor_cga/`.
2. **Restart** Home Assistant.
3. Go to **Settings → Devices & Services → Add Integration** and pick *Technicolor CGA*.
4. Enter your credentials:
   - **Host** (e.g., `192.168.0.1`)
   - **Username**
   - **Password**

## Created entities

### System sensor
- **Name:** `Technicolor CGA System Status`
- **State:** value of `CMStatus` (or `"Unknown"`)
- **Attributes:** all other system fields (e.g., `ModelName`, `SoftwareVersion`, etc.).
- **Device info:** `model`/`sw_version` are set from system data when present.

### DOCSIS RF sensors
Derived from the modem's `levels()` tables (`DSTbl`/`USTbl`, the OFDM/OFDMA
`exDSTbl`/`exUSTbl`, and the `ErrTbl` error counters). One shared, briefly
cached fetch feeds all of them.

- **Downstream Power** — average receive power across all downstream channels (`dBmV`); attributes: `min_dbmv`, `max_dbmv`, `channel_count`, and a per-channel breakdown.
- **Downstream SNR** — worst-case (minimum) SNR across downstream channels (`dB`); attributes: `min_db`, `max_db`, `avg_db`.
- **Upstream Power** — average transmit power across upstream channels (`dBmV`); attributes: per-channel breakdown.
- **Downstream Correcteds** — total corrected codewords (`total_increasing`, diagnostic).
- **Downstream Uncorrectables** — total uncorrectable codewords (`total_increasing`, diagnostic). The key line-health metric — it should stay flat.
- **DOCSIS Channels** — number of locked channels; attributes carry the raw `DSTbl`/`USTbl`/`exDSTbl`/`exUSTbl`/`ErrTbl` tables for drill-down.

### WAN / LAN interface sensors
Derived from `dig_interface()` (WAN uplink, physical LAN ports and WiFi radios).
One shared, briefly cached fetch feeds all of them.

- **WAN Status** — WAN link state (`Up`/`Down`); attributes carry link speed, duplex and all WAN counters.
- **WAN Packets Received / Sent** — cumulative packet counters (`total_increasing`).
- **WAN Bytes Received / Sent** — cumulative byte counters (`total_increasing`, `data_size`).
- **WAN Errors Received / Sent** — cumulative error counters (`total_increasing`, diagnostic).
- **LAN Ports** — number of LAN ports that are up; attributes carry the full per-port table (`LANEtherTable`) and `LANStats`.

> **Note:** the byte counters are 32-bit on this firmware and clamp at
> `2147483647` (2³¹−1) instead of wrapping, so `WAN Bytes *` becomes
> unreliable once the interface has passed ~2 GB since its last reset. The
> packet and error counters do not have this limitation.

### DHCP sensors
- **Name:** `Technicolor CGA DHCP <Key>` (for each key returned by `dhcp()`)
- **State:** corresponding value from DHCP data (or `"Unknown"`).

### Host sensor
- **Name:** `Technicolor CGA Host List`
- **State:** number of entries in `hostTbl`.
- **Attributes:** full host data structure (e.g., `hostTbl`, entries with `physaddress`, `ipaddress`, `hostname`, `active`).

### Delta / Missing devices sensor
- **Name:** `Technicolor CGA Missing Devices`
- **State:** number of detected *missing* or *inactive* devices.
- **Attributes:**
  - `missing_devices`: list of dicts `{mac, last_ip, hostname, status}`
  - `known_devices`: list of learned devices `{mac, last_ip, hostname}`
- **Notes:**
  - The `known_devices` list is **learned at runtime** (no persistence across restarts).
  - Sorting is numeric by IP; invalid IPs are placed at the end.

## Update interval

By default every **5 minutes** (`SCAN_INTERVAL = 300s`).

## Tips / Troubleshooting

- Verify `Host`, `Username`, `Password` and that the web interface is reachable.
- Some gateways return slightly different field names (`ModelName` vs. `Model`, `SoftwareVersion` vs. `SWVersion`/`FirmwareVersion`). The code handles common variants.
- The delta sensor only learns devices after they have been seen at least once.

## Development

- Entities inherit from `SensorEntity` (the base class provides `device_info`).
- **Unique IDs** are based on `config_entry_id` + entity name.
- Polling via `async_track_time_interval`.
- The API class `TechnicolorCGA` is called in the executor (`login`, `system`, `dhcp`, `aDev`).
