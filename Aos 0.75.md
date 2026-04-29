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

## Sources

- https://www.piqueserver.org/aosprotocol/protocol075.html
- ../piqueserver/pyspades/contained.pyx
- ../piqueserver/pyspades/loaders.pyx
- ../piqueserver/pyspades/constants.py
- ../zerospades/Sources/Client/NetClient.cpp
- ../zerospades/Sources/Client/NetProtocol.h
- ../BetterSpades/src/network.c
- ../BetterSpades/src/network.h
