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

## Packet Details

## Sources

- https://www.piqueserver.org/aosprotocol/protocol075.html
- ../piqueserver/pyspades/contained.pyx
- ../piqueserver/pyspades/loaders.pyx
- ../piqueserver/pyspades/constants.py
- ../zerospades/Sources/Client/NetClient.cpp
- ../zerospades/Sources/Client/NetProtocol.h
- ../BetterSpades/src/network.c
- ../BetterSpades/src/network.h
