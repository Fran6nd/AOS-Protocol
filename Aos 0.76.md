# Ace of Spades 0.76 Protocol

Wire-format reference for AoS network protocol version 0.76.

0.76 is a small superset of [0.75](Aos%200.75.md). This document lists only what differs from 0.75; for every other packet, the [Master Server & Serverlist](Aos%200.75.md#master-server--serverlist) flow, the [Protocol Extensions](Aos%200.75.md#protocol-extensions) layer, and the [Conventions](Aos%200.75.md#conventions), refer to the 0.75 document.

## Chapters

1. [Summary of Changes](#summary-of-changes) — what 0.76 adds or alters.
2. [Phases](#phases) — delta on the 0.75 phase table.
3. [Changed Packets](#changed-packets) — `Map Start`, `World Update`.
4. [New Packets](#new-packets) — `Map Cached`.
5. [Sources](#sources) — references used to compile this document.

## Summary of Changes

- **Transport:** ENet handshake version `4` on the game port (was `3` in 0.75). The serverlist `game_version` field reports `"0.76"` (vs `"0.75"`).
- **Map Start** is extended with a CRC32 of the uncompressed map and a map name, allowing client-side caching.
- **Map Cached** is a new client-to-server packet — the client tells the server whether the map (identified by `crc32` from `Map Start`) is already cached locally, so the server can skip retransmitting `Map Chunk` frames.
- **World Update** is reformatted: it now carries only active player slots and prefixes each entry with its `player_id`, instead of a fixed 32-slot array.

All other packets are byte-for-byte identical to 0.75. The `Introduced` column in the [0.75 Packet Summary](Aos%200.75.md#packet-summary) reflects each packet's first-appearance version.

## Phases

The phase table is identical to [0.75 › Phases](Aos%200.75.md#phases) except that the `map-transfer` phase can be short-circuited:

| Phase | 0.76 difference |
|---|---|
| `map-transfer` | The server skips the `Map Chunk` stream when the client confirms a cache hit via `Map Cached`. The phase still runs, but only `Map Start` → `Map Cached` → `State Data` flow on the wire. |

The optional `capability-handshake` phase (extension negotiation) is unchanged from 0.75.

## Changed Packets

### Map Start

- **ID:** `0x12` · **Direction:** S→C · **Phase:** map-transfer · **Introduced:** 0.75 → 0.76 · **Size:** variable (≥ 10 bytes)
- **Trigger:** Sent once at the start of the map transfer (right after ENet connect, or when the server advances to a new map). The client is expected to reply with `Map Cached` once it has read this header.

| Offset | Field | Type | Description |
|---|---|---|---|
| 0 | `packet_id` | u8 | `0x12` |
| 1 | `map_size` | u32 LE | total length in bytes of the upcoming compressed map stream |
| 5 | `crc32` | u32 LE | CRC-32 of the *uncompressed* map data, used as the cache key |
| 9 | `map_name` | cp437, NUL-terminated | server-supplied map name (purely informational) |

In 0.75 only `map_size` was present (5-byte fixed packet). The added `crc32` and `map_name` fields are what enable client-side caching in 0.76.

### World Update

- **ID:** `0x02` · **Direction:** S→C · **Phase:** in-game · **Introduced:** 0.75 → 0.76 · **Size:** 1 + N × 25 bytes
- **Trigger:** Periodic broadcast (server tick) of position + orientation for every active player slot.

| Offset | Field | Type | Description |
|---|---|---|---|
| 0 | `packet_id` | u8 | `0x02` |
| 1 + i·25 | `player_id` (entry i) | u8 | which slot this entry is for |
| 2 + i·25 | `pos` (entry i) | 3 × f32 LE | player's world position (X, Y, Z) |
| 14 + i·25 | `ori` (entry i) | 3 × f32 LE | player's orientation vector (X, Y, Z) |

The list contains exactly one entry per occupied slot — empty slots are omitted. The end of the list is determined by packet length: the receiver keeps reading 25-byte entries until the buffer is exhausted.

This is the main wire-level change from [0.75 › World Update](Aos%200.75.md#world-update), which uses a fixed 32-slot array without per-entry player IDs.

## New Packets

### Map Cached

- **ID:** `0x1F` · **Direction:** C→S · **Phase:** map-transfer · **Introduced:** 0.76 · **Size:** 2 bytes
- **Trigger:** Sent by the client immediately after it receives `Map Start`, to declare whether it already has the map (matched by `crc32`) cached locally. If `cached = 1`, the server may skip the `Map Chunk` stream entirely and proceed straight to `State Data`.

| Offset | Field | Type | Description |
|---|---|---|---|
| 0 | `packet_id` | u8 | `0x1F` |
| 1 | `cached` | u8 | 0 = not cached (server proceeds with `Map Chunk`), 1 = cached (server may skip to `State Data`) |

This ID is also used by `Handshake Init` (see [0.75 › Capability Handshake Packets](Aos%200.75.md#capability-handshake-packets)). Disambiguation is by phase — `Handshake Init` precedes `Map Start`; `Map Cached` only ever follows it.

## Sources

- https://www.piqueserver.org/aosprotocol/protocol075.html (covers 0.75 and notes 0.76 deltas inline)
- [piqueserver](https://www.piqueserver.org): `pyspades/contained.pyx`, `pyspades/loaders.pyx`, `pyspades/constants.py`
- [zerospades](https://github.com/siecvi/zerospades): `Sources/Client/NetClient.cpp`, `Sources/Client/NetClient.h`
- [BetterSpades](https://github.com/xtreme8000/BetterSpades): `src/network.c`, `src/network.h`
