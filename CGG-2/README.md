[README.md](https://github.com/user-attachments/files/26788398/README.md)
# Wolf CGG-2 ebusd Configuration

ebusd configuration for the **Wolf CGG-2** gas condensing boiler with **Wolf BM** control module.

Tested with:
- Wolf CGG-2-18 (Feuerungsautomat: Kromschröder, SW=0204)
- Wolf BM (Master=0x30, Slave=0x35, SW=0204)
- Wolf Funk-Empfänger (Master=0x0f, Slave=0x0a)
- ebusd v26.1.6
- Home Assistant with MQTT integration

---

## Files

| File | Description |
|---|---|
| `15.csv` | Main configuration: HG parameters, operating data, error history |
| `_templates.csv` | Field type templates (standard Wolf/Kromschröder) |
| `broadcast.csv` | Broadcast messages incl. outside temperature from Funk-Empfänger |
| `mqtt-hassio.cfg` | MQTT auto-discovery config for Home Assistant |

---

## Bus Layout

```
QQ=03 / ZZ=08  — Feuerungsautomat (boiler controller, Kromschröder)
QQ=30 / ZZ=35  — BM control module (master/slave)
QQ=0f / ZZ=0a  — Funk-Empfänger (wireless receiver)
QQ=f1          — tado° wireless receiver (E1 relay mode, HG13=1)
```

---

## Working Parameters

### Operating Data (circuit: betrd_bm, ZZ=08, PBSB=5022)

| Name | Register | Description |
|---|---|---|
| `temp_burner` | CC0D00 | Boiler temperature °C |
| `performance_burner` | CC6F01 | Modulation % |
| `performance_pump` | CC5727 | Pump speed % |
| `no_of_firing` | CC2602 | Burner starts (total) |
| `op_hrs_heating` | CC2A02 | Burner operating hours |
| `pressure` | CC1A27 | System pressure bar |

### HG Parameters (circuit: feuerung, ZZ=08, PBSB=5022/5023)

All HG parameters support read (`r`) and write (`w`). Write uses PBSB=5023 with suffix `9d010000`.

| Parameter | Register | Description | Range |
|---|---|---|---|
| HG01 | 842200 | Burner differential | 5–30 K |
| HG06 | 254101 | Pump mode | 0–2 |
| HG07 | c14201 | Pump run-on time | 0–30 min |
| HG08 | de8402 | Max flow temperature | 40–90 °C |
| HG09 | 9d4301 | Burner lockout time | 1–30 min |
| HG10 | ad7801 | eBUS address | — |
| HG11 | a56a01 | DHW quick start | °C |
| HG12 | f96b01 | Gas type | — |
| HG13 | 7e5b0a | Input E1 function | 0–11 |
| HG15 | 794001 | DHW hysteresis | 1–30 °C |
| HG16 | b95501 | Min pump speed HC | 20–100 % |
| HG17 | 5d5601 | Max pump speed HC | 20–100 % |
| HG21 | 201f00 | Min boiler temp (corrosion protection) | **do not set below 40°C** |
| HG22 | f42700 | Max boiler temp | °C |
| HG74 | d5f601 | Fan speed | U/sec |
| HG75 | 316c01 | DHW flow rate | l/min |
| HG90 | de2a02 | Burner operating hours | h |
| HG91 | aa2602 | Burner starts | — |

### Error History (HG80–HG89)

Registers `3a2902`–`hg89` contain raw 2-byte error codes. Encoding is proprietary and not fully decoded. Stored as HEX for reference.

### Flow/Return/DHW Temperatures

| Name | Register | Description |
|---|---|---|
| `vorlauf_soll` | b80200 | Flow setpoint °C |
| `vorlauf_ist` | 280d00 | Flow actual °C |
| `warmwasser_soll` | e40300 | DHW setpoint °C |
| `warmwasser_ist` | cc0e00 | DHW actual °C |

> **Note:** `temp_return` (return temperature) is **not available** on the CGG-2 hardware. The sensor register exists but always returns 0x7FFF (no sensor fitted).

---

## BM Broadcast Messages (PBSB=5022, QQ=30, ZZ=fe)

The BM control module broadcasts its own A-parameters and HG23 periodically.

**Frame structure:** `30 fe 5023 09 [ID 3 bytes] [value SIN16LE/10] 5d010000`

**ID schema:** `[CRC prefix byte, ignored by device] [TelegramNr LE 2 bytes]`

All IDs are readable via: `hex -s 31 35 5022 03 XX XX XX` (ZZ=35=BM slave)

| ID | TelegramNr | Parameter | Value | Description |
|---|---|---|---|---|
| 248101 / 6c8101 | 385 | A14 = HG23 | e.g. 65.0°C | Max DHW temperature |
| 440f01 | 271 | A00 | e.g. 4 | Room influence factor K/K |
| 28000a | 2560 | A09 | e.g. -10.0°C | Frost protection level |
| 08be02 | 702 | A12 | e.g. -20.0°C | Setback stop |
| acbd02 | 701 | A11 | 0=off / 1=on | Room temp summer/winter switchover |
| 241401 | 276 | unknown | — | BM parameter, meaning unknown |
| 642300 | 35 | unknown | — | BM parameter, meaning unknown |

> **Source for TelegramNr mapping:** `ism7mqtt` ParameterTemplates.xml / DeviceTemplates.xml (zivillian/ism7mqtt)

---

## Known Limitations

### PBSB=0503 — not decodable via ebusd CSV

Messages `f1fe0503` and `03fe0503` contain DHW status (D1) and DHW actual temperature (D6), but the `b`-type passive matching does not work in ebusd v26 for non-standard PBSB values regardless of CSV content.

### PBSB=0800 — passive matching not possible

Standard eBUS betrd broadcast (all three variants: QQ=10→03, f1→fe, 03→f1) contains:
- D0–D1: Boiler setpoint temperature (UIN16 LE / 256)
- D2–D3: Outside temperature (UIN16 LE / 256)
- D5: Operating status (0x00=heating, 0x40=setback/standby)
- D7: Active setpoint °C

Also not decodable via `b`-type in ebusd v26.

### QQ=0f broadcast (Funk-Empfänger)

Outside temperature is available via the standard `broadcast/datetime` topic (broadcast.csv). Use this instead of polling from ZZ=08.

---

## HG23 Write via Broadcast

The BM (ZZ=35) **does not accept direct MS writes** for some parameters. HG23 (DHW max temperature) must be written as a broadcast:

```bash
# Example: set HG23 to 65°C (= 650 × 10 = 0x028a → LE: 8a 02)
echo "hex -s 30 fe502309248101{lo}{hi}5d010000" | nc 172.30.33.1 8888
```

Where `{lo}` and `{hi}` are the little-endian bytes of `value × 10`.

---

## Home Assistant Integration

### MQTT auto-discovery

Use `mqtt-hassio.cfg` with the ebusd `--mqttint` option.

### Polling automation

Several registers require active polling (`read -f`). Example automation included in the accompanying HA package files.

### DHW one-time heating

Triggering a DHW heating cycle outside the scheduled times:
- Set HG15 temporarily to 1°C → boiler ignites
- Restore original HG15 value after completion
- Only works during DHW heating time windows configured in the BM

---

## Bus Scan Result (reference)

```
address 08: slave #11  MF=Kromschroeder  SW=0204  (Feuerungsautomat)
address 0a: slave #10  MF=Kromschroeder  SW=0204  (Funk-Empfänger slave)
address 35: slave #3   MF=Kromschroeder  SW=0204  (BM slave)
```

---

## References

- [ebusd](https://github.com/john30/ebusd)
- [ebusd-configuration](https://github.com/john30/ebusd-configuration)
- [ism7mqtt](https://github.com/zivillian/ism7mqtt) — Wolf SmartSet parameter extraction
- [ebusd Discussion #779](https://github.com/john30/ebusd/discussions/779) — Wolf hardware collection
- FHEM forum topic 95173 — Wolf CGG-2 eBUS findings

---

## Acknowledgements

This configuration was developed by reverse engineering the Wolf CGG-2
eBUS protocol through live bus analysis, parameter correlation, and
cross-referencing with Wolf SmartSet XML data.

Development was carried out with significant assistance from
Claude (Anthropic AI), which supported protocol analysis,
decoding, and documentation.

Community resources that contributed:
- ebusd Discussion #779 — Wolf hardware collection (john30/ebusd)
- ism7mqtt (zivillian) — Wolf SmartSet parameter extraction
- FHEM forum topic 95173 — early CGG-2 findings

---

## File Naming and Placement

### Why the files are named as they are

ebusd uses a **scanconfig mechanism** to automatically load the correct
configuration files based on the device identification received from the bus.
When a slave device responds to a scan, ebusd resolves the manufacturer name
and looks for a CSV file matching the slave address in the manufacturer
subdirectory.

The Wolf CGG-2 Feuerungsautomat responds on slave address `0x08`, and the
BM control module responds on slave address `0x35`. Our main configuration
file is named `15.csv` — this corresponds to slave address `0x15` (decimal 21),
which is the slave address of the second controller (Regler 2) in the
Kromschröder/Wolf bus layout.

The supporting files follow ebusd conventions:
- `_templates.csv` — required prefix `_` marks it as a template file, loaded first
- `broadcast.csv` — reserved name for broadcast message definitions
- `mqtt-hassio.cfg` — configuration for ebusd MQTT integration with Home Assistant

### Where to place the files

Place all files in your ebusd configuration directory. With the ebusd
Home Assistant addon, this is typically `/addon_configs/<addon_id>/`
which maps to `/config/` inside the addon container.

Set the ebusd option:
```
--configpath=/config
```

The directory should contain:
```
/config/
├── 15.csv
├── _templates.csv
├── broadcast.csv
└── mqtt-hassio.cfg
```

ebusd will load all files from this directory on startup. No `--scanconfig`
is needed — the files are loaded directly without device identification matching.

> **Note:** If you use `--scanconfig`, place the files in a `kromschroeder/`
> subdirectory and ensure the device scan returns manufacturer `Kromschroeder`.
> The CGG-2 Feuerungsautomat (ZZ=08) may return an empty ID string which
> prevents automatic scanconfig matching.
