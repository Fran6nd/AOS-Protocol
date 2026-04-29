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
| [10. Short Player Data](#10-short-player-data) | S→C | in-game | `id` u8, `pid` u8, `team` i8, `weapon` u8 | 0.75 | Compact roster update (after team/weapon change). | Same `team` / `weapon` enums as packet 9. |
| [11. Move Object](#11-move-object) | S→C | in-game | `id` u8, `object_id` u8, `team` u8, `x`/`y`/`z` f32 LE | 0.75 | A game object (tent or intel) is teleported / repositioned. | `team`: 0=blue, 1=green, 2=neutral. `object_id` enumerates intels and tents (see detail). |
| [12. Create Player](#12-create-player) | S→C | handshake / in-game | `id` u8, `pid` u8, `weapon` u8, `team` i8, `x`/`y`/`z` f32 LE, `name` cp437 (NUL-terminated) | 0.75 | Player spawns: first spawn after join, or respawn after death / team change. | Spawn position is in world units. |
| [13. Block Action](#13-block-action) | S↔C | in-game | `id` u8, `pid` u8, `action` u8, `x`/`y`/`z` i32 LE | 0.75 | Block placement or destruction. Client sends on action, server validates and broadcasts. | `action`: 0=build, 1=bullet/spade destroy (1 block), 2=spade destroy (3-block column), 3=grenade destroy (3×3×3). |
| [14. Block Line](#14-block-line) | S↔C | in-game | `id` u8, `pid` u8, `start_xyz` i32×3 LE, `end_xyz` i32×3 LE | 0.75 | Client requests a line of blocks between two points (block-tool line draw). | Server fills the voxel line between the two integer block coordinates. |
| [15. State Data](#15-state-data) | S→C | handshake | `id` u8, `pid` u8, `fog_bgr` u8×3, `team1_bgr` u8×3, `team2_bgr` u8×3, `team1_name` cp437(10), `team2_name` cp437(10), `gamemode` u8, *gamemode state* | 0.75 | Final packet of the join handshake (after the last `Map Chunk`); also delivers gamemode parameters. | `gamemode`: 0=CTF, 1=TC. Trailing payload is `CTF State` (52 B) or `TC State` (variable). |
| [16. Kill Action](#16-kill-action) | S→C | in-game | `id` u8, `victim_pid` u8, `killer_pid` u8, `kill_type` u8, `respawn_time` u8 | 0.75 | A player dies (any cause) or is forcibly removed via team/class change. | `kill_type`: 0=weapon 1=headshot 2=melee 3=grenade 4=fall 5=team change 6=class change. `respawn_time` in seconds. |
| [17. Chat Message](#17-chat-message) | S↔C | in-game | `id` u8, `pid` u8, `chat_type` u8, `message` cp437 (NUL-terminated) | 0.75 | Player sends a chat line; server broadcasts to recipients (filtered by chat type). | `chat_type`: 0=all (white) 1=team (team colour) 2=system (red, server-only). |
| [18. Map Start](#18-map-start) | S→C | map-transfer | `id` u8, `map_size` u32 LE | 0.75 | Server begins sending the map after ENet connect or on map advance. | `map_size` is the byte length of the upcoming compressed map stream (sum of all `Map Chunk` payloads). |
| [19. Map Chunk](#19-map-chunk) | S→C | map-transfer | `id` u8, `data` bytes (DEFLATE/zlib map payload) | 0.75 | Repeated after `Map Start` until the entire compressed map has been transferred; the next packet is `State Data`. | Concatenate all chunk payloads in order, then zlib-decompress to obtain the AOS-format map. |
| [20. Player Left](#20-player-left) | S→C | disconnect | `id` u8, `pid` u8 | 0.75 | A player disconnects (quit, kick, timeout). | Slot becomes free for reuse. |
| [21. Territory Capture](#21-territory-capture) | S→C | in-game | `id` u8, `tent_id` u8, `winning` u8, `team` u8 | 0.75 | A team finishes capturing a Command Post in TC mode. | `winning`: 1 if this capture wins the round, else 0. `team`: 0=blue, 1=green. |
| [22. Progress Bar](#22-progress-bar) | S→C | in-game | `id` u8, `tent_id` u8, `capturing_team` u8, `rate` i8, `progress` f32 LE | 0.75 | TC capture progress update for a contested Command Post. | `progress` 0.0–1.0. `rate` in 5%-per-second units; sign indicates direction (positive = capturing team gaining). |
| [23. Intel Capture](#23-intel-capture) | S→C | in-game | `id` u8, `pid` u8, `winning` u8 | 0.75 | A player scores by bringing the enemy intel home (CTF). | `winning`: 1 if this capture wins the round, else 0. |
| [24. Intel Pickup](#24-intel-pickup) | S→C | in-game | `id` u8, `pid` u8 | 0.75 | A player picks up the enemy intel (CTF). | Whose intel is implied by team membership. |
| [25. Intel Drop](#25-intel-drop) | S→C | in-game | `id` u8, `pid` u8, `x`/`y`/`z` f32 LE | 0.75 | The intel is dropped (carrier died, switched team, or disconnected). | Position is where the intel landed in the world. |
| [26. Restock](#26-restock) | S→C | in-game | `id` u8, `pid` u8 | 0.75 | A player restocks at their tent (full ammo, blocks, grenades, HP). | Sent only to the restocked player. |
| [27. Fog Colour](#27-fog-colour) | S→C | in-game | `id` u8, `alpha` u8, `b` u8, `g` u8, `r` u8 | 0.75 | Server sets the player's fog (sky/horizon) colour. | `alpha` byte is unused. Colour bytes are A,B,G,R order on the wire. |
| [28. Weapon Reload](#28-weapon-reload) | S↔C | in-game | `id` u8, `pid` u8, `clip_ammo` u8, `reserve_ammo` u8 | 0.75 | A player reloads their weapon. | Client sends to start reload; server broadcasts updated counts. |
| [29. Change Team](#29-change-team) | C→S | in-game | `id` u8, `pid` u8, `team` i8 | 0.75 | Client requests to switch teams. | Server responds with `Kill Action` (`kill_type=5`) then `Create Player`. `team`: 255=spectator, 0=blue, 1=green. |
| [30. Change Weapon](#30-change-weapon) | C→S | in-game | `id` u8, `pid` u8, `weapon` u8 | 0.75 | Client requests to switch weapons. | Server responds with `Kill Action` (`kill_type=6`) then `Create Player`. `weapon`: 0=rifle, 1=SMG, 2=shotgun. |

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

### 10. Short Player Data

- **ID:** `0x0A` · **Direction:** S→C · **Phase:** in-game · **Introduced:** 0.75 · **Size:** 4 bytes
- **Trigger:** Compact roster update — sent when only team/weapon attributes need to be refreshed.

| Offset | Field | Type | Description |
|---|---|---|---|
| 0 | `packet_id` | u8 | `0x0A` |
| 1 | `player_id` | u8 | which slot |
| 2 | `team` | i8 | 255 = spectator, 0 = blue, 1 = green |
| 3 | `weapon` | u8 | 0 = rifle, 1 = SMG, 2 = shotgun |

### 11. Move Object

- **ID:** `0x0B` · **Direction:** S→C · **Phase:** in-game · **Introduced:** 0.75 · **Size:** 15 bytes
- **Trigger:** A game object (intel or tent / command post) is moved or reset (e.g. intel returned).

| Offset | Field | Type | Description |
|---|---|---|---|
| 0 | `packet_id` | u8 | `0x0B` |
| 1 | `object_id` | u8 | object index (see below) |
| 2 | `team` | u8 | 0 = blue, 1 = green, 2 = neutral |
| 3 | `x` | f32 LE | world X |
| 7 | `y` | f32 LE | world Y |
| 11 | `z` | f32 LE | world Z |

Object index meaning (CTF mode):

| `object_id` | Object |
|---|---|
| 0 | blue team intel |
| 1 | green team intel |
| 2 | blue team tent (base) |
| 3 | green team tent (base) |

In Territory Control, `object_id` indexes the territory list from `State Data`.

### 12. Create Player

- **ID:** `0x0C` · **Direction:** S→C · **Phase:** handshake (first spawn) / in-game (respawn) · **Introduced:** 0.75 · **Size:** variable
- **Trigger:** Player enters the world: first spawn after the joining handshake completes, or respawn after death, team change, or weapon change.

| Offset | Field | Type | Description |
|---|---|---|---|
| 0 | `packet_id` | u8 | `0x0C` |
| 1 | `player_id` | u8 | which slot |
| 2 | `weapon` | u8 | 0 = rifle, 1 = SMG, 2 = shotgun |
| 3 | `team` | i8 | 255 = spectator, 0 = blue, 1 = green |
| 4 | `x` | f32 LE | spawn world X |
| 8 | `y` | f32 LE | spawn world Y |
| 12 | `z` | f32 LE | spawn world Z |
| 16 | `name` | cp437, NUL-terminated | player nickname |

### 13. Block Action

- **ID:** `0x0D` · **Direction:** S↔C · **Phase:** in-game · **Introduced:** 0.75 · **Size:** 15 bytes
- **Trigger:** Block placement or destruction. Client sends on player action; server validates and broadcasts to other clients.

| Offset | Field | Type | Description |
|---|---|---|---|
| 0 | `packet_id` | u8 | `0x0D` |
| 1 | `player_id` | u8 | who acted |
| 2 | `action` | u8 | see enum below |
| 3 | `x` | i32 LE | block X (voxel coordinate) |
| 7 | `y` | i32 LE | block Y |
| 11 | `z` | i32 LE | block Z |

`action`:

| Value | Meaning |
|---|---|
| 0 | build (place a block at X,Y,Z) |
| 1 | bullet / spade destroy (single block) |
| 2 | spade destroy — column of 3 blocks (the block plus the two below) |
| 3 | grenade destroy — 3×3×3 cube centred on X,Y,Z |

### 14. Block Line

- **ID:** `0x0E` · **Direction:** S↔C · **Phase:** in-game · **Introduced:** 0.75 · **Size:** 26 bytes
- **Trigger:** Block-tool line draw — client requests a line of blocks between two voxel coordinates (the start and current position when the block tool is dragged). Server validates and broadcasts.

| Offset | Field | Type | Description |
|---|---|---|---|
| 0 | `packet_id` | u8 | `0x0E` |
| 1 | `player_id` | u8 | who is placing |
| 2 | `start_x` | i32 LE | first endpoint X |
| 6 | `start_y` | i32 LE | first endpoint Y |
| 10 | `start_z` | i32 LE | first endpoint Z |
| 14 | `end_x` | i32 LE | second endpoint X |
| 18 | `end_y` | i32 LE | second endpoint Y |
| 22 | `end_z` | i32 LE | second endpoint Z |

### 15. State Data

- **ID:** `0x0F` · **Direction:** S→C · **Phase:** handshake · **Introduced:** 0.75 · **Size:** variable (≥ 52 bytes; 104 B with CTF state, more with TC state)
- **Trigger:** Sent once after the last `Map Chunk` to signal that the join is complete and to deliver gamemode parameters.

| Offset | Field | Type | Description |
|---|---|---|---|
| 0 | `packet_id` | u8 | `0x0F` |
| 1 | `player_id` | u8 | the receiving client's assigned slot |
| 2 | `fog_bgr` | 3 × u8 | fog colour (BGR order) |
| 5 | `team1_bgr` | 3 × u8 | blue team colour (BGR) |
| 8 | `team2_bgr` | 3 × u8 | green team colour (BGR) |
| 11 | `team1_name` | cp437, 10 bytes (NUL-padded) | blue team name |
| 21 | `team2_name` | cp437, 10 bytes (NUL-padded) | green team name |
| 31 | `gamemode` | u8 | 0 = CTF, 1 = TC |
| 32 | *gamemode state* | variable | `CTF State` (52 bytes) or `TC State` (variable) |

**CTF State** (when `gamemode == 0`, 52 bytes):

| Offset (rel.) | Field | Type | Description |
|---|---|---|---|
| 0 | `team1_score` | u8 | blue caps |
| 1 | `team2_score` | u8 | green caps |
| 2 | `cap_limit` | u8 | caps to win |
| 3 | `intel_flags` | u8 | bit 0 = team1 has team2's intel, bit 1 = team2 has team1's intel |
| 4 | `team1_intel` | 12 bytes | if blue holds green's intel, that section is bit 1 of `intel_flags`; here: when `intel_flags & 1` (green holds blue intel) → carrier `player_id` u8 + 11 padding bytes; otherwise → `x`/`y`/`z` f32 LE (dropped position) |
| 16 | `team2_intel` | 12 bytes | mirror of above for green's intel |
| 28 | `team1_base` | 3 × f32 LE | blue tent (base) position |
| 40 | `team2_base` | 3 × f32 LE | green tent (base) position |

**TC State** (when `gamemode == 1`, variable):

| Offset (rel.) | Field | Type | Description |
|---|---|---|---|
| 0 | `territory_count` | u8 | 0–16 |
| 1 | `territories[]` | 13 × `territory_count` | each entry: `x`/`y`/`z` f32 LE + `team` u8 (0 = blue, 1 = green, 2 = neutral) |

The TC payload is padded to a fixed 16-territory area in piqueserver's writer (`MAX_TERRITORIES * 13 = 208` bytes after the count), but readers should consume only `territory_count` entries.

### 16. Kill Action

- **ID:** `0x10` · **Direction:** S→C · **Phase:** in-game · **Introduced:** 0.75 · **Size:** 5 bytes
- **Trigger:** A player dies. Also used as the server's response when a client requests a team change (`Change Team`) or weapon change (`Change Weapon`) — the server kills the player, then respawns them via `Create Player`.

| Offset | Field | Type | Description |
|---|---|---|---|
| 0 | `packet_id` | u8 | `0x10` |
| 1 | `victim_pid` | u8 | who died |
| 2 | `killer_pid` | u8 | attributed killer (= `victim_pid` for self-inflicted causes) |
| 3 | `kill_type` | u8 | see enum below |
| 4 | `respawn_time` | u8 | seconds until the victim respawns |

`kill_type`:

| Value | Meaning |
|---|---|
| 0 | weapon (body shot) |
| 1 | headshot |
| 2 | melee (spade) |
| 3 | grenade |
| 4 | fall damage |
| 5 | team change |
| 6 | class / weapon change |

### 17. Chat Message

- **ID:** `0x11` · **Direction:** S↔C · **Phase:** in-game · **Introduced:** 0.75 · **Size:** variable (≥ 4 bytes)
- **Trigger:** Player sends a chat line; server broadcasts to recipients. Server also originates `chat_type = 2` system messages.

| Offset | Field | Type | Description |
|---|---|---|---|
| 0 | `packet_id` | u8 | `0x11` |
| 1 | `player_id` | u8 | sender (255 / 0xFF when the server is the sender for system messages) |
| 2 | `chat_type` | u8 | 0 = all chat, 1 = team chat, 2 = system message |
| 3 | `message` | cp437, NUL-terminated | message body |

### 18. Map Start

- **ID:** `0x12` · **Direction:** S→C · **Phase:** map-transfer · **Introduced:** 0.75 · **Size:** 5 bytes
- **Trigger:** Sent once at the start of the map transfer (right after ENet connect, or when the server advances to a new map).

| Offset | Field | Type | Description |
|---|---|---|---|
| 0 | `packet_id` | u8 | `0x12` |
| 1 | `map_size` | u32 LE | total length in bytes of the upcoming compressed map stream |

### 19. Map Chunk

- **ID:** `0x13` · **Direction:** S→C · **Phase:** map-transfer · **Introduced:** 0.75 · **Size:** variable (1 byte header + payload)
- **Trigger:** Repeated zero or more times after `Map Start` until the full compressed map has been delivered. The next packet after the final chunk is `State Data`.

| Offset | Field | Type | Description |
|---|---|---|---|
| 0 | `packet_id` | u8 | `0x13` |
| 1 | `data` | bytes | DEFLATE / zlib payload chunk |

The client concatenates all `data` payloads in order, then zlib-decompresses to recover the AOS-format map (column-major voxel + colour data).

### 20. Player Left

- **ID:** `0x14` · **Direction:** S→C · **Phase:** disconnect · **Introduced:** 0.75 · **Size:** 2 bytes
- **Trigger:** A player disconnects (graceful quit, server kick, or ENet timeout). Slot is now free.

| Offset | Field | Type | Description |
|---|---|---|---|
| 0 | `packet_id` | u8 | `0x14` |
| 1 | `player_id` | u8 | which slot left |

### 21. Territory Capture

- **ID:** `0x15` · **Direction:** S→C · **Phase:** in-game · **Introduced:** 0.75 · **Size:** 4 bytes
- **Trigger:** A team completes capturing a Command Post in Territory Control mode.

| Offset | Field | Type | Description |
|---|---|---|---|
| 0 | `packet_id` | u8 | `0x15` |
| 1 | `tent_id` | u8 | index into the territory list from `State Data` |
| 2 | `winning` | u8 | 1 = this capture wins the round, 0 = does not |
| 3 | `team` | u8 | 0 = blue captured, 1 = green captured |

Note: the original web spec lists a 5-byte form including a player ID; both `piqueserver` and `BetterSpades` implement the 4-byte form (no player ID). The 4-byte form is what is actually on the wire.

### 22. Progress Bar

- **ID:** `0x16` · **Direction:** S→C · **Phase:** in-game · **Introduced:** 0.75 · **Size:** 8 bytes
- **Trigger:** TC capture progress update for a contested Command Post (sent while progress changes).

| Offset | Field | Type | Description |
|---|---|---|---|
| 0 | `packet_id` | u8 | `0x16` |
| 1 | `tent_id` | u8 | index into the territory list |
| 2 | `capturing_team` | u8 | 0 = blue, 1 = green |
| 3 | `rate` | i8 (signed) | capture rate in 5%-per-second units; positive when the capturing team is making progress, negative when contested |
| 4 | `progress` | f32 LE | normalised progress (0.0 – 1.0); 1.0 means a full capture is imminent |

### 23. Intel Capture

- **ID:** `0x17` · **Direction:** S→C · **Phase:** in-game · **Introduced:** 0.75 · **Size:** 3 bytes
- **Trigger:** A player completes a CTF capture by returning the enemy intel to their tent.

| Offset | Field | Type | Description |
|---|---|---|---|
| 0 | `packet_id` | u8 | `0x17` |
| 1 | `player_id` | u8 | the capturing player |
| 2 | `winning` | u8 | 1 = this capture wins the round, 0 = does not |

### 24. Intel Pickup

- **ID:** `0x18` · **Direction:** S→C · **Phase:** in-game · **Introduced:** 0.75 · **Size:** 2 bytes
- **Trigger:** A player picks up the enemy intel.

| Offset | Field | Type | Description |
|---|---|---|---|
| 0 | `packet_id` | u8 | `0x18` |
| 1 | `player_id` | u8 | the picker (which intel is implied by their team — they pick up the *opposing* team's flag) |

### 25. Intel Drop

- **ID:** `0x19` · **Direction:** S→C · **Phase:** in-game · **Introduced:** 0.75 · **Size:** 14 bytes
- **Trigger:** The intel is dropped — the carrier died, switched teams, or disconnected.

| Offset | Field | Type | Description |
|---|---|---|---|
| 0 | `packet_id` | u8 | `0x19` |
| 1 | `player_id` | u8 | the former carrier |
| 2 | `x` | f32 LE | drop world X |
| 6 | `y` | f32 LE | drop world Y |
| 10 | `z` | f32 LE | drop world Z |

### 26. Restock

- **ID:** `0x1A` · **Direction:** S→C · **Phase:** in-game · **Introduced:** 0.75 · **Size:** 2 bytes
- **Trigger:** A player restocks at their tent — clip + reserve ammo, blocks, grenades, and HP all refill. Only the restocked player receives this packet.

| Offset | Field | Type | Description |
|---|---|---|---|
| 0 | `packet_id` | u8 | `0x1A` |
| 1 | `player_id` | u8 | the restocked player |

### 27. Fog Colour

- **ID:** `0x1B` · **Direction:** S→C · **Phase:** in-game · **Introduced:** 0.75 · **Size:** 5 bytes
- **Trigger:** Server changes the fog (sky / horizon) colour for the receiving client.

| Offset | Field | Type | Description |
|---|---|---|---|
| 0 | `packet_id` | u8 | `0x1B` |
| 1 | `alpha` | u8 | unused, typically 0 |
| 2 | `blue` | u8 | B channel |
| 3 | `green` | u8 | G channel |
| 4 | `red` | u8 | R channel |

### 28. Weapon Reload

- **ID:** `0x1C` · **Direction:** S↔C · **Phase:** in-game · **Introduced:** 0.75 · **Size:** 4 bytes
- **Trigger:** Player reloads. Client sends a request when the player presses reload; server broadcasts the resulting clip / reserve counts to other clients (and updates the local player's UI).

| Offset | Field | Type | Description |
|---|---|---|---|
| 0 | `packet_id` | u8 | `0x1C` |
| 1 | `player_id` | u8 | who is reloading |
| 2 | `clip_ammo` | u8 | rounds in clip after reload |
| 3 | `reserve_ammo` | u8 | rounds remaining in reserve |

### 29. Change Team

- **ID:** `0x1D` · **Direction:** C→S · **Phase:** in-game · **Introduced:** 0.75 · **Size:** 3 bytes
- **Trigger:** Client requests a team change. The server response is **not** an echo of this packet; instead, it sends `Kill Action` with `kill_type = 5` (team change), then `Create Player` to respawn the player on the new team.

| Offset | Field | Type | Description |
|---|---|---|---|
| 0 | `packet_id` | u8 | `0x1D` |
| 1 | `player_id` | u8 | the requesting player |
| 2 | `team` | i8 | 255 = spectator, 0 = blue, 1 = green |

### 30. Change Weapon

- **ID:** `0x1E` · **Direction:** C→S · **Phase:** in-game · **Introduced:** 0.75 · **Size:** 3 bytes
- **Trigger:** Client requests a weapon change. As with `Change Team`, the server responds via `Kill Action` (`kill_type = 6`, class change) followed by `Create Player`.

| Offset | Field | Type | Description |
|---|---|---|---|
| 0 | `packet_id` | u8 | `0x1E` |
| 1 | `player_id` | u8 | the requesting player |
| 2 | `weapon` | u8 | 0 = rifle, 1 = SMG, 2 = shotgun |

## Sources

- https://www.piqueserver.org/aosprotocol/protocol075.html
- ../piqueserver/pyspades/contained.pyx
- ../piqueserver/pyspades/loaders.pyx
- ../piqueserver/pyspades/constants.py
- ../zerospades/Sources/Client/NetClient.cpp
- ../zerospades/Sources/Client/NetProtocol.h
- ../BetterSpades/src/network.c
- ../BetterSpades/src/network.h
