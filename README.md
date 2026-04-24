# TAK-APRS Protocol Extension

**Version:** 2.0 (draft)
**Date:** 2026-04-24
**Author:** cot_radio project
**Status:** Breaking change vs v1.x — no backwards compatibility. Designed for bandwidth-constrained links (APRS RF, HF digital) where every byte on the wire matters.

## What changed in v2.0

v2.0 is a **ground-up rewrite** of the TAK-APRS encoding optimized for airtime. v1.x used human-readable fields with many colon delimiters (`TAK:ST1:FALCON:Cyan:a-f-G-U-C | DM:KN6ZPL-7 FALCON` = 47 bytes). v2.0 uses packed fixed-width fields, letter/numeric codes, and an iconset-dictionary lookup (`TAK01CMfguc_:FALCON` = 19 bytes for the same SA).

Major differences from v1.4:

| Aspect | v1.x | v2.0 |
|---|---|---|
| Gateway identity | `tactical_callsign` string (`ST1`, `BASE`) | `station_number` int 0–99 (2 digits) |
| Team color | full name string (`Cyan`, `Green`) | single letter A-N |
| Role | not encoded | single letter (M/L/H/R/K/F/S/D) |
| COT type | full MIL-STD string with dashes (`a-f-G-U-C`) | 6 chars stripped, lowercase, `_`-padded (`afguc_`) |
| Iconset | full UUID+path (60+ bytes) | 4-char code via dictionary (`D042`) |
| Delimiters | colons between every field | packed fixed-width prefix, one `:` before variable fields |
| Hint tail | `| DM:…` included | removed |
| Chat UUID | 6 hex (16M) | 4 hex (65K) |
| Chat UUID TTL | 300s | 30s |
| SA iconset | optional 6th field | **forbidden** — SA renders via team affiliation only |
| Backwards compat | required across v1.0-v1.4 | clean break; v1.x/v2.0 do not interoperate |

v1.x senders and v2.0 senders cannot share an APRS network — receivers must match the sender version.

---

## Part 97 (Amateur Radio) Compliance

The United States FCC Part 97 rules for amateur radio prohibit "codes and ciphers intended to obscure the meaning" of a transmission (§97.113(a)(4)). v2.0 uses compressed codes (letters for team/role, 6-char COT types, 4-char iconset IDs) — but **compressed is not obscured**: the codes are 1-to-1 mapped to their canonical meanings via tables published in this repository.

Any licensed operator can:

1. Download this repository.
2. Read the plaintext mapping tables (team colors, roles, COT types, iconsets).
3. Decode any v2.0 packet heard on the air.

This places v2.0 in the same legal category as APRS compression (Mic-E, Base91, APRS-101 compressed position) — efficient on-the-wire encoding that is publicly documented. Operators using v2.0 on amateur frequencies MUST:

- Transmit their FCC callsign in the AX.25 header of every frame (already true for APRS).
- Use a `station_number` tied to their licensed station (operator discipline).
- Not modify the v2.0 tables locally in a way that differs from the canonical published version (that would cross into obscuring meaning).

Operator-custom iconsets (codes `X`, `Y`, `Z`) are permitted because the operator's custom `iconset_dict.user.json` MUST be published and shared with anyone on the network — the transparency requirement stands.

---

## Design Principles

1. **Packed fixed-width prefix.** Known-width fields run together with no internal delimiters, parsed positionally.
2. **Letters and numbers over words.** Team, role, COT type, iconset — all compressed to short codes from a shared dictionary.
3. **SA is affiliation-only.** Situational-awareness beacons (`a-*` COT types) never carry a per-station iconset. They render via team color + MIL-STD symbology only. Icons are a marker/shape concept, not an SA concept.
4. **cot_radio owns the compression boundary.** Other services in the TAK suite (cot_bridge, cot_mesh, WinTAK/ATAK) see standard TAK COT XML with full paths and plain strings. cot_radio translates at the APRS boundary.

---

## Object Format (SA and Markers)

```
ORIGINATOR>APRS,PATH:;NAME     *DDHHMMzDDMM.MMN/DDDMM.MMW<s><c>TAK<NN><t><r><TTTTTT>:CALLSIGN[:ICON]
                                                         └─────── packed prefix, 13 chars fixed ───┘
```

### Packed Prefix (13 chars, no internal delimiters)

| Pos | Width | Field | Values |
|---|---|---|---|
| 1-3 | 3 | `TAK` | literal — identifies this as a v2.0 TAK-encoded comment |
| 4-5 | 2 | `NN` | station number, 2 digits `00`-`99`, zero-padded |
| 6 | 1 | `t` | team color, 1 letter `A`-`N` (see §Team Color Table) |
| 7 | 1 | `r` | role, 1 letter (see §Role Table) |
| 8-13 | 6 | `TTTTTT` | COT type, dashes stripped, lowercase, `_`-padded right (see §COT Type Encoding) |

### Variable-Length Fields (after `:` delimiter)

| Field | Description |
|---|---|
| `CALLSIGN` | Full TAK callsign. May contain spaces, dashes, digits, periods. May differ from the 9-char APRS object name. |
| `ICON` | Optional. `<iconset-code><3-digit id>` like `D042`. Absent for SA types (`a-*`). May be a literal full path as a fallback for unknown icons. |

### Field Formats

#### Team Color Table (14 entries + future)

The 14 standard TAK team colors as defined in the ATAK-CIV source. Each letter maps to a color name + the canonical ARGB hex used on TAK clients.

| Letter | Color | Hex (RGB) | ARGB (signed int — TAK wire) |
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

**Source:** Team color names are defined in the ATAK-CIV public repo at <https://github.com/deptofdefense/AndroidTacticalAssaultKit-CIV> (class `com.atakmap.android.user.PlacePointTool` and `TeamColor` enum).

Alphabetical ordering is arbitrary but stable; future TAK team colors append new letters (`O`, `P`, …, `Z`). Receivers that see an unknown letter SHOULD fall back to `Green` (the cot_radio default).

#### Role Table

The 8 standard TAK roles as defined in the ATAK-CIV source:

| Letter | Role | Description |
|---|---|---|
| M | Team Member | Default — regular team participant |
| L | Team Lead | Unit leader |
| H | HQ | Headquarters / command element |
| R | RTO | Radio Telephone Operator |
| K | K9 | Dog handler |
| F | Forward Observer | Observation / reconnaissance |
| S | Sniper | Designated marksman |
| D | Medic | Combat medic / EMT |

**Source:** Roles are defined in the ATAK-CIV public repo at <https://github.com/deptofdefense/AndroidTacticalAssaultKit-CIV>. The `role` attribute lives inside the CoT `<__group>` detail element alongside team color.

Receivers MUST default to `M` (Team Member) when the role letter is empty or unknown. Roles beyond the 8 defined here use the letter `U` (Unknown) and render as Team Member on strict-parsing clients.

#### COT Type Encoding

The COT type field is exactly 6 chars on the wire.

Encoding rules:

1. Strip all `-` characters from the canonical TAK type string.
2. Lowercase the result.
3. Right-pad with `_` to exactly 6 chars.
4. Truncate to 6 chars if longer — receiver will render a less-specific symbol but still decode affiliation.

**The canonical TAK type strings themselves derive from MIL-STD-2525C** (NATO Joint Military Symbology — the US DoD standard for battlefield graphics). The TAK dot-separated form (`a-f-G-U-C`) maps to the 15-character SIDC (Symbol Identification Code) at positions 1 (coding scheme), 2 (affiliation), 3 (battle dimension), and 4+ (function ID).

**Public references:**

- MIL-STD-2525C spec (DoD, freely downloadable): <https://www.jcs.mil/Portals/36/Documents/Doctrine/Other_Pubs/ms_2525c.pdf>
- NATO Joint Military Symbology — public overview: <https://en.wikipedia.org/wiki/NATO_Joint_Military_Symbology>
- TAK public CoT type catalog (canonical enumeration used by ATAK / WinTAK): <https://github.com/TAK-Product-Center/Server/tree/main/src/takserver-core/coreServer/conf/types>
- TAK Technical Overview (public PDF): <https://www.tak.gov>

Examples of the v2.0 6-char encoding:

| Canonical TAK type | MIL-STD-2525 description | 6-char encoding |
|---|---|---|
| `a-f-G-U-C` | friendly ground unit combat (infantry) | `afguc_` |
| `a-f-G-U-C-I` | friendly ground unit infantry | `afguci` |
| `a-f-G-U-C-I-M` | motorized infantry | `afguci` *(truncated, `M` lost)* |
| `a-f-G-E-V-C` | friendly civilian vehicle | `afgevc` |
| `a-h-G` | hostile ground (generic) | `ahg___` |
| `a-n-G` | neutral ground | `ang___` |
| `a-u-G` | unknown ground | `aug___` |
| `a-f-A-M-F` | friendly air fixed-wing military | `afamf_` |
| `a-f-A-M-H` | friendly air rotary-wing (helo) | `afamh_` |
| `a-f-S-C-L` | friendly surface combat large | `afscl_` |
| `b-a-o-tbl` | emergency alert — 911 Alert | `baotbl` |
| `b-a-o-pan` | emergency alert — ring the bell | `baopan` |
| `b-m-p-s-m` | marker — spot | `bmpsm_` |
| `u-d-f` | polygon / freehand line | `udf___` |
| `u-d-r` | rectangle | `udr___` |

Emergency is encoded via the COT type only (`baotbl`, `baopan`). There is no separate emergency flag.

**Decoding note:** The receiver reverses the process — strip trailing `_`, insert `-` between transitions (based on the canonical TAK hierarchy: `a-{affiliation}-{dim}-{function...}`). A 6-char encoding loses the dash positions, so receivers rebuild the canonical form from a lookup table. Unknown codes render as the base affiliation symbol (first 3 chars) with a warning.

#### Iconset Encoding

Markers and shapes may carry a 4-char icon code:

```
<iconset-code><3-digit-id>
```

- `<iconset-code>` is a single character identifying the iconset (A-Z, 0-9).
- `<3-digit-id>` is a zero-padded numeric ID for an icon within that iconset, `000`-`999`.

The ID → full iconset-path mapping is defined by the **canonical iconset dictionary published in this repo**:

**[`iconset_dict_v1.json`](./iconset_dict_v1.json)** ← authoritative reference

Ships with cot_radio; also lives here on GitHub for Part 97 transparency. Any licensed operator can read the dictionary and decode what's on the air.

Canonical iconset codes (v1.0 initial):

- `D` = Default ATAK iconset (UUID `34ae1613-9645-4222-a9d2-e5f243dea2865`) — ships with ATAK-CIV
- `F` = FEMA / Incident Management iconset (UUID `f3723f30315ea30f2f4b9101556772e2`) — public FEMA-published set
- Codes `A`-`W` (excluding `D`, `F` and others reserved canonical) available for future canonical additions
- Codes `X`, `Y`, `Z` reserved for operator-custom iconsets via an operator-maintained `iconset_dict.user.json` (MUST be synced across all cot_radio gateways in the network)

Examples:

| Wire | Canonical iconsetpath |
|---|---|
| `D042` | `34ae1613-9645-4222-a9d2-e5f243dea2865/Animals/antelope.png` |
| `D015` | `34ae1613-9645-4222-a9d2-e5f243dea2865/Transportation/car.png` |
| `F007` | `f3723f30315ea30f2f4b9101556772e2/Incident Management/command_post.png` |

**Fallback chain at emit:**

1. Try exact-match lookup of `(iconset UUID, filename)` in the canonical dict. On hit → emit `<code><id>`.
2. Miss → emit the literal full path (breaking the 4-char budget). Receiver still stores it. Note: this loses airtime efficiency and is a signal that the canonical dict should be extended.

**Fallback chain at receive:**

1. Wire starts with a letter + 3 digits matching a known iconset code → resolve to full path via dict.
2. Wire looks like a literal path (contains `/`) → treat as full path verbatim.
3. Otherwise → emit default icon for the COT type's affiliation + log unknown-icon warning.

**SA never carries an iconset.** Any SA-type (`a-*`) emission MUST omit the icon field. Parsers that receive an `a-*` event with an icon field MUST ignore the icon (render via affiliation only) and log a warning.

**Versioning:** `iconset_dict_v1.json` is additive-only within a major version. Adding a new icon bumps minor (`1.0` → `1.1`). Removing or renumbering is a major-version change (`v1` → `v2`) — which invalidates archived wire traffic and MUST NOT be done lightly. The current version number is published at the top of the dict file.

---

### Example Objects

#### SA (team member, no icon)
```
KN6ZPL-7>APRS,WIDE1-1,WIDE2-1:;FALCON   *241846z3406.36N/11818.12W[TAK01JMafguc_:FALCON
```
- station 01
- team J (Cyan)
- role M (Member)
- type `afguc_` (a-f-G-U-C)
- callsign `FALCON`

#### Marker (placed POI with a custom icon)
```
KN6ZPL-7>APRS,WIDE1-1,WIDE2-1:;FLAG1    *241846z3406.35N/11818.10W[TAK01AMbmpsm_:FLAG1:D042
```
- station 01
- team A (White)
- role M
- type `bmpsm_` (b-m-p-s-m — spot marker)
- callsign `FLAG1`
- icon `D042` (Default/Animals/antelope.png)

#### Killed (deleted) object
```
KN6ZPL-7>APRS,WIDE1-1,WIDE2-1:;FLAG1    _241850z3406.35N/11818.10W[TAK01AMbmpsm_:FLAG1:D042
```
The APRS object data type changes from `*` (live) to `_` (killed). TAK prefix is unchanged.

---

## Parsing

```python
import re

PREFIX_RE = re.compile(
    r'^TAK(?P<station>\d{2})'
    r'(?P<team>[A-N])'
    r'(?P<role>[A-Z])'
    r'(?P<cot_type>[a-z_]{6})'
    r':(?P<callsign>[^:]+)'
    r'(?::(?P<icon>.+))?$'
)

def parse_comment(comment: str) -> dict | None:
    m = PREFIX_RE.match(comment)
    if not m:
        return None
    return {
        'station_number': int(m.group('station')),
        'team_letter': m.group('team'),
        'role_letter': m.group('role'),
        'cot_type_6': m.group('cot_type'),
        'callsign': m.group('callsign'),
        'icon': m.group('icon'),  # None if absent
    }
```

The regex is strict: v2.0 packets are unambiguously identified by the 2-digit station, 1-letter team (A-N), 1-letter role (A-Z), 6-char COT type pattern. Anything else is either a v1.x packet (should be rejected since v2.0 is not back-compat) or a non-TAK APRS comment.

### Emergency Detection

Receivers key off the COT type field:

```python
if decoded['cot_type_6'] == 'baotbl':
    route_as_emergency(...)
```

Same as v1.x — no dedicated flag.

---

## Object Name Encoding (unchanged from v1.x)

The APRS object name is 9 chars, space-padded. TAK callsigns:

1. Strip all period (`.`) chars from the callsign.
2. Truncate to 9 chars.
3. Right-pad with spaces to exactly 9 chars.

The full unmodified callsign is preserved in the prefix's CALLSIGN field.

| TAK Callsign | Object Name | Prefix CALLSIGN |
|---|---|---|
| HAWK | `HAWK     ` | `HAWK` |
| FALCON | `FALCON   ` | `FALCON` |
| ANTELOPE 1 | `ANTELOPE ` | `ANTELOPE 1` |
| M.5.679477 | `M5679477 ` | `M.5.679477` |

---

## Chat Format (Messages / Bulletins)

Chat packets use the standard APRS message format (`::ADDRESSEE:body`). The body carries a v2.0 relay prefix:

```
KN6ZPL-7>APRS,WIDE1-1,WIDE2-1::BLN1     :TAK:<H><UUUU>:<SENDER>:<body>
```

| Token | Width | Description |
|---|---|---|
| `TAK:` | 4 | literal prefix |
| `H` | 1 hex | hop count, 0-F (see §Loop Prevention) |
| `UUUU` | 4 hex | UUID4 — random per-message, cross-gateway loop prevention |
| `:` | 1 | delimiter |
| `SENDER` | var | original TAK callsign (or mesh short-name) who said it |
| `:` | 1 | delimiter |
| `body` | var | chat body — may start with a multi-part marker (see below) |

**Identification regex:** `^TAK:(?P<token>[0-9a-fA-F]{5}):(?P<sender>[^:]+):(?P<body>.*)` with DOTALL. The 5-hex token distinguishes v2.0 chat from v1.x and from a v2.0 object prefix (which has no early colon — `TAK<digits>` not `TAK:<hex>`).

### Addressee Routing (TAK chatroom → APRS addressee)

| TAK chatroom | APRS addressee | Prefix shape | Notes |
|---|---|---|---|
| `All Chat Rooms` | `BLN1` | v2.0 relay | Bulletin broadcast; hop=0, fresh UUID |
| Valid amateur callsign (e.g. `KN6YYY-9`) | recipient callsign | v2.0 relay | DM to APRS station |
| Tactical TAK callsign (e.g. `HAWK`, `FALCON`) | recipient callsign | v2.0 relay | DM cross-TAK; receiving cot_radio rebuilds GeoChat to that callsign |
| Mesh node short name (recipient UID begins `MESH-`) | (dropped) | — | No APRS equivalent |

### Multi-Part Marker

Bodies longer than ~55 chars are split across multiple APRS messages. Each segment's body is prefixed with `(n/N)` one-based count:

```
KN6ZPL-7>APRS::BLN1     :TAK:0a1b2:HAWK:(1/2) first part of a long message
KN6ZPL-7>APRS::BLN1     :TAK:0a1b2:HAWK:(2/2) and here is the second part
```

Segments share the same UUID for reassembly. Max 5 segments; longer bodies are ellipsis-truncated at the emitter.

### Loop Prevention

1. **UUID cache.** Each gateway keeps an LRU cache of recently-seen UUIDs (default TTL **30 s**, max 1000 entries). If a UUID has been seen within the window, drop the duplicate.
2. **Hop count.** Each re-relaying gateway increments `H` by 1 (hex). At `H ≥ bridge_max_hops` (default 2) the gateway delivers locally only and does not re-relay.
3. **Own-sender drop.** Chat from the gateway's own ham callsign is always dropped (echoed self-transmission).

### Emitter Behaviour (TAK → APRS)

When a TAK chat packet arrives with `chatroom="All Chat Rooms"`:

1. Generate a fresh random 4-hex UUID, set hop `0`.
2. Pre-observe the UUID in the local cache so the re-relay path does not re-emit own traffic.
3. Split body into ~55-char segments on space boundaries; ellipsis-truncate to 5 segments max.
4. For each segment, emit `::BLN1     :TAK:0<UUUU>:<SENDER>:(n/N) body`.

For DMs (`chatroom` matches a callsign / TAK callsign):

1. Same UUID + hop logic.
2. Addressee = recipient callsign (9-char padded).
3. No multi-part support for DMs (APRS spec-level — DMs are typically short).

### Origin Display

When a v2.0 chat arrives and is delivered to the local TAK net as GeoChat, the remarks body is prefixed with `<SENDER> | APRS <class>: ` where class is one of `Bulletin`, `Announcement`, `APRS`. Operators can disable this via settings (`prefix_group_origin=false`).

---

## Iconset Dictionary (`iconset_dict.json`)

Ships with cot_radio at `/opt/cot_radio/iconset_dict.json`. Versioned; additive-only.

### Shape

```json
{
  "version": "1.0",
  "iconsets": {
    "D": {
      "uuid": "34ae1613-9645-4222-a9d2-e5f243dea2865",
      "name": "Default ATAK",
      "icons": {
        "000": "Animals/bear.png",
        "001": "Animals/cat.png",
        "042": "Animals/antelope.png",
        "015": "Transportation/car.png",
        ...
      }
    },
    "F": {
      "uuid": "f3723f30315ea30f2f4b9101556772e2",
      "name": "FEMA / Incident Management",
      "icons": {
        "001": "Incident/command_post.png",
        "002": "Incident/staging_area.png",
        ...
      }
    }
  }
}
```

### Operator Custom Iconsets (`iconset_dict.user.json`)

Operator-maintained, merges into the canonical dict at startup. Reserved codes `X`, `Y`, `Z`. Must be synced across all cot_radio gateways that need to exchange those icons.

### Versioning

- `version` key is a semver-ish string.
- Adding an icon → bump minor (`1.0` → `1.1`).
- Changing an existing ID → bump major (`1.x` → `2.0`). **AVOID** — invalidates archived traffic.
- Receivers of an unknown ID log a warning and render the default icon for the COT type affiliation.

---

## Configuration Changes (cot_radio settings.json)

Old `identity.tactical_callsign` (string) is replaced by `identity.station_number` (int 0–99).

```json
"identity": {
  "station_number": 1,
  "ssid": 7,
  ...
}
```

Admin UI label: "Station number" (was "Tactical callsign").

Other settings affected:

| Setting | Old default | New default | Notes |
|---|---|---|---|
| `gating.uuid_cache_ttl_s` | 300 | 30 | Aligned with v2.0 UUID width (4 hex) |
| `gating.bridge_max_hops` | 2 | 2 | Unchanged |
| `igate.sa_comment_hint` | active | **removed / ignored** | v2.0 drops the hint tail |

---

## Echo Suppression

### Own-Packet Detection

A cot_radio instance MUST drop an inbound packet whose wire prefix carries the instance's own `station_number`:

```python
if decoded['station_number'] == my_station_number:
    drop(packet, reason='own echo')
```

### Own-Callsign Chat Echo

A cot_radio MUST drop an inbound chat whose originating ham callsign (parsed from the APRS frame header) matches its own `callsign-SSID`:

```python
if frame.from_callsign.upper() == my_callsign_ssid.upper():
    drop(packet, reason='own chat echo')
```

### Peer-Echo Filter (contact_registry)

When an inbound object's packed prefix decodes to a `CALLSIGN` already observed on the local TAK multicast (via `contact_registry`), drop the packet. Prevents two cot_radios on the same LAN (or one cot_radio + cot_mesh bridging the same entity) from spawning duplicate markers.

---

## Compatibility with Plain APRS Clients

v2.0 packets are valid APRS objects / messages. Plain APRS clients (YAAC, APRSIS32, aprs.fi, Xastir):
- Display the object name and coordinates normally.
- Show the TAK prefix as opaque comment text (unlike v1.x which was semi-readable).

Since v2.0 encoding is optimized for machine parsing, plain-APRS operators lose the `TAK:BASE:DUSTY:Cyan:…` readable form. If readability on aprs.fi is an operational requirement, operators should deploy v1.4 instead.

---

## Version History

| Version | Date | Summary |
|---|---|---|
| 1.0 | 2026-04-17 | Initial spec |
| 1.1 | 2026-04-17 | Chat sender prefix |
| 1.2 | 2026-04-17 | GATEWAY field added |
| 1.3 | 2026-04-20 | Bidirectional group-chat relay prefix |
| 1.4 | 2026-04-20 | Human-readable hint tail |
| **2.0** | **2026-04-24** | **Ground-up rewrite for airtime efficiency. Packed prefix, letter/numeric codes, iconset dictionary. Breaking change — no back-compat with v1.x.** |
