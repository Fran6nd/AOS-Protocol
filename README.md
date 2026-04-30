# AoS Protocol

Working repository for drafting Ace of Spades network protocol versions **0.77** and **0.78**.

Existing protocol versions are documented for reference:

- [Aos 0.75.md](Aos%200.75.md)
- [Aos 0.76.md](Aos%200.76.md)

## Goal

Improve AoS. This is not a place to design a new game. Proposals that change the spirit of AoS, or that could ship as a server-side script under the current protocol, do not belong here.

## Scope: core vs. scriptable

Every proposal must state which side it belongs on:

- **Core (this repo):** changes to the wire protocol — new packets, new fields, changed semantics, or any behaviour that cannot be expressed cleanly server-side. Core features must be sound by design; hacks and dirty workarounds do not belong in the protocol.
- **Scriptable (out of scope):** anything that can be implemented server-side in a clean, maintainable way (piqueserver scripts, gamemodes, admin tools). These are tracked elsewhere.

A feature is only eligible for the protocol if a server-side implementation would require unreasonable workarounds — abuse of unrelated packets, fragile state inference, or behaviour that contradicts the existing protocol. Convenience or duplication of effort across servers is not, on its own, sufficient justification.

## Process

1. **Open an issue.** One idea per issue. Describe the problem, the proposed packet/field changes, the expected behaviour, and the impact on existing clients/servers.
2. **Discuss.** The idea is debated in the issue. If it does not reach approval, it stops there.
3. **Vote.** Approved ideas are voted on. Only detailed, specified proposals are eligible — vague ideas are sent back to discussion.
4. **Pull request.** A PR implementing the documentation change must reference the originating issue. PRs without an approved issue will be closed.

Heavy features may be split across 0.77 and 0.78 if a single bump would be too disruptive. Call out the split in the issue.

## Version lifecycle

Each protocol version moves through three stages and never goes back:

1. **Open** — proposals accepted, discussion and votes happen.
2. **Feature freeze** — no new proposals; only clarifications, wording fixes, and corrections to what is already accepted.
3. **Released** — frozen forever. A released protocol version is immutable. Anything new goes into the next version.

## Security

Never trust data coming from the client. Clients can be modified, recompiled, or fully reimplemented; any field a client sends can be forged or tampered with. Every proposal that adds or changes a client→server packet must specify how the server validates it (bounds, ownership, rate, state). "The client should not send X" is not a defence.

## Sources

- https://www.piqueserver.org/aosprotocol/protocol075.html
- piqueserver (server, Python/Cython)
- zerospades (client, C++)
- BetterSpades (client, C)
