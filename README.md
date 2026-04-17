# TAK-APRS Protocol Extension

**Version:** 1.0
**Date:** 2026-04-17
**Author:** cot_radio project

## Overview

This document describes how cot_radio encodes TAK (Team Awareness Kit) entity data within standard APRS packets. The encoding uses the APRS Object format with a structured comment field that carries TAK-specific metadata for round-trip fidelity between TAK and APRS systems.

Any APRS client or gateway can decode these packets as standard APRS objects. The TAK metadata in the comment field is optional — clients that don't understand it simply display the object normally.

---

## Packet Format

### APRS Object (Position)

TAK entities are transmitted as APRS Objects (not position reports):

```
ORIGINATOR>APRS,PATH:;NAME     *DDHHMMzDDMM.MMN/DDDMM.MMW[TAK:CALLSIGN:TEAM:COT_TYPE:ICONSETPATH
```

### Field Breakdown

| Field | Position | Format | Description |
|-------|----------|--------|-------------|
| ORIGINATOR | Header | Callsign-SSID | Gateway's ham callsign + SSID (e.g., `KN6ZPL-7`) |
| APRS | Header | Literal | APRS destination tocall |
| PATH | Header | Comma-separated | Digipeater path (e.g., `WIDE1-1,WIDE2-1`) |
| `;` | Byte 1 | Literal | APRS Object data type identifier |
| NAME | Bytes 2-10 | 9 chars, space-padded | Display name — TAK callsign with periods stripped, truncated to 9 chars |
| `*` or `_` | Byte 11 | Literal | `*` = live object, `_` = killed (deleted) object |
| TIMESTAMP | Bytes 12-18 | `DDHHMMz` | UTC day-hour-minute with `z` suffix |
| LATITUDE | Bytes 19-26 | `DDMM.MMN` | Latitude in degrees-minutes, N/S suffix |
| SYMBOL TABLE | Byte 27 | Single char | `/` = primary table, `\` = alternate table |
| LONGITUDE | Bytes 28-36 | `DDDMM.MMW` | Longitude in degrees-minutes, E/W suffix |
| SYMBOL CODE | Byte 37 | Single char | APRS symbol character (e.g., `[` = person) |
| COMMENT | Remaining | Free text | TAK metadata (see below) |

### Example Packets

**SA Position (team member):**
```
KN6ZPL-7>APRS,WIDE1-1,WIDE2-1:;HAWK     *170230z3406.36N/11818.12W[TAK:HAWK:Cyan:a-f-G-U-C
```

**Marker with custom icon:**
```
KN6ZPL-7>APRS,WIDE1-1,WIDE2-1:;ANTELOPE *170230z3404.84N/11737.01W[TAK:ANTELOPE 1:White:a-f-G-U-C:34ae1613-9645-4222-a9d2-e5f243dea2865/Animals/antelope.png
```

**Killed (deleted) object:**
```
KN6ZPL-7>APRS,WIDE1-1,WIDE2-1:;HAWK     _170235z3406.36N/11818.12W[TAK:HAWK:Cyan:a-f-G-U-C
```

---

## TAK Comment Field

The comment field begins with the `TAK:` prefix, followed by colon-separated metadata fields:

```
TAK:CALLSIGN:TEAM:COT_TYPE[:ICONSETPATH]
```

### Fields

| # | Field | Required | Description |
|---|-------|----------|-------------|
| 1 | `TAK` | Yes | Literal prefix — identifies this as a TAK-encoded object |
| 2 | CALLSIGN | Yes | Full TAK callsign (not truncated). May differ from the 9-char object name. |
| 3 | TEAM | Yes | TAK team color (e.g., `Cyan`, `White`, `Red`, `Green`). Empty string if no team. |
| 4 | COT_TYPE | Yes | Full COT type string (MIL-STD-2525 symbology). Examples: `a-f-G-U-C` (friendly ground unit), `a-h-G` (hostile ground), `a-n-G` (neutral ground). |
| 5 | ICONSETPATH | No | ATAK iconset path for custom icon rendering. Format: `{iconset_uid}/{group}/{filename.png}`. Only present for markers with custom icons. |

### Parsing Rules

1. Check if the comment field starts with `TAK:`
2. Split on `:` — fields are positional
3. Field 1 is always `TAK` (literal)
4. Field 2 is the full callsign
5. Field 3 is the team color (may be empty string between colons)
6. Field 4 is the COT type
7. Field 5 (optional) is the iconset path — if present, it contains `/` characters which should NOT be confused with the `:` delimiter

### Example Parsing

```
TAK:HAWK:Cyan:a-f-G-U-C
  → callsign = "HAWK"
  → team = "Cyan"
  → cot_type = "a-f-G-U-C"
  → iconsetpath = (none)

TAK:ANTELOPE 1::a-u-G:34ae1613.../Animals/antelope.png
  → callsign = "ANTELOPE 1"
  → team = "" (empty)
  → cot_type = "a-u-G"
  → iconsetpath = "34ae1613.../Animals/antelope.png"
```

---

## Object Name Encoding

The APRS object name is limited to exactly 9 characters (space-padded). TAK callsigns are encoded as follows:

1. Strip all period (`.`) characters from the callsign
2. Truncate to 9 characters
3. Right-pad with spaces to exactly 9 characters

The full unmodified callsign is preserved in the TAK comment field (field 2).

### Examples

| TAK Callsign | Object Name | Comment Field |
|--------------|-------------|---------------|
| HAWK | `HAWK     ` | `TAK:HAWK:Cyan:a-f-G-U-C` |
| FALCON | `FALCON   ` | `TAK:FALCON:Cyan:a-f-G-U-C` |
| ANTELOPE 1 | `ANTELOPE ` | `TAK:ANTELOPE 1:White:a-f-G-U-C` |
| M.5.679477 | `M5679477 ` | `TAK:M.5.679477::a-f-G` |

---

## APRS Message Encoding

TAK chat messages are encoded as standard APRS messages:

### All Chat → APRS Bulletin
```
KN6ZPL-7>APRS,WIDE1-1,WIDE2-1::BLN1     :message text here
```

### Direct Message → APRS DM
```
KN6ZPL-7>APRS,WIDE1-1,WIDE2-1::KN6YYY   :message text here{42
```

| Field | Description |
|-------|-------------|
| `::` | APRS message data type (double colon) |
| Addressee | 9-char space-padded recipient callsign. `BLN1` = bulletin (broadcast). |
| Message | Free text after the `:` |
| `{ID` | Optional message ID for ack/rej (e.g., `{42`) |

---

## Echo Suppression

### Own-Object Detection

When a cot_radio instance receives an APRS object with a `TAK:` comment containing its own callsign+SSID, it drops the packet as an echo of its own transmission.

### Multi-Gateway Coordination

Each cot_radio instance MUST use a unique SSID (0-15) within its coverage area. The SSID is used for echo suppression — if two gateways share the same callsign+SSID, they will suppress each other's objects.

---

## COT Type Reference

The COT_TYPE field uses MIL-STD-2525C type codes:

| COT Type | Meaning |
|----------|---------|
| `a-f-G-U-C` | Friendly ground unit, combat |
| `a-f-G-U-C-I` | Friendly ground unit, infantry |
| `a-h-G` | Hostile ground |
| `a-n-G` | Neutral ground |
| `a-u-G` | Unknown ground |
| `a-f-A-M-F` | Friendly air, military, fixed-wing |
| `a-f-A-M-H` | Friendly air, military, rotary-wing |

The full COT type determines both the affiliation (friendly/hostile/neutral/unknown) and the MIL-STD-2525 symbology on TAK displays.

---

## Compatibility

- **Standard APRS clients** (YAAC, APRSIS32, Xastir, aprs.fi): display TAK objects as normal APRS objects with the selected symbol. The TAK comment field appears as regular comment text.
- **cot_radio instances**: parse the TAK comment field to reconstruct full COT entities with correct callsign, team, affiliation, and custom icon.
- **ATAK/WinTAK**: receive TAK entities via cot_radio's multicast COT emission, not directly from APRS.

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-04-17 | Initial specification |
