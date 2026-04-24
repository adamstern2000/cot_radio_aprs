# TAK-APRS Protocol Extension

**Version:** 2.0
**Date:** 2026-04-24
**Status:** Current. Breaking change vs v1.x — no backwards compatibility.

This document is **self-contained**. Every code on the wire is defined here, in plain English, so any licensed amateur radio operator can decode what they hear on the air without consulting external sources. This satisfies FCC Part 97 §97.113(a)(4) which prohibits "codes or ciphers intended to obscure the meaning" of a transmission. Compressed is not obscured — these tables make the meaning of every code unambiguous.

---

## 1. Why v2.0

Amateur radio APRS channels are bandwidth-limited. HF digital modes are even more so. v1.x of this protocol used full English names (`TAK:ST1:FALCON:Cyan:a-f-G-U-C`) = 29 characters for a basic situational-awareness beacon. v2.0 compresses the same information to 19 characters while keeping every value decodable from this document.

Typical savings:

- SA (team member beacon): 47 bytes → 19 bytes
- Marker with custom icon: 70-100 bytes → 24 bytes
- Chat bulletin: 22 bytes → 20 bytes

v1.x and v2.0 are not interoperable. Operators upgrade all gateways on their network at once.

---

## 2. Object Format (Situational Awareness + Markers)

Object packets follow the standard APRS Object format. The APRS "comment" field carries the TAK metadata.

```
ORIGINATOR>APRS,PATH:;NAME     *DDHHMMzDDMM.MMN/DDDMM.MMW<s><c>TAK<NN><T><R><TTTTTT>:CALLSIGN[:ICON]
                                                               └────── packed prefix ──────┘
```

- **ORIGINATOR** — the ham radio callsign + SSID of the gateway sending this packet (FCC-licensed callsign, required for Part 97 identification).
- **PATH** — standard APRS digipeater path, typically `WIDE1-1,WIDE2-1`.
- **NAME** — the object name, exactly 9 characters, right-padded with spaces. This is the TAK callsign with periods stripped and truncated to 9 chars.
- **DDHHMMz** — APRS timestamp (day, hour, minute, UTC).
- **DDMM.MMN/DDDMM.MMW** — latitude/longitude in standard APRS format.
- **`<s>`** — APRS symbol table (`/` or `\`).
- **`<c>`** — APRS symbol code character.
- Everything after that is the v2.0 packed prefix (fixed-width) + optional variable fields.

### 2.1 The Packed Prefix (13 characters)

| Position | Width | Name | What it is |
|---|---|---|---|
| 1-3 | 3 | `TAK` | Literal prefix. Marks this packet as carrying v2.0 TAK metadata. |
| 4-5 | 2 | `NN` | Station number (2 digits, `00`-`99`). The gateway's self-assigned station number. Each gateway in a network picks a unique number (see §7 echo suppression). |
| 6 | 1 | `T` | Team color letter (`A`-`N`, see §2.2). |
| 7 | 1 | `R` | Role letter (`A`-`Z`, see §2.3). |
| 8-13 | 6 | `TTTTTT` | COT type (entity classification, see §2.4). Exactly 6 characters, lowercase letters and underscores. |

After the 13-character packed prefix, there is a `:` delimiter, then the variable-length CALLSIGN field, optionally followed by another `:` and an ICON code.

### 2.2 Team Color — Single Letter `T`

The team color letter serves **two purposes** depending on the packet's COT type:

- **For situational awareness beacons (`a-*` types):** the team the operator belongs to. Rendered as the team-color affiliation icon (person-in-color on TAK clients).
- **For markers (`b-*` and `u-*` types):** the color tint applied to the custom icon. A marker with icon `D054` (car) and team letter `E` renders as a red-tinted car.

One letter, two uses — saves a separate "icon color" field on the wire.

| Letter | Color name | RGB hex | ARGB (TAK wire format, signed 32-bit int) |
|---|---|---|---|
| A | White       | `#FFFFFF` | -1         |
| B | Yellow      | `#FFFF00` | -256       |
| C | Orange      | `#FF8C00` | -7636      |
| D | Magenta     | `#FF00FF` | -65281     |
| E | Red         | `#FF0000` | -65536     |
| F | Maroon      | `#7F0000` | -8454144   |
| G | Purple      | `#800080` | -8388480   |
| H | Dark Blue   | `#000080` | -16777088  |
| I | Blue        | `#0000FF` | -16776961  |
| J | Cyan        | `#00FFFF` | -16711681  |
| K | Teal        | `#008080` | -16744320  |
| L | Green       | `#00FF00` | -16711936  |
| M | Dark Green  | `#008000` | -16744448  |
| N | Brown       | `#A52A2A` | -5952982   |

**Null value:** the character `_` (underscore) at position 6 means "no team color / no tint specified." For SA this is treated as `Green` (default team) with a warning. For markers this means the icon renders with no color tint (the icon's natural colors). Emitters MAY use `_` when the source entity has no assigned team.

If a receiver sees a letter outside `A`-`N` and not `_` (currently `O`-`Z` are reserved for future colors), it substitutes `Green` (letter `L`) and logs a warning.

### 2.3 Role — Single Letter `R`

The operator's role on the team. Rendered as a text label and affects icon variant on TAK clients.

| Letter | Role name | Meaning |
|---|---|---|
| M | Team Member | Default. Regular participant in the team. |
| L | Team Lead | Leader of the team or unit. |
| H | HQ | Headquarters / command element. Coordinates multiple teams. |
| R | RTO | Radio Telephone Operator. Handles communications. |
| K | K9 | Dog handler (canine team). |
| F | Forward Observer | Observation / reconnaissance / spotting. |
| S | Sniper | Designated marksman / sharpshooter. |
| D | Medic | Combat medic / emergency medical technician. |

If a receiver sees a letter outside this table, it substitutes `Team Member` (letter `M`) and logs a warning.

### 2.4 COT Type — 6 Characters `TTTTTT`

The COT type is the classification of the entity being reported. COT (Cursor on Target) uses a dot-separated hierarchy like `a-f-G-U-C` — "atom, friendly, ground, unit, combat". v2.0 strips the dashes and pads with underscores to exactly 6 characters.

#### 2.4.1 Encoding Rule

1. Start with the canonical dashed string (e.g. `a-f-G-U-C`).
2. Remove all `-` characters (`afGUC`).
3. Lowercase the result (`afguc`).
4. Right-pad with `_` to exactly 6 characters (`afguc_`).
5. If the stripped string exceeds 6 characters, truncate to 6 (losing the final specificity). This is rare — only the deepest sub-classifications exceed 6.

#### 2.4.2 Position Meanings (for interpreting any code)

Position 1 — **entity class**:

| Char | Meaning |
|---|---|
| `a` | Atom: a unit, person, or vehicle with a position (SA beacons, contacts) |
| `b` | Bit: a momentary event or alert (emergency, spot marker) |
| `u` | User graphic: a drawn shape or route (polygon, line, rectangle, circle, route) |
| `t` | Tasking / orders |
| `r` | Reply / acknowledgment |

Position 2 (for `a-*` entities) — **affiliation**:

| Char | Meaning |
|---|---|
| `f` | Friendly |
| `h` | Hostile |
| `n` | Neutral |
| `u` | Unknown |
| `p` | Pending (not yet identified) |
| `s` | Suspect (probably hostile, not confirmed) |
| `a` | Assumed friend |

Position 3 (for `a-*-X`) — **battle dimension**:

| Char | Meaning |
|---|---|
| `G` | Ground (land-based) |
| `A` | Air (aircraft) |
| `S` | Sea surface (ship) |
| `U` | Subsurface (submarine) |
| `P` | Space |
| `F` | Special forces |

Position 4+ (for `a-*-G-*`) — **function, ground**:

| Char | Meaning |
|---|---|
| `U` | Unit (organized group or person) |
| `E` | Equipment (vehicle or machine) |
| `I` | Installation (fixed facility) |

Position 5+ (for `a-*-G-U-*`, ground units) — **specialty**:

| Char | Meaning |
|---|---|
| `C` | Combat (infantry, armor) |
| `S` | Combat Support |
| `X` | Combat Service Support (logistics) |

Position 6+ (for `a-*-G-U-C-*`, ground combat) — **unit type**:

| Char | Meaning |
|---|---|
| `I` | Infantry |
| `A` | Armor |
| `F` | Artillery |
| `R` | Reconnaissance |

#### 2.4.3 Reference Table — Every COT Type This Protocol Emits

This table is the complete set of COT types cot_radio_aprs v2.0 encodes on the wire. Receivers map the 6-char encoding to canonical form using this table.

| 6-char | Canonical dashed | Plain English |
|---|---|---|
| `afguc_` | a-f-G-U-C | Friendly ground unit, combat (infantry, default for team members) |
| `afguci` | a-f-G-U-C-I | Friendly ground infantry (specific) |
| `afgevc` | a-f-G-E-V-C | Friendly ground civilian vehicle (car) |
| `afgevm` | a-f-G-E-V-M | Friendly ground military vehicle |
| `afamf_` | a-f-A-M-F | Friendly aircraft, military, fixed-wing |
| `afamh_` | a-f-A-M-H | Friendly aircraft, military, helicopter |
| `afscl_` | a-f-S-C-L | Friendly ship, combat, large |
| `afscm_` | a-f-S-C-M | Friendly ship, combat, medium |
| `afscs_` | a-f-S-C-S | Friendly ship, combat, small |
| `ahg___` | a-h-G | Hostile ground (generic) |
| `ang___` | a-n-G | Neutral ground |
| `aug___` | a-u-G | Unknown ground |
| `apg___` | a-p-G | Pending ground (not yet identified) |
| `asg___` | a-s-G | Suspect ground (probably hostile) |
| `baotbl` | b-a-o-tbl | Emergency alert — 911 Alert (highest priority) |
| `baopan` | b-a-o-pan | Emergency alert — Ring The Bell |
| `baoabn` | b-a-o-abn | Emergency alert — Absence Notification |
| `bmpsm_` | b-m-p-s-m | Marker — spot (a placed point of interest) |
| `udf___` | u-d-f | User-drawn polygon or freehand line |
| `udr___` | u-d-r | User-drawn rectangle |
| `udc___` | u-d-c | User-drawn circle or ellipse |
| `urb___` | u-r-b | Range ring / bullseye |
| `bmr___` | b-m-r | Route (multi-waypoint path) |

Emergency is encoded via the COT type only — the first three characters `bao` mark the packet as an emergency alert. There is no separate emergency flag field.

### 2.5 Callsign

After the 13-character packed prefix and a `:` delimiter, the full TAK callsign follows. This is the entity's display name in TAK (may differ from the 9-character APRS object name, which is truncated).

Allowed characters: letters, digits, space, hyphen, period. Callsign ends at the next `:` (if an ICON follows) or at the end of the APRS comment.

### 2.6 Icon (Optional)

Markers and shapes MAY carry an optional 4-character icon code after another `:` delimiter. Situational-awareness beacons (any `a-*` COT type) MUST NOT carry an icon — they render using only the team color and COT type symbology.

Icon format:

```
<iconset-code><3-char-id>
```

- `<iconset-code>` is a single character (A-Z) identifying the iconset. 11 iconsets defined in v1.0 (see §3).
- `<3-char-id>` is a 3-character base-36 alphanumeric ID (0-9, A-Z) within that iconset. Range `000` through `ZZZ` = 46,656 unique icons per iconset. IDs under 1,000 use pure decimal digits (e.g. `054`); IDs 1,000 and above use alphanumeric (e.g. `1AB`).

Example: `D0QX` → the antelope icon from the Default ATAK iconset (the actual ID for antelope in the canonical dict).

If an emitter has an iconsetpath not in the dictionary, it MAY emit the literal full path instead of a 4-char code. Receivers accept either format.

---

## 3. Iconset Dictionary

The canonical iconset dictionary — **every icon code decodable on v2.0 wire** — is published in this repository as a separate JSON file because its size (4,186 icons in v1.0) makes inlining impractical. The README here is the human-facing index; the JSON is the authoritative machine-readable source.

**Canonical dictionary file:** [`iconset_dict_v1.json`](./iconset_dict_v1.json)

Each entry has the form:

```json
"D":{
  "uuid":"34ae1613-9645-4222-a9d2-e5f243dea2865",
  "name":"Default",
  "icons":{
    "000":{"path":"Animals/ant.png","desc":"ant"},
    "001":{"path":"Animals/antelope.png","desc":"antelope"},
    ...
  }
}
```

The `path` field is the file path within the iconset (which combined with the iconset UUID forms the full iconsetpath stored in TAK COT XML). The `desc` field is the plain-English description, derived from the filename (underscores and hyphens → spaces). Any licensed amateur operator can download the JSON and decode any v2.0 icon code on the air.

### 3.1 Iconset Letter-Code Assignments (v1.0)

| Code | Iconset name | UUID | Icon count |
|---|---|---|---|
| `A` | APRS (non-tak) | `APRS-NONTAK` | 188 |
| `D` | Default (ATAK-CIV) | `34ae1613-9645-4222-a9d2-e5f243dea2865` | 821 |
| `E` | FEMA Icons | `f8f7f666-8b28-4b57-9fbb-e48e61d33b79` | 42 |
| `F` | FalconView | `67441c4a4924b8812e0f3e2191a2228a` | 489 |
| `G` | Generic Icons | `ad78aafb-83a6-4c07-b2b9-a897a8b6a38f` | 657 |
| `I` | Incident Management | `f3723f30315ea30f2f4b9101556772e2` | 12 |
| `L` | Google | `f7f71666-8b28-4b57-9fbb-e38e61d33b79` | 96 |
| `M` | OSM | `6d781afb-89a6-4c07-b2b9-a89748b6a38f` | 347 |
| `O` | GeoOps | `83198b4872a8c34eb9c549da8a4de5a28f07821185b39a2277948f66c24ac17a` | 68 |
| `P` | Public Safety Air | `ba0f5e196dfcee47a00ee3f4a494d64d` | 50 |
| `R` | Responder Icons | `8a2105090b5f30fd2cefc64f3eae8ad3` | 1,416 |

**Total v1.0:** 4,186 icons across 11 iconsets.

### 3.2 ID Format

Each icon ID is a **3-character base-36 alphanumeric string** (0-9, A-Z).

- Range `000`-`ZZZ` = 46,656 unique IDs per iconset (more than enough; largest iconset in v1.0 is 1,416 icons).
- IDs below 1,000 look like plain decimal digits (`001`, `042`, `999`).
- IDs at or above 1,000 use alphanumeric base-36 (`1000` → `ZZZ`; `1000` decimal = `RS` base-36 padded to `0RS` = 3 chars).
- Ordering within an iconset is alphabetical by (group, filename). Deterministic so IDs never shift as the dictionary is extended.

### 3.3 Reserved Letters

- **Codes in use (v1.0):** `A`, `D`, `E`, `F`, `G`, `I`, `L`, `M`, `O`, `P`, `R`.
- **Codes available for future canonical iconsets:** `B`, `C`, `H`, `J`, `K`, `N`, `Q`, `S`, `T`, `U`, `V`, `W`.
- **Codes reserved for operator-custom iconsets:** `X`, `Y`, `Z`. Operators using these codes on amateur frequencies MUST publish their custom `iconset_dict.user.json` additions (filename + English description) to every operator on their network, preserving Part 97 transparency.

### 3.4 Versioning Rules

- **Adding a new icon to an existing iconset** → bump minor version (`1.0` → `1.1`). Existing IDs never change.
- **Adding a new iconset code** → bump minor version. Existing codes never change.
- **Renumbering or removing an icon ID** → major-version bump (`1.x` → `2.0`). Invalidates archived wire traffic; only done in emergencies.

The version number in force at time of transmission is published at the top of `iconset_dict_v1.json` (currently **v1.0**, dated 2026-04-24).

---

## 4. Chat Format (Messages and Bulletins)

Chat packets use the standard APRS message format:

```
ORIGINATOR>APRS,PATH::ADDRESSEE :TAK:<H><UUUU>:<SENDER>:<body>
```

- **ORIGINATOR** — gateway's ham callsign + SSID.
- **ADDRESSEE** — 9-character space-padded recipient. `BLN1` = bulletin (broadcast to all); a specific callsign = direct message.
- **TAK:** — literal prefix marking this as a v2.0 chat packet.
- **`<H>`** — hop count, 1 hex character (`0`-`F`). The gateway that first emits the packet sets hop=`0`. Each gateway that re-relays increments hop by 1. A gateway at hop ≥ `bridge_max_hops` (default 2) delivers locally only and does not re-transmit.
- **`<UUUU>`** — UUID, 4 hex characters. Random per-message identifier for loop prevention.
- **`<SENDER>`** — the original sender's display name (TAK callsign or mesh node short-name). May contain letters, digits, space, hyphen, period — ends at the next `:`.
- **`<body>`** — the chat message text.

### 4.1 Addressee Routing (How TAK Chatrooms Map to APRS Addressees)

| TAK chatroom | APRS addressee | Notes |
|---|---|---|
| `All Chat Rooms` | `BLN1` | Bulletin broadcast. Hop=0, fresh UUID. |
| Valid amateur callsign (e.g. `KN6YYY-9`) | The callsign, 9-char padded | Direct message to an APRS station. |
| TAK tactical callsign (e.g. `HAWK`, `FALCON`) | The callsign, 9-char padded | Direct message cross-TAK; receiving gateway rebuilds as a TAK GeoChat. |
| Mesh node short name (recipient UID begins `MESH-`) | (not transmitted) | No APRS equivalent — dropped at emit. |

### 4.2 Multi-Part Messages

Chat bodies longer than ~55 characters are split across multiple APRS messages. Each segment's body begins with a marker `(n/N) ` where `n` is the 1-based segment number and `N` is the total segment count. Both segments share the same UUID so receivers can reassemble:

```
KN6ZPL-7>APRS::BLN1     :TAK:0a1b2:HAWK:(1/2) first part of a long message
KN6ZPL-7>APRS::BLN1     :TAK:0a1b2:HAWK:(2/2) and here is the second part
```

Maximum 5 segments per message. Bodies that would exceed 5 segments are ellipsis-truncated at the emitter.

### 4.3 Loop Prevention

Each gateway keeps an in-memory cache of recently-seen UUIDs (default TTL 30 seconds, maximum 1000 entries). If a UUID has been seen within the window, the duplicate is dropped. Combined with the hop-count limit, this prevents relay loops in multi-gateway networks.

---

## 5. Configuration Changes for v2.0

Gateways running cot_radio update their `settings.json`:

- `identity.tactical_callsign` (string) → **`identity.station_number`** (integer, 0-99)
- `gating.uuid_cache_ttl_s` default 300 → **30**
- `igate.sa_comment_hint` (string) → removed / ignored (no hint tail in v2.0)

---

## 6. Echo Suppression

A cot_radio instance MUST drop packets that are echoes of its own transmissions:

1. **Own station number** — if the packed prefix's `NN` matches the gateway's own `station_number`, drop the packet.
2. **Own chat callsign** — if a chat packet's ORIGINATOR field matches the gateway's own ham callsign-SSID, drop.
3. **Peer echo** — if an inbound object's CALLSIGN is already observed on the gateway's local TAK multicast (via a short-TTL contact registry), drop. Prevents duplicate markers when two gateways share a LAN or a mesh bridge is in use.

---

## 7. Part 97 Compliance

Amateur radio operators using this protocol on FCC-licensed frequencies (USA) are bound by 47 CFR Part 97, specifically §97.113(a)(4) which prohibits "messages in codes or ciphers intended to obscure the meaning thereof."

v2.0 uses compressed codes (single letters for team color and role, 6-character COT types, 4-character icon references). These codes are **compressed** but not **obscured**:

- Every letter and every code is defined in this document, in plain English.
- This document is publicly published at <https://github.com/adamstern2000/cot_radio_aprs>.
- Any licensed operator can download this document and decode any v2.0 packet heard on the air.
- The iconset dictionary (§3) is complete — every icon ID has a canonical path and an English description published here.

Operator obligations when running v2.0 on amateur frequencies:

1. **Transmit your FCC callsign** in the AX.25 header of every frame. (Already required by APRS; v2.0 does not change this.)
2. **Use a station number tied to your licensed station** — operator discipline; two stations on one network each pick a unique `station_number` and document which physical station uses which number.
3. **Do not modify the v2.0 tables locally** in a way that differs from the canonical tables in this document. Local modification would cross from "compressed" into "obscured" and is not Part 97 compliant.
4. **If you use a custom iconset** (codes `X`, `Y`, `Z`), publish your custom iconset additions (filename + English description) to every operator on your network. The transparency requirement applies to custom content just as it does to canonical content.

v2.0 is in the same Part 97 category as existing APRS compression methods (Mic-E, Base91, compressed position) — efficient on-the-wire encoding that is publicly documented.

---

## 8. Version History

| Version | Date | Summary |
|---|---|---|
| 1.0 | 2026-04-17 | Initial spec. Human-readable fields, colons between every field. |
| 1.1 | 2026-04-17 | Added `TAK:SENDER:body` chat prefix so original sender preserved across bridge. |
| 1.2 | 2026-04-17 | Added GATEWAY field (tactical callsign) to every prefix. |
| 1.3 | 2026-04-20 | Bidirectional group-chat relay (TAK All Chat ↔ APRS BLN1) with UUID-based loop prevention. |
| 1.4 | 2026-04-20 | Human-readable hint tail appended to object comments for plain-APRS operator visibility. |
| **2.0** | **2026-04-24** | **Ground-up rewrite for airtime efficiency. Packed fixed-width prefix, single-letter codes for team/role, 6-character COT type, 4-character iconset IDs via published dictionary, 4-hex UUID with 30s cache TTL. No back-compat with v1.x. Fully self-contained — every code defined in this document for Part 97 transparency.** |
