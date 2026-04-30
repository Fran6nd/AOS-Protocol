# AoS Protocol

An attempt to consolidate the **Ace of Spades** wire-format documentation into a single, GitHub-renderable reference per protocol version. The actual authority remains the source code and existing docs of [piqueserver](https://www.piqueserver.org), [zerospades](https://github.com/siecvi/zerospades), [BetterSpades](https://github.com/xtreme8000/BetterSpades), [OpenSpades](https://openspades.yvt.jp), [SpadesX](https://github.com/SpadesX), and the master-list archive — this repo just bundles what they describe.

Existing docs are scattered across forum threads, source files, and a partial wiki on the piqueserver site; the goal here is to make them easier to browse in one place. There may well be inaccuracies — corrections are welcome.

- [Aos 0.75.md](Aos%200.75.md) — vanilla protocol.
- [Aos 0.76.md](Aos%200.76.md) — small superset of 0.75 (delta doc; links into 0.75 for shared content).

## Could this also be a place to discuss future protocol versions?

Possibly. If anyone is interested in proposing changes for 0.77 onwards, this repo could host the discussion — one idea per issue, scoped to core wire-protocol concerns. Anything implementable as a server-side script (piqueserver scripts, gamemodes, admin tools) probably belongs elsewhere.

This is just an idea, not a settled process. If it sounds useful, open an issue.

## A note on security for any future proposal

Never trust data coming from the client. Clients can be modified, recompiled, or fully reimplemented; any field a client sends can be forged. Any proposal that adds or changes a client→server packet should specify how the server validates it (bounds, ownership, rate, state). "The client should not send X" is not a defence.

## Sources

- https://www.piqueserver.org/aosprotocol/protocol075.html
- [piqueserver](https://www.piqueserver.org) — server, Python/Cython
- [zerospades](https://github.com/siecvi/zerospades) — client, C++
- [BetterSpades](https://github.com/xtreme8000/BetterSpades) — client, C
- [OpenSpades](https://openspades.yvt.jp) — client, C++
- [SpadesX](https://github.com/SpadesX) — server, C
- master-list archive
