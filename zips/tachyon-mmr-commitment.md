    ZIP: Unassigned {to be folded into ZIP 221; see "Changes to ZIP 221"}
    Title: Tachyon Chain History Commitment
    Owners: Tal Derei <talderei99@gmail.com>
    Status: Draft
    Category: Consensus
    Created: 2026-06-02
    License: MIT
    Discussions-To: <https://github.com/tachyon-zcash/tachyon/pull/119>
    Pull-Request: <https://github.com/tachyon-zcash/zips/pull/???>


# Terminology

The key words "MUST", "MUST NOT", "SHOULD", and "MAY" in this document are to be
interpreted as described in BCP 14 [^BCP14] when, and only when, they appear in
all capitals.

The terms "consensus branch", "epoch", and "network upgrade" are to be
interpreted as defined in ZIP 200. [^zip-0200]

The terms "light client", "Merkle Mountain Range" (MMR), "node", and "leaf", and
the block-commitment construction `hashBlockCommitments`, are to be interpreted
as defined in ZIP 221. [^zip-0221]

The character § is used when referring to sections of the Zcash Protocol
Specification. [^protocol]

The terms below are specific to the Tachyon shielded protocol [^tachyon-protocol]
and are to be interpreted as follows.

Tachyon anchor

: The end-of-block state of the Tachyon accumulator [^tachyon-accumulator] — a
  sequential Poseidon hash chain that absorbs each stamp's tachygram-set
  commitment coordinates. It is a 32-byte $\mathbb{F}_p$ element and plays the
  same role for the Tachyon pool that a note commitment tree root plays for
  Sapling and Orchard: the value that a spendable note references to prove it is
  spending against a valid pool state.

`nTachyonTxCount`

: The number of transactions in a block that contain Tachyon actions.

Tachyon network upgrade

: The network upgrade that activates the Tachyon shielded protocol.
  [^tachyon-protocol]


# Abstract

ZIP 221 [^zip-0221] replaces the block-header field `hashFinalSaplingRoot` with
`hashBlockCommitments`, a commitment to a Merkle Mountain Range (MMR) over block
history. Each MMR leaf carries per-pool metadata — for Sapling and Orchard, the
note commitment tree root and a transaction count — letting light clients verify
chain history and pool state in logarithmic time (the FlyClient protocol).

This proposal extends the MMR leaf/node definition to commit Tachyon pool state.
Because the Tachyon pool has no note commitment tree, the committed "root" is the
end-of-block Tachyon anchor [^tachyon-accumulator], which serves the same role as
a treestate root. It adds `hashEarliestTachyonRoot`, `hashLatestTachyonRoot`, and
`nTachyonTxCount`, mirroring the Sapling and Orchard fields. The change is
additive against ZIP 221 and mirrors how the Orchard fields were added at NU5. It
commits only public, block-level values and therefore has no privacy
implications beyond those already present in ZIP 221.


# Motivation

ZIP 221 lets light clients — which do not store or validate the block chain —
verify chain history and per-pool state efficiently. For Sapling and Orchard, the
MMR leaf commits the pool's note commitment tree root, so a light client can
verify the treestate a transaction anchors against, and a per-pool transaction
count, so it can detect withheld transactions and skip block ranges with no
activity in that pool.

The Tachyon shielded protocol [^tachyon-protocol] introduces a new pool whose
spendable notes anchor against the Tachyon anchor [^tachyon-accumulator] rather
than a note commitment tree root. Without extending ZIP 221, light clients would
have no committed, verifiable view of Tachyon pool state at block boundaries, and
no way to account for Tachyon activity per block.


# Privacy Implications

The fields added by this proposal are block-level public values: the end-of-block
Tachyon anchor and the count of transactions containing Tachyon actions. The
anchor is already a public consequence of the block's contents, and
`nTachyonTxCount` is analogous to the existing `nSaplingTxCount` and
`nOrchardTxCount` fields. This proposal adds no per-transaction information and
therefore has no privacy implications beyond those already present in ZIP 221.


# Requirements

The extension to ZIP 221 must:

- commit, in each MMR leaf, a value sufficient for a light client to verify
  Tachyon pool state at block boundaries — namely the end-of-block Tachyon
  anchor, which plays the role that the note commitment tree root plays for the
  existing pools;
- commit a per-block count of Tachyon-bearing transactions, with the same
  internal-node aggregation rule as the existing transaction-count fields;
- define values for these fields for blocks before the Tachyon network upgrade
  activates; and
- be an additive, editorial change to ZIP 221 that leaves the existing node
  fields and the rest of the construction unchanged.


# Specification

## Changes to ZIP 221

The following changes are to be applied to ZIP 221 [^zip-0221]. They are
additive: no existing node field is modified or removed, and they follow the
existing Orchard fields (`hashEarliestOrchardRoot`, `hashLatestOrchardRoot`,
`nOrchardTxCount`).

Append the following fields to the MMR leaf/node definition, present from the
Tachyon network upgrade onward:

| Field | Type | Leaf value | Internal-node value |
| ----- | ---- | ---------- | ------------------- |
| `hashEarliestTachyonRoot` | `char[32]` | End-of-block Tachyon anchor | Inherited from the left child |
| `hashLatestTachyonRoot` | `char[32]` | End-of-block Tachyon anchor | Inherited from the right child |
| `nTachyonTxCount` | CompactSize | Number of transactions in the block that contain Tachyon actions | The sum of the `nTachyonTxCount` field of both children |

Before the Tachyon network upgrade has activated, `hashEarliestTachyonRoot` and
`hashLatestTachyonRoot` MUST be set to all-zero bytes, and `nTachyonTxCount`
MUST be zero.

The node serialized size is updated accordingly: the three additional fields
contribute 32 bytes, 32 bytes, and the encoded length of a CompactSize,
respectively, and are serialized after the Orchard fields.

## Folding into ZIP 221

This proposal is intended to be folded into ZIP 221's text by the ZIP Editors,
following the precedent by which the Orchard fields were added to ZIP 221 for
NU5, rather than deployed as a standalone numbered ZIP. The normative content
above is expected to be incorporated as part of the Tachyon shielded protocol
specification [^tachyon-protocol], which carries a "Changes to ZIP 221" section.
ZIP 221 would then gain an `Updated-By` header referencing that specification
once it reaches a Released status.


# Rationale

The Tachyon anchor is the end-of-block Tachyon accumulator state
[^tachyon-accumulator], and it serves the same role as a note commitment tree
root for Sapling and Orchard: spendable notes reference an anchor to prove they
are spending against a valid pool state. Committing the end-of-block anchor in
the MMR leaf therefore gives light clients the same verification capability for
the Tachyon pool that `hashLatestSaplingRoot` and `hashLatestOrchardRoot` give
for theirs. The anchor is a 32-byte $\mathbb{F}_p$ element — the same width as the
existing roots — so it fits the existing `char[32]` field layout without changing
the node format beyond appending fields.

`nTachyonTxCount` follows the same rationale as `nSaplingTxCount` and
`nOrchardTxCount`: it lets light clients detect withheld transactions and skip
block ranges with no Tachyon activity, reducing bandwidth. Using the same
"sum of both children" aggregation rule keeps the internal-node construction
uniform across pools.

Two anchor fields (`hashEarliest…`/`hashLatest…`) are included, mirroring the
Sapling and Orchard pairs, so that an internal node can carry the boundary anchors
of the block range it covers. For a leaf, both equal the block's end-of-block
anchor, exactly as the existing pools set both root fields to the block's root.


# Deployment

These fields are present in the MMR leaf/node definition from the activation of
the Tachyon network upgrade [^tachyon-protocol] onward, analogously to how the
Orchard fields were deployed from NU5 activation [^zip-0252]. For blocks before
the Tachyon network upgrade has activated, the two anchor fields are all-zero and
`nTachyonTxCount` is zero, as specified above.


# Reference implementation

Consensus nodes already construct each MMR leaf from per-block pool metadata and
aggregate internal nodes from their children. Supporting this proposal requires
populating the two Tachyon anchor fields from the block's end-of-block Tachyon
anchor and `nTachyonTxCount` from the count of Tachyon-bearing transactions, then
aggregating them with the same rules as the Orchard fields. No other change to
the MMR construction is required.


# References

[^BCP14]: [Information on BCP 14 — "RFC 2119: Key words for use in RFCs to Indicate Requirement Levels" and "RFC 8174: Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words"](https://www.rfc-editor.org/info/bcp14)

[^protocol]: [Zcash Protocol Specification, Version 2025.6.3 [NU6.1] or later](protocol/protocol.pdf)

[^zip-0200]: [ZIP 200: Network Upgrade Mechanism](zip-0200.rst)

[^zip-0221]: [ZIP 221: FlyClient - Consensus-Layer Changes](zip-0221.rst)

[^zip-0252]: [ZIP 252: Deployment of the NU5 Network Upgrade](zip-0252.rst)

[^tachyon-protocol]: [Tachyon Shielded Protocol (tachyon-zcash/tachyon issue #103)](https://github.com/tachyon-zcash/tachyon/issues/103)

[^tachyon-accumulator]: [Tachyon Accumulator (tachyon-zcash/tachyon issue #105)](https://github.com/tachyon-zcash/tachyon/issues/105)
