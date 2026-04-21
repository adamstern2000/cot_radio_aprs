# TAK-APRS Protocol Extension

**Version:** 1.3
**Date:** 2026-04-20
**Author:** cot_radio project

## Overview

This document describes how cot_radio encodes TAK (Team Awareness Kit) entity data within standard APRS packets. The encoding uses the APRS Object format with a structured comment field that carries TAK-specific metadata for round-trip fidelity between TAK and APRS systems.

Any APRS client or gateway can decode these packets as standard APRS objects. The TAK metadata in the comment field is optional — clients that don't understand it simply display the object normally.

---

## Packet Format

### APRS Object (Position)

TAK entities are transmitted as APRS Objects (not position reports):

```
ORIGINATOR>APRS,PATH:;NAME     *DDHHMMzDDMM.MMN/DDDMM.MMW[TAK:GATEWAY:CALLSIGN:TEAM:COT_TYPE:ICONSETPATH
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
KN6ZPL-7>APRS,WIDE1-1,WIDE2-1:;HAWK     *170230z3406.36N/11818.12W[TAK:BASE:HAWK:Cyan:a-f-G-U-C
```

**Marker with custom icon:**
```
KN6ZPL-7>APRS,WIDE1-1,WIDE2-1:;ANTELOPE *170230z3404.84N/11737.01W[TAK:BASE:ANTELOPE 1:White:a-f-G-U-C:34ae1613-9645-4222-a9d2-e5f243dea2865/Animals/antelope.png
```

**Killed (deleted) object:**
```
KN6ZPL-7>APRS,WIDE1-1,WIDE2-1:;HAWK     _170235z3406.36N/11818.12W[TAK:BASE:HAWK:Cyan:a-f-G-U-C
```

---

## TAK Comment Field

The comment field begins with the `TAK:` prefix, followed by colon-separated metadata fields:

```
TAK:GATEWAY:CALLSIGN:TEAM:COT_TYPE[:ICONSETPATH]
```

### Fields

| # | Field | Required | Description |
|---|-------|----------|-------------|
| 1 | `TAK` | Yes | Literal prefix — identifies this as a TAK-encoded object |
| 2 | GATEWAY | Yes | Tactical callsign of the cot_radio gateway that emitted this packet (`identity.tactical_callsign` in settings). Must be unique across all cot_radio gateways in the network. Used by receivers to detect own emissions, attribute traffic, and de-duplicate. |
| 3 | CALLSIGN | Yes | Full TAK callsign of the entity (not truncated). May differ from the 9-char object name. |
| 4 | TEAM | Yes | TAK team color (e.g., `Cyan`, `White`, `Red`, `Green`). Empty string if no team. |
| 5 | COT_TYPE | Yes | Full COT type string (MIL-STD-2525 symbology). Examples: `a-f-G-U-C` (friendly ground unit), `a-h-G` (hostile ground), `a-n-G` (neutral ground). |
| 6 | ICONSETPATH | No | ATAK iconset path for custom icon rendering. Format: `{iconset_uid}/{group}/{filename.png}`. Only present for markers with custom icons. |

### Parsing Rules

1. Check if the comment field starts with `TAK:`
2. Split on `:` — fields are positional
3. Field 1 is always `TAK` (literal)
4. Field 2 is the gateway tactical callsign
5. Field 3 is the full entity callsign
6. Field 4 is the team color (may be empty string between colons)
7. Field 5 is the COT type
8. Field 6 (optional) is the iconset path — if present, it contains `/` characters which should NOT be confused with the `:` delimiter

### Example Parsing

```
TAK:BASE:HAWK:Cyan:a-f-G-U-C
  → gateway    = "BASE"
  → callsign   = "HAWK"
  → team       = "Cyan"
  → cot_type   = "a-f-G-U-C"
  → iconsetpath = (none)

TAK:RELAY-7:ANTELOPE 1::a-u-G:34ae1613.../Animals/antelope.png
  → gateway    = "RELAY-7"
  → callsign   = "ANTELOPE 1"
  → team       = "" (empty)
  → cot_type   = "a-u-G"
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

TAK chat messages are encoded as standard APRS messages with a TAK sender prefix in the body. Two prefix formats coexist:

- **v1.2 object/DM prefix** — `TAK:GATEWAY:SENDER:body` — used for DMs cross-TAK where the receiver must rebuild a specific senderCallsign.
- **v1.3 relay prefix** — `TAK:HHUUUUUU:SENDER:body` — used for bidirectional group-chat relay (TAK All Chat ↔ APRS BLN1). The second field is a 7-char hex token (1 hop digit + 6 hex UUID) for loop prevention and multi-gateway coordination.

Receivers distinguish the two by the shape of the second field: **exactly 7 hex chars = v1.3 relay**; anything else = v1.2 (or v1.0 plain APRS).

### All Chat → APRS Bulletin (v1.3 relay)
```
KN6ZPL-7>APRS,WIDE1-1,WIDE2-1::BLN1     :TAK:0a1b2c3:HAWK:hello world
```

### Direct Message → APRS DM (v1.2)
```
KN6ZPL-7>APRS,WIDE1-1,WIDE2-1::KN6YYY   :TAK:BASE:HAWK:reply text{42
```

| Field | Description |
|-------|-------------|
| `::` | APRS message data type (double colon) |
| Addressee | 9-char space-padded recipient callsign. `BLN1` = bulletin (broadcast). |
| Message | `TAK:GATEWAY:SENDER:body` — gateway tactical + original sender + body |
| `{ID` | Optional message ID for ack/rej (e.g., `{42`) |

### TAK Body Prefix

The message body begins with `TAK:GATEWAY:SENDER:` where:

- **GATEWAY** — tactical callsign of the cot_radio gateway that emitted this packet (`identity.tactical_callsign`). Lets receivers detect own emissions and route attribution correctly.
- **SENDER** — original TAK callsign (or mesh node short name) of whoever sent the chat in the originating TAK environment.

Receiving cot_radio gateways parse this prefix and reconstruct the GeoChat COT with `senderCallsign=SENDER`, so the message displays as coming from the original sender — not from the gateway's ham callsign.

#### Parsing Rules

1. Strip the addressee header (`::ADDRESSEE:`) per standard APRS.
2. If the body starts with `TAK:`, split on `:` (max 4 parts).
3. Field 2 is the gateway tactical callsign.
4. Field 3 is the original sender callsign — use as `senderCallsign` in the GeoChat COT.
5. Field 4 is the message body — put in `<remarks>`.
6. If the body does not start with `TAK:`, treat as a normal APRS message and use the gateway callsign as the sender (legacy / non-cot_radio APRS clients).

#### Example Parsing

```
TAK:BASE:HAWK:Hello team
  → gateway        = "BASE"
  → senderCallsign = "HAWK"
  → body           = "Hello team"

TAK:RELAY-7:A#1:reply from mesh
  → gateway        = "RELAY-7"
  → senderCallsign = "A#1"
  → body           = "reply from mesh"

Plain APRS message from a non-cot_radio client → senderCallsign = origin gateway
```

### Routing Rules (TAK → APRS)

The cot_radio implementation maps the TAK GeoChat `chatroom` field to the APRS addressee as follows:

| TAK chatroom | APRS addressee | Prefix | Notes |
|--------------|----------------|--------|-------|
| `All Chat Rooms` | `BLN1` | v1.3 relay | Bulletin broadcast; hop=0, fresh UUID |
| Valid amateur callsign (e.g. `KN6YYY-9`) | recipient callsign | v1.2 object | DM to APRS station |
| Tactical TAK callsign (e.g. `HAWK`, `FALCON`) | recipient callsign | v1.2 object | DM cross-TAK; receiving cot_radio rebuilds GeoChat to that callsign |
| Mesh node short name (recipient UID begins with `MESH-`) | (dropped) | — | No APRS equivalent — not transmitted |

---

## v1.3 Relay Prefix — Group-Chat Bidirectional Relay

The v1.3 relay prefix supports **multi-gateway group chat** (TAK All Chat ↔ APRS BLN1) with loop prevention, hop limits, and multi-part reassembly for bodies longer than the APRS 67-char message cap.

### Prefix Format

```
TAK:HHUUUUUU:SENDER:BODY
```

| Segment | Length | Format | Purpose |
|---------|--------|--------|---------|
| `TAK:` | 4 | Literal | Identifies a cot_radio extension |
| `HH` | 1 hex char | `0`–`9`, `a`–`f` | Hop count (0–15). Emitter sets 0; each re-relay increments by 1. Dropped once count ≥ `bridge_max_hops`. |
| `UUUUUU` | 6 hex chars | `0`–`9`, `a`–`f` | UUID6 — random per-message identifier for cross-gateway loop prevention |
| `:` | 1 | Literal | Delimiter |
| `SENDER` | 1+ | any non-`:` | Original TAK callsign (or mesh short-name) of whoever said it in TAK |
| `:` | 1 | Literal | Delimiter |
| `BODY` | 0+ | any | The message. May start with a multi-part marker (see below). |

**Identification regex:** `^TAK:([0-9a-fA-F]{7}):([^:]+):(.*)` with DOTALL. The 7-hex-char second field is the distinguishing feature — parsers check this shape first; if it doesn't match, fall back to v1.2 parsing (`TAK:GATEWAY:SENDER:body` with arbitrary GATEWAY).

### Multi-Part Marker

Bodies longer than 60 characters are split across multiple APRS messages. Each segment's body is prefixed with a one-based count marker:

```
(n/N) segment text
```

Where `n` is the 1-based segment number and `N` is the total segment count. Both segments share the same UUID so receivers can reassemble:

```
KN6ZPL-7>APRS::BLN1     :TAK:0a1b2c3:HAWK:(1/2) first part of a long message that doesn't fit
KN6ZPL-7>APRS::BLN1     :TAK:0a1b2c3:HAWK:(2/2) and here is the second part
```

**Segment limit:** 60 chars body + 15 chars prefix ≈ 67 total. Max 5 segments per message (longer bodies are ellipsis-truncated at the emitter).

### Receiver Behaviour

1. **UUID de-dup.** Maintain an LRU cache `{uuid → first_seen_ts}`, TTL = `uuid_cache_ttl_s` (default 300). If a UUID has been seen within TTL, drop the duplicate.
2. **Hop-count drop.** If `hop ≥ bridge_max_hops` (default 2), do not re-relay — deliver locally only.
3. **Reassembly.** Group segments by UUID; deliver to TAK when all parts arrive or after `reassembly_timeout_s` (default 30), inserting `[part N missing]` for gaps.
4. **Re-relay.** If the gateway has `bridge_tak_broadcast_to_aprs=true` and `hop < bridge_max_hops`, retransmit the segment(s) to APRS with hop+1 and the **same UUID**. The UUID is pre-observed in the local cache before emission so the gateway does not re-relay its own output.
5. **Origin display.** Delivered group-chat bodies are prefixed with `<SENDER> | APRS <class>: ` in the TAK `<remarks>` (where class ∈ `Bulletin`, `NWS`, `Announcement`, `APRS`). Operators can disable via `prefix_group_origin=false`.

### Emitter Behaviour (TAK All Chat → APRS)

When a TAK chat packet arrives with `chatroom="All Chat Rooms"`:

1. Generate a fresh `uuid6` (6 random hex chars) and set hop `0`.
2. Split the body into 60-char segments on space boundaries where possible; ellipsis-truncate to 5 segments max.
3. Prepend `(n/N) ` to each segment.
4. For each segment, emit `TAK:0UUUUUU:SENDER:(n/N) body` to APRS addressee `BLN1`.
5. Pre-observe the UUID in the local cache so the re-relay path does not re-emit own traffic.

### Loop Prevention Guarantees

- **Single gateway, echoed via digipeater:** UUID cache catches it on the first receive.
- **Two gateways in mutual view:** Gateway A emits (hop=0). Gateway B receives, delivers, re-relays (hop=1). Gateway A receives B's packet, UUID matches cache — dropped. Gateway B receives its own packet, UUID matches — dropped.
- **Three gateways chained (A→B→C):** A emits hop=0; B re-emits hop=1; C re-emits hop=2. If `bridge_max_hops=2`, C delivers but does not re-relay.

### Default Behaviour for APRS-to-APRS Chatter

v1.3 relay is **one-way by default** for plain APRS-to-APRS DMs: a DM between two amateur callsigns (neither of which has a live TAK equivalent) is **not** bridged to TAK unless the operator sets `bridge_aprs_chatter=true`. This keeps unrelated third-party ham conversations off TAK screens.

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
| 1.1 | 2026-04-17 | Chat: added `TAK:SENDER:` body prefix so original TAK/mesh sender is preserved across the bridge. Added explicit routing rules table for chatroom → APRS addressee mapping. Mesh-targeted DMs (recipient UID begins `MESH-`) are dropped. |
| 1.2 | 2026-04-17 | Added `GATEWAY` field as the second element of every TAK: prefix on both objects and chat. GATEWAY is the emitting gateway's `identity.tactical_callsign` — must be unique across the network. Lets receivers detect own emissions, attribute traffic, and de-duplicate. Object format: `TAK:GATEWAY:CALLSIGN:TEAM:COT_TYPE[:ICONSETPATH]`. Chat format: `TAK:GATEWAY:SENDER:body`. Backwards compatible: receivers should accept v1.1 format (no GATEWAY) and v1.0 format (no TAK: prefix). |
| 1.3 | 2026-04-20 | Added **v1.3 relay prefix** for bidirectional group chat (TAK All Chat ↔ APRS BLN1). Format `TAK:HHUUUUUU:SENDER:body` where `HH` is a 1-char hop count and `UUUUUU` is a 6-char UUID6. Adds multi-part marker `(n/N)` for bodies > 60 chars, hop-count limit (default 2), UUID cache for cross-gateway loop prevention (default 300 s TTL), reassembly timeout (default 30 s), and origin-labeled body prefix `<SENDER> | APRS <class>: ` on delivery. v1.2 object/DM format is unchanged and coexists with v1.3 on the wire — receivers distinguish by the shape of field 2 (exactly 7 hex chars = v1.3 relay). APRS-to-APRS DMs between third-party hams are not bridged to TAK unless `bridge_aprs_chatter=true`. |
