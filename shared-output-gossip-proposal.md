# Taproot and shared-output gossip for the BLAKE2b fork

*A proposal for discussion, not spec text.*

This builds on René Malmgren's proposal ([multi-node-gossip][rene]) and Elle
Mouton's Gossip V2 draft ([lightning/bolts#1059][gv2]).

## The problem

The fork's gossip can announce only P2WSH channels with two owners.
`bolt-simple-taproot.md` says taproot channels cannot be announced, so on the
fork they are private-only. An output with more than two owners cannot be
announced at all.

## The proposal

One change, in one spec PR: bring Gossip V2 into the fork with René's
changes in it from the start.

- `channel_announcement_2` carries `node_ids` and `bitcoin_keys` as lists
  instead of exactly two of each.
- `channel_update_2` names its signer with `scid_endpoint` (the
  `short_channel_id` plus an index into `node_ids`), so each owner publishes
  its own routing policy.
- A `proof_type` field names how the announcement is verified, leaving room
  for other proofs later.

Every channel then uses the same standard: an ordinary taproot channel is
the case with two entries, and an output with many owners (an Ark round, a
liquidity exchange point) uses the same messages. Doing it in one go means
the network never has two kinds of channel to tell apart. Upstream has
stalled and nothing deploys Gossip V2 yet, so the format can still be chosen
freely.

How a shared output works inside is not part of this proposal. That is
software only its members run, and different groups can build it
differently.

## Why it matters

- **Public taproot channels.** They use less block space, and a cooperative
  close looks like an ordinary spend.
- **Public k-of-n routing nodes.** A k-of-n node can already route with
  P2WSH channels, but splitting an ECDSA signature across signers is hard.
  With taproot channels each signature is Schnorr, where threshold signing
  is much simpler (MuSig2, FROST). `option_independent_secrets`
  ([bolts#17][bolts17]) already works with taproot channel types; this
  change lets those channels route publicly.
- **Shared outputs.** Several operators can pool capital in one output,
  route through it, and prove its size and membership publicly.

## Fork-specific choices

| Topic | Proposal |
|---|---|
| Feature bits | Upstream's 70 to 75 become **516/517, 518/519 and 520/521**, next to the fork's own bits. Bits 256 to 267, 270/271, 512 to 515, 560 to 563, 729, 826 to 829 and 2022/2023 are taken or claimed by bLIPs, the fork or lnd (checked 22 Oct 2026). |
| Existing P2WSH channels | Move to v2 only if their node opts in (`option_gossip_v2_p2wsh`, as upstream). Otherwise they stay on BOLT 7. |
| Activation height | As in the fork's BOLT 7: an output below the BLAKE2b activation height is never announced, and its gossip is ignored. |
| Replay | A fork-specific tag in `MsgHash` in place of `"lightning"`, so gossip signed for one chain never verifies on the other. Without it, a main-network channel's funding transaction mined at the same position on the fork would make its gossip valid there too: harmless, since payments over it fail, but avoidable at the cost of one constant. |
| Defects | Fix the two René found: the validation branch for absent `bitcoin_keys` uses `taproot_key` without requiring P2TR, and Appendix A sets `u` to 2 while listing four keys. |

## What the fork has to write itself

Upstream left these as TODOs:

- **Sync.** `gossip_timestamp_filter` by block height, and
  `reply_channel_range` and `query_short_channel_ids` covering v2 messages.
- **Test vectors**, for two owners and for more.
- **Pathfinding for outputs with many owners**, since they are allowed from
  the start (open question 1).

## Where the code is today

- **lnd fork** (`paulscode/lightning-fork` `blake2b`): has upstream's Gossip
  V2 groundwork (the `lnwire` messages, `validateChannelAnn2`, graph database
  migration `000009_graph_v2`), but the gossiper processes no v2 messages.
  `validateChannelAnn2` already aggregates a list of keys, so reading lists
  is a small change there.
- **CLN fork** (`privkeyio/lightning` `blake2b-unified`): no Gossip V2 code.

## Open questions

1. **More than two owners from the first version?** René's view, and this
   note's: yes. Turning it on later would split the network into nodes that
   understand shared outputs and nodes that do not. The cost is that
   pathfinding has to handle them from the start.
2. **Capacity.** If every owner may advertise up to the output's capacity,
   the total can exceed what is locked. Require the owners' maximums to add
   up to no more than the capacity, or leave pathfinders to treat it as
   shared?
3. **Graph model.** Should a receiver treat a shared output as one edge with
   many ends, or as pairs of owners with a shared limit?
4. **Expiry.** Some shared outputs have a deadline (an Ark round expires).
   Should the announcement say so?
5. **Script paths.** Should a verifier be able to tell that an output has no
   script path, by comparing `merkle_root_hash` with the internal key?
6. **The fork's tag string** for `MsgHash`.

## What it leaves room for

- **Liquidity exchange points**: a small group of always-online operators
  sharing one pool of liquidity.
- **Channel factories**, for example with ZmnSCPxj's *Pluggable Channel
  Factories* ([lightning/blips#56][blip56]).
- **Ark** rounds.
- **Later designs**, through `proof_type`.

Each would be its own design document.

## Order of work

1. `option_independent_secrets` ([bolts#17][bolts17] and its lnd and CLN
   pull requests) finishes first.
2. The maintainers confirm or change the choices above.
3. One spec PR to `lightning-blake2b/bolts`.
4. lnd and CLN implement it, with interop tests on a BLAKE2b regtest.

## Credits

- René Malmgren: the shared-output proposal and the two defects.
- Elle Mouton: the Gossip V2 draft.

## References

- [René Malmgren, multi-node-gossip proposal][rene]
- [Gossip V2 draft, lightning/bolts#1059][gv2]
- [`option_independent_secrets`, lightning-blake2b/bolts#17][bolts17]
- [ZmnSCPxj, *Pluggable Channel Factories*, lightning/blips#56][blip56]

[rene]: https://github.com/DarkWebDivingClub/bolts/blob/multi-node-gossip/taproot-proposal.md
[gv2]: https://github.com/lightning/bolts/pull/1059
[bolts17]: https://github.com/lightning-blake2b/bolts/pull/17
[blip56]: https://github.com/lightning/blips/pull/56
