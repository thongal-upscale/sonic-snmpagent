# SONiC SNMP — MIB Compliance & Gap Guide for Developers

> **Who is this for?**  
> Engineers who are new to SNMP and SONiC and want to understand what the three core MIBs expose today,
> where the known gaps are, and how to run real commands against the switch to verify or investigate each OID.
>
> **Platform:** Spectrum-4 based SONiC target  
> **Enterprise (PEN):** UpscaleAI — `64820`  
> **Branch:** `thongal_nms_compliance1` · Base commit `6bc7412`  
> **Scope:** Interface MIB (RFC 1213 / RFC 2863) · Entity MIB (RFC 2737) · Entity Sensor MIB (RFC 3433)

---

## Table of Contents

1. [Quick-Start: Environment Setup](#1-quick-start-environment-setup)
2. [How SONiC Serves SNMP Data](#2-how-sonic-serves-snmp-data)
3. [Interface MIB (RFC 1213 / RFC 2863)](#3-interface-mib-rfc-1213--rfc-2863)
   - [What is implemented](#31-what-is-implemented)
   - [Known gaps & how to confirm them](#32-known-gaps--how-to-confirm-them)
4. [Entity MIB (RFC 2737)](#4-entity-mib-rfc-2737)
   - [What is implemented](#41-what-is-implemented)
   - [Known gaps & how to confirm them](#42-known-gaps--how-to-confirm-them)
5. [Entity Sensor MIB (RFC 3433)](#5-entity-sensor-mib-rfc-3433)
   - [What is implemented](#51-what-is-implemented)
   - [Known gaps & how to confirm them](#52-known-gaps--how-to-confirm-them)
6. [Verification Cheatsheet](#6-verification-cheatsheet)
7. [How to Fill a Gap](#7-how-to-fill-a-gap)

---

## 1. Quick-Start: Environment Setup

### On the SONiC switch (one-time)

SNMP community strings live in `/etc/sonic/config_db.json`. Add a read community for your test session:

```json
{
    "SNMP": {
        "global": {
            "chassis_id": "Spectrum4-Switch01",
            "contact":    "network-admin@company.com",
            "location":   "DataCenter-Rack-05"
        }
    },
    "SNMP_COMMUNITY": {
        "public": { "name": "public", "vlan": "all" }
    }
}
```

Apply the change:

```bash
sudo config reload -y
```

Verify the SNMP container is running:

```bash
docker ps | grep snmp
```

### On your Linux test machine

Install the net-snmp CLI tools and the Python SNMP library used in the test examples:

```bash
sudo apt-get install snmp snmp-mibs-downloader   # Ubuntu/Debian
pip install pysnmp pytest
```

Replace `<SWITCH_IP>` with your switch management IP in every command below.

---

## 2. How SONiC Serves SNMP Data

Understanding the data flow helps you know *where* to look when an OID returns a wrong value.

```
NMS / your laptop
      │  UDP 161
      ▼
  snmpd (Net-SNMP master)          ← docker-snmp container
      │  AgentX tcp:3161
      ▼
  sonic_ax_impl (Python subagent)  ← reads Redis every ~5 seconds
      │  SonicV2Connector
      ▼
  Redis STATE_DB / COUNTERS_DB / APPL_DB / CONFIG_DB
      ▲
  platform daemons (xcvrd, psud, fand, thermalctld) write sensor data here
```

Key takeaway: **if an OID returns wrong data, the problem is usually in Redis, not in snmpd**.
Check Redis first:

```bash
# On the switch — inspect what the subagent actually reads
docker exec -it database redis-cli -n 6 HGETALL "TRANSCEIVER_DOM_INFO|Ethernet0"
docker exec -it database redis-cli -n 6 HGETALL "PSU_INFO|PSU 1"
docker exec -it database redis-cli -n 6 HGETALL "FAN_INFO|FAN 1"
docker exec -it database redis-cli -n 6 HGETALL "THERMAL_INFO|ASIC"
```

---

## 3. Interface MIB (RFC 1213 / RFC 2863)

The Interface MIB is the most commonly queried MIB. It reports link status, speeds, and traffic counters for every port on the switch.

**OID root:** `.1.3.6.1.2.1` (MIB-II)

### 3.1 What is implemented

| OID | Name | What it tells you | Status |
|---|---|---|---|
| `.2.2.1.2` | `ifDescr` | Interface name (e.g. `Ethernet0`) | ✅ Works |
| `.2.2.1.3` | `ifType` | Physical type — `ethernetCsmacd(6)`, `propVirtual(53)` for LAGs | ✅ Works |
| `.2.2.1.4` | `ifMtu` | MTU in bytes | ✅ Works |
| `.2.2.1.5` | `ifSpeed` | 32-bit speed in bps — **wraps at 4 Gbps**, use `ifHighSpeed` for 100G+ | ✅ Works |
| `.2.2.1.6` | `ifPhysAddress` | MAC address | ✅ Works |
| `.2.2.1.7` | `ifAdminStatus` | Admin configured state: `up(1)` / `down(2)` | ✅ Works |
| `.2.2.1.8` | `ifOperStatus` | Actual link state: `up(1)` / `down(2)` / `dormant(5)` | ✅ Works |
| `.2.2.1.10`–`.21` | `ifIn/Out*` | 32-bit counters (wrap at ~4 GB — prefer HC below) | ✅ Works |
| `.31.1.1.1.6`–`.13` | `ifHCIn/OutOctets` etc. | **64-bit counters** — use these for 100G/400G ports | ✅ Works |
| `.31.1.1.1.15` | `ifHighSpeed` | Speed in Mbps — correct for 100G / 400G | ✅ Works |
| `.31.1.1.1.18` | `ifAlias` | Interface description field | ✅ Works |

**Quick test — verify a port is up and passing traffic:**

```bash
# From your test machine
SWITCH=<SWITCH_IP>
COMMUNITY=public

# Check all port states
snmpwalk -v2c -c $COMMUNITY $SWITCH .1.3.6.1.2.1.2.2.1.8

# Check 64-bit rx byte counter for index 10 (find the right index from ifDescr first)
snmpwalk -v2c -c $COMMUNITY $SWITCH .1.3.6.1.2.1.31.1.1.1.6

# Or locally on the switch (no firewall issues)
docker exec -it snmp snmpwalk -v2c -c public localhost .1.3.6.1.2.1.31.1.1.1.6
```

---

### 3.2 Known gaps & how to confirm them

#### GAP-IF-01 — `ifLastChange` always returns 0

**What it should be:** The timestamp (in hundredths of a second since boot, a.k.a. `sysUpTime`) of the last time this interface changed its `ifOperStatus`.

**What you get today:** Always `0`.

**Why it matters:** You cannot tell how long a port has been up or down. Post-mortem analysis and SLA reporting require this.

**How to confirm:**

```bash
# Every interface returns Timeticks: (0) 0:00:00.00
snmpwalk -v2c -c public <SWITCH_IP> .1.3.6.1.2.1.2.2.1.9
```

**Where the gap lives in code:**

```
src/sonic_ax_impl/mibs/ietf/rfc1213.py  line 631
ifLastChange = SubtreeMIBEntry('2.1.9', if_updater, ValueType.TIME_TICKS, lambda sub_id: 0)
# FIXME Placeholder — hardcoded zero
```

**What a fix looks like (conceptual):** The updater needs to record the current `sysUpTime` whenever `ifOperStatus` changes for an interface, then return that stored value here.

---

#### GAP-IF-02 — Link traps disabled (`ifLinkUpDownTrapEnable` hardcoded `disabled(2)`)

**What it should be:** When set to `enabled(1)`, the SNMP agent automatically sends a `linkUp` or `linkDown` notification to configured trap receivers whenever a port changes state. NMS platforms like Zabbix, Nagios, and PRTG rely on this for near-real-time alerting.

**What you get today:** Always `disabled(2)` — traps are never sent.

**How to confirm:**

```bash
# All entries return INTEGER: 2 (disabled)
snmpwalk -v2c -c public <SWITCH_IP> .1.3.6.1.2.1.31.1.1.1.14
```

**Where the gap lives in code:**

```
src/sonic_ax_impl/mibs/ietf/rfc2863.py  line 432
ifLinkUpDownTrapEnable = SubtreeMIBEntry('1.1.14', if_updater, ValueType.INTEGER, lambda sub_id: 2)
```

---

#### GAP-IF-03 — Management interface (`mgmt0`) counters always return 0

**What it should be:** Real byte and packet counters for the out-of-band management port (`eth0` / `mgmt0`).

**What you get today:** All counter OIDs for the management interface return `0` because the COUNTERS_DB does not track Linux-native interfaces.

**How to confirm:**

```bash
# Find the ifIndex for the management port
snmpwalk -v2c -c public <SWITCH_IP> .1.3.6.1.2.1.2.2.1.2  # ifDescr — look for eth0/mgmt0

# Then walk its counters — all return 0
snmpwalk -v2c -c public <SWITCH_IP> .1.3.6.1.2.1.2.2.1.10  # ifInOctets
```

**Where the gap lives in code:**

```
src/sonic_ax_impl/mibs/ietf/rfc1213.py  lines 395–399
if oid in self.mgmt_oid_name_map:
    # TODO: mgmt counters not available through SNMP right now
    return 0
```

---

#### GAP-IF-04 — `ifStackTable` not implemented

**What it should be:** A table that maps LAG (PortChannel) interfaces to their member physical ports — the standard SNMP way to discover bond topology.

**What you get today:** The entire OID subtree is absent.

**How to confirm:**

```bash
# Returns nothing
snmpwalk -v2c -c public <SWITCH_IP> .1.3.6.1.2.1.31.1.2
```

---

#### GAP-IF-05/06/07 — Minor hardcoded values (low impact)

| OID | Name | Current value | Correct value |
|---|---|---|---|
| `.31.1.1.1.16` | `ifPromiscuousMode` | always `true(1)` | Reflect actual kernel state |
| `.31.1.1.1.17` | `ifConnectorPresent` | always `true(1)` | `false(2)` for virtual interfaces |
| `.31.1.1.1.19` | `ifCounterDiscontinuityTime` | always `0` | `sysUpTime` at last counter reset |

```bash
# Verify each — all return the hardcoded value regardless of actual interface type
snmpwalk -v2c -c public <SWITCH_IP> .1.3.6.1.2.1.31.1.1.1.16
snmpwalk -v2c -c public <SWITCH_IP> .1.3.6.1.2.1.31.1.1.1.17
snmpwalk -v2c -c public <SWITCH_IP> .1.3.6.1.2.1.31.1.1.1.19
```

---

## 4. Entity MIB (RFC 2737)

The Entity MIB describes the *physical inventory* of the switch — chassis, line cards, fan drawers, PSUs, transceivers, and sensors — as a tree. Every component is one row in `entPhysicalTable`.

**OID root:** `.1.3.6.1.2.1.47`  
**UpscaleAI vendor-type OIDs:** `1.3.6.1.4.1.64820.1.1.4.*` (PEN 64820)

### Understanding the physical entity tree

```
Index 1               chassis 1                      ← CHASSIS(3)
├── Index 200000000   MGMT CPU                        ← CPU(12)
├── Index 5001000     Fan Drawer 1                    ← CONTAINER(5)
│   └── Index ...     Fan 1                           ← FAN(7)
├── Index 6001000     PSU 1                           ← POWERSUPPLY(6)
│   └── Index ...     PSU voltage/temp/current sensors ← SENSOR(8)
├── Index 7001000     Fabric Card 1                   ← MODULE(9)
└── Index 1000100000  Ethernet0 transceiver           ← PORT(10)
    └── Index ...     Transceiver temperature/voltage  ← SENSOR(8)
```

### 4.1 What is implemented

| OID suffix | Name | What it tells you | Status |
|---|---|---|---|
| `1.1.1.2` | `entPhysicalDescr` | Human-readable description (e.g. "PSU 1") | ✅ Works |
| `1.1.1.4` | `entPhysicalContainedIn` | Parent entity index (0 = root) | ✅ Works |
| `1.1.1.5` | `entPhysicalClass` | Type: chassis/fan/sensor/module/port (see table below) | ✅ Works |
| `1.1.1.6` | `entPhysicalParentRelPos` | Slot position within parent | ✅ Works |
| `1.1.1.7` | `entPhysicalName` | Short name / DB key (e.g. `Ethernet0`) | ✅ Works |
| `1.1.1.8` | `entPhysicalHardwareVersion` | Hardware revision from TRANSCEIVER_INFO | ✅ Works |
| `1.1.1.11` | `entPhysicalSerialNumber` | Serial number from PSU/FAN/XCVR info | ✅ Works |
| `1.1.1.12` | `entPhysicalMfgName` | Manufacturer name | ✅ Works |
| `1.1.1.13` | `entPhysicalModelName` | Part number / model | ✅ Works |
| `1.1.1.16` | `entPhysicalIsFRU` | `true(1)` if field-replaceable, `false(2)` if not | ✅ Works |

**Physical class integer values** (what you'll see in `entPhysicalClass` walks):

| Integer | Class name | Examples |
|---|---|---|
| 3 | CHASSIS | The switch itself |
| 5 | CONTAINER | Fan drawer, PSU bay |
| 6 | POWERSUPPLY | PSU |
| 7 | FAN | Individual fan |
| 8 | SENSOR | Temperature, voltage, current sensors |
| 9 | MODULE | Fabric card, line card |
| 10 | PORT | Transceiver slot |
| 12 | CPU | Management CPU |

**Quick test — walk the full physical inventory tree:**

```bash
SWITCH=<SWITCH_IP>

# See all physical entities with their class
snmpwalk -v2c -c public $SWITCH .1.3.6.1.2.1.47.1.1.1.1.5

# See descriptions for all entities
snmpwalk -v2c -c public $SWITCH .1.3.6.1.2.1.47.1.1.1.1.2

# Check serial numbers of all FRUs
snmpwalk -v2c -c public $SWITCH .1.3.6.1.2.1.47.1.1.1.1.11

# Verify the chassis is at index 1 with class 3
snmpget -v2c -c public $SWITCH .1.3.6.1.2.1.47.1.1.1.1.5.1
# Expected: INTEGER: 3
```

---

### 4.2 Known gaps & how to confirm them

#### GAP-ENT-01 — `entPhysicalFirmwareVersion` always returns empty string

**What it should be:** The firmware version of the physical component (CPLD version, transceiver firmware, PSU MCU version, etc.).

**What you get today:** An empty string `""` for every entity.

**How to confirm:**

```bash
# All entries return empty string ""
snmpwalk -v2c -c public <SWITCH_IP> .1.3.6.1.2.1.47.1.1.1.1.9
```

**Cross-check in Redis** — if the data is in the DB but missing from SNMP:

```bash
docker exec -it database redis-cli -n 6 HGET "TRANSCEIVER_INFO|Ethernet0" "firmware_version"
```

**Where the gap lives in code:**

```
src/sonic_ax_impl/mibs/ietf/rfc2737.py
def get_phy_fw_ver(self, sub_id):
    return "" if sub_id in self.physical_entities else None
    # Returns empty string instead of reading from DB
```

---

#### GAP-ENT-02 — `entPhysicalSoftwareRevision` always returns empty string

Same pattern as GAP-ENT-01. OID `.1.3.6.1.2.1.47.1.1.1.1.10` returns `""` for every entity.

```bash
snmpwalk -v2c -c public <SWITCH_IP> .1.3.6.1.2.1.47.1.1.1.1.10
# Expected (gap): all empty strings
```

---

#### GAP-ENT-03 — `entPhysicalVendorType` always returns empty (UpscaleAI PEN not populated)

**What it should be:** A vendor-specific OID that identifies the *exact hardware part*. For UpscaleAI, this should be a sub-OID of `1.3.6.1.4.1.64820.1.1.4.*` (PEN 64820).

**What you get today:** Empty string for every entity.

**How to confirm:**

```bash
snmpwalk -v2c -c public <SWITCH_IP> .1.3.6.1.2.1.47.1.1.1.1.3
# Expected (gap): all empty strings instead of 1.3.6.1.4.1.64820.1.1.4.x OIDs
```

**How to fix (conceptual):**
1. Ensure platform plugins write the vendor OID string into Redis `STATE_DB` under the relevant info table
2. Update `get_phy_vendor_type()` in `rfc2737.py` to read that field from the DB

---

#### GAP-ENT-04 — `entPhysicalContainsTable` not implemented

**What it should be:** A reverse lookup table — given a parent entity index, list all child entity indices it contains. This is the other direction of `entPhysicalContainedIn`.

**What you get today:** Subtree absent.

```bash
snmpwalk -v2c -c public <SWITCH_IP> .1.3.6.1.2.1.47.1.3.3
# Returns nothing
```

---

#### GAP-ENT-05 — Line card MODULE entities not populated

**What it should be:** For modular chassis designs, each line card should appear as a `MODULE(9)` entity in the tree, parented to the chassis, with its own serial number and model.

**What you get today:** Only fabric cards read from `CHASSIS_MODULE_TABLE` are added. Data-path line cards are absent.

**How to confirm:**

```bash
# Walk entPhysicalClass and count MODULE(9) entries — fabric cards only, no line cards
snmpwalk -v2c -c public <SWITCH_IP> .1.3.6.1.2.1.47.1.1.1.1.5 | grep " 9$"
```

---

## 5. Entity Sensor MIB (RFC 3433)

The Entity Sensor MIB extends the Entity MIB. Every `SENSOR(8)` entity in `entPhysicalTable` has a corresponding row here that gives the *actual numeric reading* (temperature, voltage, current, power, fan speed).

**OID root:** `.1.3.6.1.2.1.99.1.1`  
**Index:** Same `entPhysicalIndex` as the SENSOR entries in RFC 2737.

### Understanding sensor value encoding

Sensors use a scale + precision system to encode floats as integers:

```
Actual value = raw_integer / (10 ^ precision)

Examples:
  Temperature  40500000  precision=6  → 40.5 °C
  Voltage      33000     precision=4  → 3.3 V
  Current      7500      precision=3  → 7.5 mA
  Power        60000     precision=3  → 60 W
  Fan speed    3600      precision=0  → 3600 RPM
```

### Sensor sources in Redis

| Sensor | Redis table | Key |
|---|---|---|
| Transceiver temperature | `TRANSCEIVER_DOM_INFO\|EthernetN` | `temperature` |
| Transceiver voltage | `TRANSCEIVER_DOM_INFO\|EthernetN` | `voltage` |
| Transceiver RX power | `TRANSCEIVER_DOM_INFO\|EthernetN` | `rx1power` … `rx8power` |
| Transceiver TX power | `TRANSCEIVER_DOM_INFO\|EthernetN` | `tx1power` … `tx8power` |
| Transceiver TX bias | `TRANSCEIVER_DOM_INFO\|EthernetN` | `tx1bias` … `tx8bias` |
| PSU temperature | `PSU_INFO\|PSU N` | `temp` |
| PSU voltage | `PSU_INFO\|PSU N` | `voltage` |
| PSU current | `PSU_INFO\|PSU N` | `current` |
| PSU power | `PSU_INFO\|PSU N` | `power` |
| Fan speed | `FAN_INFO\|FAN N` | `speed` |
| Chassis thermal | `THERMAL_INFO\|<sensor name>` | `temperature` |

### 5.1 What is implemented

| OID suffix | Name | What it tells you | Status |
|---|---|---|---|
| `1.1.1.1` | `entPhySensorType` | Sensor kind — Celsius(8), Volts(4), Amperes(5), Watts(6), RPM(10) | ✅ Works |
| `1.1.1.2` | `entPhySensorScale` | SI multiplier — `units(9)`, `milli(8)`, `micro(7)` | ✅ Works |
| `1.1.1.3` | `entPhySensorPrecision` | Decimal places to divide the value by | ✅ Works |
| `1.1.1.4` | `entPhySensorValue` | The actual reading as an integer | ✅ Works |
| `1.1.1.5` | `entPhySensorOperStatus` | `ok(1)` if the reading is valid, `unavailable(2)` if parsing failed | ✅ Works |

**Quick test — read all sensor values:**

```bash
SWITCH=<SWITCH_IP>

# Walk all sensor readings (raw integers — decode using type/scale/precision)
snmpwalk -v2c -c public $SWITCH .1.3.6.1.2.1.99.1.1.1.4

# Check which sensors are OK vs unavailable
snmpwalk -v2c -c public $SWITCH .1.3.6.1.2.1.99.1.1.1.5

# Read sensor type for a specific index (e.g. index 100000001)
snmpget -v2c -c public $SWITCH .1.3.6.1.2.1.99.1.1.1.1.100000001
# 8 = CELSIUS, 4 = VOLTS_DC, 5 = AMPERES, 6 = WATTS

# Match sensor index back to its entity name
snmpget -v2c -c public $SWITCH .1.3.6.1.2.1.47.1.1.1.1.7.100000001
```

**Cross-check directly in Redis when a sensor shows `unavailable(2)`:**

```bash
docker exec -it database redis-cli -n 6 HGET "TRANSCEIVER_DOM_INFO|Ethernet0" "temperature"
docker exec -it database redis-cli -n 6 HGET "PSU_INFO|PSU 1" "temp"
```

---

### 5.2 Known gaps & how to confirm them

#### GAP-SENS-01 — `entPhySensorUnitsDisplay` not implemented

**What it should be:** A human-readable string showing the unit for each sensor — `"Celsius"`, `"Volts DC"`, `"Watts"`, `"rpm"`. This OID lets generic monitoring tools label sensor readings correctly without hardcoding logic.

**What you get today:** The entire OID subtree is absent.

**How to confirm:**

```bash
snmpwalk -v2c -c public <SWITCH_IP> .1.3.6.1.2.1.99.1.1.1.6
# Returns nothing (no rows at all)
```

**Workaround for now:** Map sensor type integers to unit strings in your NMS config manually:

```
type 4 → "Volts DC"
type 5 → "Amperes"
type 6 → "Watts"
type 8 → "Celsius"
type 10 → "RPM"
```

---

#### GAP-SENS-02 — `entPhySensorValueTimeStamp` not implemented

**What it should be:** The `sysUpTime` value at the moment this sensor reading was last refreshed. Useful for detecting stale readings (the subagent polls Redis every ~5 seconds — if the daemon has died, readings could be hours old).

**What you get today:** OID subtree absent.

```bash
snmpwalk -v2c -c public <SWITCH_IP> .1.3.6.1.2.1.99.1.1.1.7
# Returns nothing
```

---

#### GAP-SENS-03 — `entPhySensorValueUpdateRate` not implemented

**What it should be:** The sensor's polling interval in milliseconds. Lets NMS tools know the fastest meaningful polling rate (no point querying faster than the update rate).

**What you get today:** OID subtree absent. The actual internal rate is 5 000 ms.

```bash
snmpwalk -v2c -c public <SWITCH_IP> .1.3.6.1.2.1.99.1.1.1.8
# Returns nothing
```

---

#### GAP-SENS-04 — `entSensorThresholdTable` not implemented

**What it should be:** For each sensor, one or more threshold rows define the warning/critical/shutdown limits. Standard NMS platforms query this table to automatically configure threshold-based alerts without needing platform-specific knowledge.

**What you get today:** The entire table is absent.

**How to confirm:**

```bash
snmpwalk -v2c -c public <SWITCH_IP> .1.3.6.1.2.1.99.1.1.2
# Returns nothing
```

**Cross-check — the thresholds ARE in Redis** (they just aren't exposed via SNMP):

```bash
docker exec -it database redis-cli -n 6 HGET "THERMAL_INFO|ASIC" "high_threshold"
docker exec -it database redis-cli -n 6 HGET "THERMAL_INFO|ASIC" "critical_high_threshold"
docker exec -it database redis-cli -n 6 HGET "PSU_INFO|PSU 1" "temp_threshold"
```

This means the data exists — it just needs to be wired into the subagent's threshold table implementation.

---

#### GAP-SENS-05 — ASIC die temperature not mapped

**What it should be:** The Spectrum-4 ASIC die temperature is the most critical thermal metric on the platform. It should appear as a `SENSOR(8)` entity in `entPhysicalTable` and have a reading in `entPhySensorTable`.

**What you get today:** `ASIC_TEMPERATURE_INFO` table in Redis is never read by the subagent.

**How to confirm:**

```bash
# Verify the data exists in Redis
docker exec -it database redis-cli -n 6 HGETALL "ASIC_TEMPERATURE_INFO"

# Then confirm it is NOT visible via SNMP by walking sensor values
# and checking that no ASIC entry appears
snmpwalk -v2c -c public <SWITCH_IP> .1.3.6.1.2.1.99.1.1.1.4
snmpwalk -v2c -c public <SWITCH_IP> .1.3.6.1.2.1.47.1.1.1.1.2 | grep -i asic
```

---

## 6. Verification Cheatsheet

A reference table of the most useful OIDs and commands, ready to copy-paste:

| What to check | OID | Command |
|---|---|---|
| All port states | `.1.3.6.1.2.1.2.2.1.8` | `snmpwalk -v2c -c public <IP> .1.3.6.1.2.1.2.2.1.8` |
| Port names | `.1.3.6.1.2.1.31.1.1.1.1` | `snmpwalk -v2c -c public <IP> .1.3.6.1.2.1.31.1.1.1.1` |
| 64-bit RX bytes | `.1.3.6.1.2.1.31.1.1.1.6` | `snmpwalk -v2c -c public <IP> .1.3.6.1.2.1.31.1.1.1.6` |
| 64-bit TX bytes | `.1.3.6.1.2.1.31.1.1.1.10` | `snmpwalk -v2c -c public <IP> .1.3.6.1.2.1.31.1.1.1.10` |
| Port speed (Mbps) | `.1.3.6.1.2.1.31.1.1.1.15` | `snmpwalk -v2c -c public <IP> .1.3.6.1.2.1.31.1.1.1.15` |
| ifLastChange (gap) | `.1.3.6.1.2.1.2.2.1.9` | `snmpwalk -v2c -c public <IP> .1.3.6.1.2.1.2.2.1.9` |
| Trap enable flag (gap) | `.1.3.6.1.2.1.31.1.1.1.14` | `snmpwalk -v2c -c public <IP> .1.3.6.1.2.1.31.1.1.1.14` |
| Full entity tree | `.1.3.6.1.2.1.47.1.1.1` | `snmpwalk -v2c -c public <IP> .1.3.6.1.2.1.47.1.1.1` |
| Entity classes | `.1.3.6.1.2.1.47.1.1.1.1.5` | `snmpwalk -v2c -c public <IP> .1.3.6.1.2.1.47.1.1.1.1.5` |
| Serial numbers | `.1.3.6.1.2.1.47.1.1.1.1.11` | `snmpwalk -v2c -c public <IP> .1.3.6.1.2.1.47.1.1.1.1.11` |
| Firmware versions (gap) | `.1.3.6.1.2.1.47.1.1.1.1.9` | `snmpwalk -v2c -c public <IP> .1.3.6.1.2.1.47.1.1.1.1.9` |
| Vendor type OIDs (gap) | `.1.3.6.1.2.1.47.1.1.1.1.3` | `snmpwalk -v2c -c public <IP> .1.3.6.1.2.1.47.1.1.1.1.3` |
| All sensor values | `.1.3.6.1.2.1.99.1.1.1.4` | `snmpwalk -v2c -c public <IP> .1.3.6.1.2.1.99.1.1.1.4` |
| Sensor op status | `.1.3.6.1.2.1.99.1.1.1.5` | `snmpwalk -v2c -c public <IP> .1.3.6.1.2.1.99.1.1.1.5` |
| Sensor units (gap) | `.1.3.6.1.2.1.99.1.1.1.6` | `snmpwalk -v2c -c public <IP> .1.3.6.1.2.1.99.1.1.1.6` |
| Sensor thresholds (gap) | `.1.3.6.1.2.1.99.1.1.2` | `snmpwalk -v2c -c public <IP> .1.3.6.1.2.1.99.1.1.2` |

**Useful bulk walk (fastest for large tables):**

```bash
snmpbulkwalk -v2c -c public -Cn0 -Cr25 <SWITCH_IP> .1.3.6.1.2.1.47.1.1.1
snmpbulkwalk -v2c -c public -Cn0 -Cr25 <SWITCH_IP> .1.3.6.1.2.1.99.1.1.1.4
```

**Run everything from inside the container (bypasses firewall / ACL issues):**

```bash
docker exec -it snmp snmpwalk -v2c -c public localhost .1.3.6.1.2.1.47.1.1.1
docker exec -it snmp snmpwalk -v2c -c public localhost .1.3.6.1.2.1.99.1.1.1
```

---

## 7. How to Fill a Gap

All gap fixes follow the same pattern inside `src/sonic_ax_impl/`:

```
1. Find the updater class in the relevant MIB file
   - Interface gaps  → mibs/ietf/rfc1213.py  or  mibs/ietf/rfc2863.py
   - Entity gaps     → mibs/ietf/rfc2737.py
   - Sensor gaps     → mibs/ietf/rfc3433.py

2. Add or update the data-fetch method in the updater
   - Read from Redis using self.statedb / self.counters_db
   - Key patterns:  STATE_DB  THERMAL_INFO|*  PSU_INFO|*  FAN_INFO|*
                    TRANSCEIVER_DOM_INFO|EthernetN
                    COUNTERS_DB  COUNTERS:<oid>

3. Register the OID by adding a SubtreeMIBEntry to the MIB class:
   entPhySensorUnitsDisplay = \
       SubtreeMIBEntry('1.6', updater, ValueType.OCTET_STRING, updater.get_sensor_units_display)

4. Test locally:
   docker exec -it snmp python3 -c "
   from sonic_ax_impl.mibs.ietf.rfc3433 import PhysicalSensorTableMIB
   print('loaded ok')
   "

5. Run the existing unit tests to make sure nothing regressed:
   cd /path/to/sonic-snmpagent
   pytest tests/ -v
```

---

*Analysis performed against commit `6bc7412` · Branch `thongal_nms_compliance1` · UpscaleAI PEN `64820`*
