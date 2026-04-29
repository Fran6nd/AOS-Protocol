# Ace of Spades 0.75 Protocol

Wire-format reference for AoS network protocol version 0.75. Transport is ENet over UDP. All multi-byte integers and floats are little-endian unless otherwise noted. Strings are CP437-encoded.

## Phases

| Phase | Meaning |
|---|---|
| `handshake` | After ENet connect, before the player is in-game. Covers exchanging player identity and initial game state. |
| `map-transfer` | Server pushes the compressed map to the client. Bracketed by `Map Start` and the final `State Data`. |
| `in-game` | Player is alive or spectating in the world; gameplay packets flow. |
| `disconnect` | Player is leaving (kick, quit, timeout). |

## Summary

| Packet Name | Direction | Phase | Structure | Introduced | Trigger | Field Values |
|---|---|---|---|---|---|---|
| [0. Position Data](#0-position-data) | C↔S | in-game | `id` u8, `x`/`y`/`z` f32 LE | 0.75 | Player position changes; client sends, server may set position by sending back. | Coords in world units (0–512 horizontal, 0–63 vertical). |
| [1. Orientation Data](#1-orientation-data) | C↔S | in-game | `id` u8, `x`/`y`/`z` f32 LE | 0.75 | Player view direction changes; server may force orientation. | Unit vector for view direction. |
| [2. World Update](#2-world-update) | S→C | in-game | `id` u8, then 32× (`pos` f32×3, `ori` f32×3) LE | 0.75 | Periodic broadcast of every slot's position + orientation. | Fixed 32-slot array regardless of fill; unused slots may carry stale/zero data. |
| [3. Input Data](#3-input-data) | C→S | in-game | `id` u8, `pid` u8, `keys` u8 (bitfield) | 0.75 | Movement key state changes (press or release). | `keys` bits: 0=up 1=down 2=left 3=right 4=jump 5=crouch 6=sneak 7=sprint. |
| [4. Weapon Input](#4-weapon-input) | C→S | in-game | `id` u8, `pid` u8, `fire` u8 (bitfield) | 0.75 | Primary/secondary fire button state changes. | `fire` bits: 0=primary 1=secondary. |
| [5. Hit Packet](#5-hit-packet) | C→S | in-game | `id` u8, `target_pid` u8, `hit_type` u8 | 0.75 | Client reports it hit another player (client-side hit detection). | `hit_type`: 0=torso 1=head 2=arms 3=legs 4=melee. |
| [5. Set HP](#5-set-hp) | S→C | in-game | `id` u8, `hp` u8, `damage_type` u8, `src_x`/`src_y`/`src_z` f32 LE | 0.75 | Server informs the receiving player their HP changed (after damage). | `damage_type`: 0=fall 1=weapon. `src_*` is damage source position (used to draw hit indicator). |
| [6. Grenade Packet](#6-grenade-packet) | S↔C | in-game | `id` u8, `pid` u8, `fuse` f32 LE, `pos` f32×3 LE, `vel` f32×3 LE | 0.75 | Grenade is thrown. Client sends on throw; server broadcasts to other clients. | `fuse` in seconds; `pos`, `vel` in world units. |
| [7. Set Tool](#7-set-tool) | S↔C | in-game | `id` u8, `pid` u8, `tool` u8 | 0.75 | Player switches active tool (1–4 keys). | `tool`: 0=spade 1=block 2=gun 3=grenade. |
| [8. Set Colour](#8-set-colour) | S↔C | in-game | `id` u8, `pid` u8, `b` u8, `g` u8, `r` u8 | 0.75 | Player picks a new block colour. | BGR byte order on the wire. |
| [9. Existing Player](#9-existing-player) | S↔C | handshake / in-game | `id` u8, `pid` u8, `team` i8, `weapon` u8, `tool` u8, `kills` u32 LE, `b`/`g`/`r` u8, `name` cp437 (≤16, NUL-terminated) | 0.75 | C→S: new player announces itself on join. S→C: server tells a client about each already-connected player. | `team`: 255=spectator, 0=blue, 1=green. `weapon`: 0=rifle 1=smg 2=shotgun. `tool`: 0=spade 1=block 2=gun 3=grenade. |

## Packet Details

### 0. Position Data

- **ID:** `0x00` · **Direction:** C↔S · **Phase:** in-game · **Introduced:** 0.75 · **Size:** 13 bytes
- **Trigger:** Sent by the client when its position changes. Server may emit the same packet to forcibly relocate the receiving player.

| Offset | Field | Type | Description |
|---|---|---|---|
| 0 | `packet_id` | u8 | `0x00` |
| 1 | `x` | f32 LE | world X (0.0 – 512.0) |
| 5 | `y` | f32 LE | world Y (0.0 – 512.0) |
| 9 | `z` | f32 LE | world Z, vertical axis (downward positive in 0.75 maps; ~0 at the sky, ~63 at solid ground) |

### 1. Orientation Data

- **ID:** `0x01` · **Direction:** C↔S · **Phase:** in-game · **Introduced:** 0.75 · **Size:** 13 bytes
- **Trigger:** Sent by the client when its view direction changes. Server may emit it to force a view direction on the client.

| Offset | Field | Type | Description |
|---|---|---|---|
| 0 | `packet_id` | u8 | `0x01` |
| 1 | `x` | f32 LE | view-vector X component |
| 5 | `y` | f32 LE | view-vector Y component |
| 9 | `z` | f32 LE | view-vector Z component (unit-length expected) |

### 2. World Update

- **ID:** `0x02` · **Direction:** S→C · **Phase:** in-game · **Introduced:** 0.75 · **Size:** 1 + 32 × 24 = 769 bytes
- **Trigger:** Periodic broadcast (server tick) of position + orientation for every player slot.

| Offset | Field | Type | Description |
|---|---|---|---|
| 0 | `packet_id` | u8 | `0x02` |
| 1 + i·24 | `pos` (slot i) | 3 × f32 LE | player slot `i`'s position (X, Y, Z) for `i` in 0..31 |
| 13 + i·24 | `ori` (slot i) | 3 × f32 LE | player slot `i`'s orientation vector (X, Y, Z) for `i` in 0..31 |

The array is always exactly 32 entries regardless of how many slots are occupied. Empty slots have undefined values that clients are expected to ignore based on roster knowledge from `Existing Player` / `Player Left`.

### 3. Input Data

- **ID:** `0x03` · **Direction:** C→S · **Phase:** in-game · **Introduced:** 0.75 · **Size:** 3 bytes
- **Trigger:** Movement key state change (press or release). Server relays the resulting state to other clients via the same packet ID.

| Offset | Field | Type | Description |
|---|---|---|---|
| 0 | `packet_id` | u8 | `0x03` |
| 1 | `player_id` | u8 | player whose input this describes |
| 2 | `keys` | u8 | bitfield, see below |

`keys` bit layout (LSB = bit 0):

| Bit | Meaning |
|---|---|
| 0 | up (W) |
| 1 | down (S) |
| 2 | left (A) |
| 3 | right (D) |
| 4 | jump |
| 5 | crouch |
| 6 | sneak |
| 7 | sprint |

### 4. Weapon Input

- **ID:** `0x04` · **Direction:** C→S · **Phase:** in-game · **Introduced:** 0.75 · **Size:** 3 bytes
- **Trigger:** Primary or secondary fire button state changes (press / release).

| Offset | Field | Type | Description |
|---|---|---|---|
| 0 | `packet_id` | u8 | `0x04` |
| 1 | `player_id` | u8 | player whose fire input this describes |
| 2 | `fire` | u8 | bitfield: bit 0 = primary fire held, bit 1 = secondary fire held |

### 5. Hit Packet

- **ID:** `0x05` · **Direction:** C→S · **Phase:** in-game · **Introduced:** 0.75 · **Size:** 3 bytes
- **Trigger:** Client-side hit detection — sent when the local client believes its bullet/melee struck another player. The server is authoritative about whether the hit lands.

| Offset | Field | Type | Description |
|---|---|---|---|
| 0 | `packet_id` | u8 | `0x05` |
| 1 | `target_player_id` | u8 | who was hit |
| 2 | `hit_type` | u8 | see enum below |

`hit_type`:

| Value | Meaning |
|---|---|
| 0 | torso |
| 1 | head |
| 2 | arms |
| 3 | legs |
| 4 | melee (spade) |

### 5. Set HP

- **ID:** `0x05` · **Direction:** S→C · **Phase:** in-game · **Introduced:** 0.75 · **Size:** 15 bytes
- **Trigger:** Server informs the receiving player their HP changed (took damage from any source). Sent only to the damaged player.

Note: this shares packet ID `0x05` with `Hit Packet`. Direction disambiguates them.

| Offset | Field | Type | Description |
|---|---|---|---|
| 0 | `packet_id` | u8 | `0x05` |
| 1 | `hp` | u8 | current HP after damage (0–100) |
| 2 | `damage_type` | u8 | 0 = fall, 1 = weapon |
| 3 | `source_x` | f32 LE | world X of damage source |
| 7 | `source_y` | f32 LE | world Y of damage source |
| 11 | `source_z` | f32 LE | world Z of damage source |

### 6. Grenade Packet

- **ID:** `0x06` · **Direction:** S↔C · **Phase:** in-game · **Introduced:** 0.75 · **Size:** 30 bytes
- **Trigger:** A grenade is thrown. Client sends on throw with its starting state; server broadcasts to other clients (the thrower's own client adds the grenade locally and does not expect a server echo).

| Offset | Field | Type | Description |
|---|---|---|---|
| 0 | `packet_id` | u8 | `0x06` |
| 1 | `player_id` | u8 | thrower |
| 2 | `fuse` | f32 LE | seconds until detonation |
| 6 | `pos_x` / `pos_y` / `pos_z` | 3 × f32 LE | initial world position |
| 18 | `vel_x` / `vel_y` / `vel_z` | 3 × f32 LE | initial velocity (world units / s) |

### 7. Set Tool

- **ID:** `0x07` · **Direction:** S↔C · **Phase:** in-game · **Introduced:** 0.75 · **Size:** 3 bytes
- **Trigger:** Player switches their active tool (1/2/3/4 keys).

| Offset | Field | Type | Description |
|---|---|---|---|
| 0 | `packet_id` | u8 | `0x07` |
| 1 | `player_id` | u8 | who switched |
| 2 | `tool` | u8 | 0 = spade, 1 = block, 2 = gun, 3 = grenade |

### 8. Set Colour

- **ID:** `0x08` · **Direction:** S↔C · **Phase:** in-game · **Introduced:** 0.75 · **Size:** 5 bytes
- **Trigger:** Player picks a new block colour from the colour palette.

| Offset | Field | Type | Description |
|---|---|---|---|
| 0 | `packet_id` | u8 | `0x08` |
| 1 | `player_id` | u8 | who changed colour |
| 2 | `blue` | u8 | B channel |
| 3 | `green` | u8 | G channel |
| 4 | `red` | u8 | R channel |

Note: the colour bytes appear on the wire in **BGR** order, not RGB.

### 9. Existing Player

- **ID:** `0x09` · **Direction:** S↔C · **Phase:** handshake (then occasionally in-game) · **Introduced:** 0.75 · **Size:** variable (≥28 bytes incl. NUL)
- **Trigger:**
  - C→S: the joining client sends one of these to announce its identity once map transfer + `State Data` are complete.
  - S→C: the server sends one for each already-connected player so the new client can populate its roster.

| Offset | Field | Type | Description |
|---|---|---|---|
| 0 | `packet_id` | u8 | `0x09` |
| 1 | `player_id` | u8 | which slot this describes |
| 2 | `team` | i8 | 255 (`-1`) = spectator, 0 = blue, 1 = green |
| 3 | `weapon` | u8 | 0 = rifle, 1 = SMG, 2 = shotgun |
| 4 | `tool` | u8 | 0 = spade, 1 = block, 2 = gun, 3 = grenade |
| 5 | `kills` | u32 LE | kill count |
| 9 | `blue` / `green` / `red` | 3 × u8 | block colour, BGR order |
| 12 | `name` | cp437, NUL-terminated, ≤16 bytes | player nickname |

## Sources

- https://www.piqueserver.org/aosprotocol/protocol075.html
- ../piqueserver/pyspades/contained.pyx
- ../piqueserver/pyspades/loaders.pyx
- ../piqueserver/pyspades/constants.py
- ../zerospades/Sources/Client/NetClient.cpp
- ../zerospades/Sources/Client/NetProtocol.h
- ../BetterSpades/src/network.c
- ../BetterSpades/src/network.h
