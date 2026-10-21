# Taproot and shared-output gossip

*A proposal for discussion, not spec text.*

This builds on René Malmgren's proposal ([multi-node-gossip][rene]) and Elle
Mouton's Gossip V2 draft ([lightning/bolts#1059][gv2]), for the Lightning
specification in `lightning-blake2b/bolts`: Lightning on Bitcoin since its
move to BLAKE2b proof of work at block 961,640.

## The problem

Gossip here can announce only P2WSH channels with two owners.
`bolt-simple-taproot.md` says taproot channels cannot be announced, so they
are private-only. An output with more than two owners cannot be announced at
all.

## First, a prerequisite

Since the BLAKE2b activation, a new channel's `channel_type` must include
`option_unified_sigs`, and `option_unified_sigs` is not yet defined for the
simple taproot channel types ([bolts#12][bolts12]). So at present a taproot
channel cannot be opened at all.

[bolts#13][bolts13] sets out what defining it would take: MuSig2 commitment
and closing signatures over the unified digest with hash type `0x21`, a
65-byte key-path witness, the HTLC hash types BOLT 3 already uses, unified
weight tables, and test vectors. As the discussion there concludes, it
changes no message. It also gives reasons to wait, one of which is that
taproot channels cannot be announced, which is what this proposal addresses.
The two can therefore move together. Everything below depends on it.

## The proposal

One change, in one spec PR: bring Gossip V2 into this specification with
René's changes in it from the start.

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

## Choices specific to this specification

| Topic | Proposal |
|---|---|
| Feature bits | Upstream's 70 to 75 become **516/517, 518/519 and 520/521**, next to `option_blake2b` and `option_unified_sigs`. Bits 256 to 267, 270/271, 512 to 515, 560 to 563, 729, 826 to 829 and 2022/2023 are taken or claimed by bLIPs, this specification or lnd (checked 22 Oct 2026). |
| V2 for every channel | V2 becomes the standard: new announcements use it, P2WSH channels included. BOLT 7 gossip stays for a transition period, so nodes that have not upgraded still see channels, and is dropped once Lightning Fork's and privkeyio's Core Lightning's releases both support V2. The timing is the maintainers' choice. |
| Activation height | As in BOLT 7 here: an output below the BLAKE2b activation height is never announced, and its gossip is ignored. |
| Replay | A tag of its own in `MsgHash` in place of `"lightning"`, so gossip signed for Bitcoin never verifies on Spam coin, the SHA256d chain, or the other way round. Without it, a Spam coin channel's funding transaction mined at the same position on Bitcoin would make its gossip valid here too: harmless, since payments over it fail, but avoidable at the cost of one constant. |
| Defects | Fix the two René found: the validation branch for absent `bitcoin_keys` uses `taproot_key` without requiring P2TR, and Appendix A sets `u` to 2 while listing four keys. |

## What has to be written here

Upstream left these as TODOs:

- **Sync.** `gossip_timestamp_filter` by block height, and
  `reply_channel_range` and `query_short_channel_ids` covering v2 messages.
- **Test vectors**, for two owners and for more.
- **Pathfinding for outputs with many owners**, since they are allowed from
  the start (open question 1).

## Where the code is today

- **Lightning Fork** (`paulscode/lightning-fork` `blake2b`): has upstream
  lnd's Gossip V2 groundwork (the `lnwire` messages, `validateChannelAnn2`,
  graph database migration `000009_graph_v2`), but the gossiper processes no
  v2 messages. `validateChannelAnn2` already aggregates a list of keys, so
  reading lists is a small change there.
- **privkeyio's Core Lightning** (`privkeyio/lightning` `blake2b-unified`):
  no Gossip V2 code.

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
6. **The tag string** for `MsgHash`.
7. **How long BOLT 7 gossip stays** during the move to V2.

## What it leaves room for

- **Liquidity exchange points**: a small group of always-online operators
  sharing one pool of liquidity.
- **Channel factories**, for example with ZmnSCPxj's *Pluggable Channel
  Factories* ([lightning/blips#56][blip56]).
- **Ark** rounds.
- **Later designs**, through `proof_type`.

Each would be its own design document.

## Order of work

1. `option_independent_secrets` ([bolts#17][bolts17] and its Core Lightning
   pull request) finishes first.
2. Taproot channels under `option_unified_sigs` are specified
   ([bolts#13][bolts13]).
3. The maintainers confirm or change the choices above.
4. One spec PR to `lightning-blake2b/bolts`.
5. Lightning Fork and privkeyio's Core Lightning implement it, with interop
   tests on a BLAKE2b regtest.

## Credits

- René Malmgren: the shared-output proposal, the two defects, and moving
  every channel to V2.
- Elle Mouton: the Gossip V2 draft.

## References

- [René Malmgren, multi-node-gossip proposal][rene]
- [Gossip V2 draft, lightning/bolts#1059][gv2]
- [`option_independent_secrets`, lightning-blake2b/bolts#17][bolts17]
- [`option_unified_sigs` is not defined for simple taproot channel types,
  lightning-blake2b/bolts#12][bolts12]
- [Defining `option_unified_sigs` for MuSig2, lightning-blake2b/bolts#13][bolts13]
- [ZmnSCPxj, *Pluggable Channel Factories*, lightning/blips#56][blip56]

[rene]: https://github.com/DarkWebDivingClub/bolts/blob/multi-node-gossip/taproot-proposal.md
[gv2]: https://github.com/lightning/bolts/pull/1059
[bolts17]: https://github.com/lightning-blake2b/bolts/pull/17
[bolts12]: https://github.com/lightning-blake2b/bolts/pull/12
[bolts13]: https://github.com/lightning-blake2b/bolts/issues/13
[blip56]: https://github.com/lightning/blips/pull/56
