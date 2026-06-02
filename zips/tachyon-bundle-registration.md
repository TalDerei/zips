    ZIP: Unassigned {bundle-type registration under ZIP 248; see "Bundle Type Registration"}
    Title: Tachyon Transaction Bundle
    Owners: Tal Derei <talderei99@gmail.com>
    Status: Draft
    Category: Consensus
    Created: 2026-06-02
    License: MIT
    Discussions-To: <https://github.com/tachyon-zcash/tachyon/pull/120>
    Pull-Request: <https://github.com/tachyon-zcash/zips/pull/???>


# Terminology

The key words "MUST", "SHOULD", and "MAY" in this document are to be interpreted
as described in BCP 14 [^BCP14] when, and only when, they appear in all capitals.

This proposal depends on the Extensible Transaction Format defined in ZIP 248
[^zip-0248] (currently a draft). The terms "protocol bundle", "bundle type
registry", `bundleType`, `bundleVariant`, "effecting data", "authorizing data",
and the value-pool delta map `mValuePoolDeltas` are to be interpreted as defined
in ZIP 248. The terms "txid" and "wtxid" are to be interpreted as defined in
ZIP 244. [^zip-0244]

The character § is used when referring to sections of the Zcash Protocol
Specification. [^protocol]

The terms below are specific to the Tachyon shielded protocol
[^tachyon-protocol] and the Tachyon bundle format [^tachyon-bundle], and are to
be interpreted as follows.

tachyaction

: A `{cv, rk, sig}` triple: a value commitment `cv` (32 bytes), a randomized
  verification key `rk` (32 bytes), and a RedPallas signature `sig` (64 bytes).
  The `cv` and `rk` are effecting data; the `sig` is authorizing data.

stamp

: The aggregation-related fields of a Tachyon bundle (anchor, tachygrams, and
  proof). A bundle that carries its stamp is "stamped" (`tachyonBundleState`
  `0x01`); a bundle whose stamp has been removed during aggregation is "stripped"
  (`0x02`) and instead references the covering aggregate by `tachyonAggregateId`.

`valueBalanceTachyon`

: The signed net value transferred between the Tachyon pool and the transparent
  value pool by a bundle, as defined by the Tachyon shielded protocol.
  [^tachyon-protocol] It is the Tachyon entry in `mValuePoolDeltas`.

Tachyon network upgrade

: The network upgrade that activates the Tachyon shielded protocol.
  [^tachyon-protocol]


# Abstract

ZIP 248 [^zip-0248] defines an Extensible Transaction Format in which a V6
transaction is a type-length-value sequence of protocol bundles, each identified
by a `(bundleType, bundleVariant)` pair. A wallet that does not understand a
bundle type can skip it, still compute the txid from the opaque effecting data,
and still account for its value flows via `mValuePoolDeltas`.

This proposal registers the Tachyon shielded pool [^tachyon-protocol] as a bundle
type under ZIP 248: it requests a `(bundleType, bundleVariant)` pair and specifies
the Tachyon bundle's effecting data, authorizing data, and its contributions to
the txid and the authorizing-data commitment. The bundle commits only the public
`valueBalanceTachyon` to `mValuePoolDeltas`, so it has no privacy implications
beyond those already present in the Tachyon shielded protocol. This proposal is
contingent on ZIP 248 being finalized; see Open Issues.


# Motivation

Historically, adding a shielded pool (Sapling, then Orchard) changed the
monolithic transaction format and forced every wallet — even transparent-only
ones — to update their parser, occasionally locking funds in wallets that could
not parse post-upgrade transactions. ZIP 248 [^zip-0248] removes this coupling:
new pools register a bundle type and slot into the TLV sequence, and
non-supporting wallets skip unknown bundles while still computing the txid and
tracking value flows.

The Tachyon shielded protocol [^tachyon-protocol] introduces a new pool. Rather
than hardcoding a Tachyon field in the V6 transaction structure, this proposal
registers it as a ZIP 248 bundle type, so that `librustzcash` and other
implementations emit a Tachyon bundle in the TLV sequence and wallets that do not
support Tachyon can ignore it safely.


# Privacy Implications

A Tachyon bundle's only contribution to a wallet that does not support it is its
opaque effecting data (counted toward the txid) and its `valueBalanceTachyon`
entry in `mValuePoolDeltas`. `valueBalanceTachyon` is already a public value of
every Tachyon-bearing transaction, identical to the field ZIP 209 tracks for the
turnstile. This proposal adds no new per-transaction information and therefore
has no privacy implications beyond those already present in the Tachyon shielded
protocol.


# Requirements

The registration must:

- allocate a `(bundleType, bundleVariant)` pair for the Tachyon bundle in the
  ZIP 248 bundle type registry, declaring that it contributes to
  `mValuePoolDeltas`, effecting bundles, and authorizing bundles;
- specify the Tachyon bundle's effecting data and authorizing data, with the
  effecting/authorizing split matching the existing txid/wtxid separation defined
  in ZIP 244 [^zip-0244];
- specify the bundle's contribution to the txid such that it is invariant under
  stripping the stamp during aggregation; and
- be expressible entirely through the ZIP 248 registration process, requiring no
  change to ZIP 248 itself.


# Specification

This specification follows the bundle type registration process defined in
ZIP 248 [^zip-0248]. It is normative only if and when ZIP 248 is finalized; see
Open Issues.

## Bundle Type Registration

Request allocation of a `(bundleType, bundleVariant)` pair in the V6 transaction
bundle type registry:

| Registry field | Value |
| -------------- | ----- |
| `bundleType` | TBD (requested allocation) |
| `bundleVariant` | `0` |
| `mValuePoolDeltas` | yes — the Tachyon pool has a value balance (`valueBalanceTachyon`) |
| `mEffectBundles` | yes |
| `mAuthBundles` | yes |

## Effecting Data

The effecting data for a Tachyon bundle encodes:

| Field | Size | Description |
| ----- | ---- | ----------- |
| `nActionsTachyon` | compactSize | number of tachyactions |
| `vActionsTachyon` | `64 * nActionsTachyon` | `cv` (32 bytes) + `rk` (32 bytes) per action |
| `tachyonBundleState` | 1 byte | `0x01` = stamped, `0x02` = stripped |
| `valueBalanceTachyon` | 8 bytes | signed net value (also the `mValuePoolDeltas` entry) |
| stamped only: `anchorTachyon` | 32 bytes | pool state reference |
| stamped only: `nTachygrams` | compactSize | number of tachygrams |
| stamped only: `vTachygrams` | `32 * nTachygrams` | tachygram field elements |
| stamped only: `proofTachyon` | `PROOF_SIZE_COMPRESSED` | recursive proof |
| stripped only: `tachyonAggregateId` | 64 bytes | `wtxid` of the covering aggregate |

## Authorizing Data

| Field | Size | Description |
| ----- | ---- | ----------- |
| `vActionSigsTachyon` | `64 * nActionsTachyon` | RedPallas signature per action |
| `bindingSigTachyon` | 64 bytes | binding signature |

## Transaction Identifier Contribution

The Tachyon bundle's contribution to the txid is:

```
BLAKE2b-512("ZTxIdTachyonHash", action_acc || valueBalanceTachyon)
```

where `action_acc` is the accumulated commitment over the per-action effecting
data (`cv`, `rk`). This excludes the stamp (which is stripped during aggregation)
and the signatures (which are authorizing data), so the txid is invariant under
stripping.

## Authorizing Data Commitment Contribution

The Tachyon bundle's contribution to the authorizing-data commitment (the
wtxid component, per ZIP 244 [^zip-0244]) depends on `tachyonBundleState`: a
stamped bundle commits its action signatures, binding signature, and stamp,
whereas a stripped bundle commits its action signatures, binding signature, and
`tachyonAggregateId`. The precise formula is specified by the Tachyon bundle
format. [^tachyon-bundle]


# Rationale

The effecting/authorizing split required by ZIP 248 already matches the Tachyon
bundle's existing structure: the value commitments and randomized verification
keys (`cv`, `rk`), the value balance, and the stamp or stripped reference are
effecting data that contribute to the txid, while the signatures are authorizing
data that contribute to the wtxid. Defining the bundle this way means a Tachyon
transaction's txid is unchanged when its stamp is stripped during aggregation —
the same invariance the txid scheme of ZIP 244 [^zip-0244] provides for other
bundles.

Registering Tachyon as a bundle type, rather than amending the transaction
structure, is the entire purpose of ZIP 248: it lets the new pool be added
without changing the parser of any wallet that does not support it. The value
flow remains visible to all wallets through `mValuePoolDeltas`, so even a wallet
that skips the Tachyon bundle can compute correct value balances.


# Deployment

Subject to Open Issues, the Tachyon bundle type is available from the activation
of the Tachyon network upgrade [^tachyon-protocol]. Before that upgrade
activates, no transaction contains a Tachyon bundle.


# Open Issues

- **Dependency on ZIP 248.** This proposal is contingent on ZIP 248
  [^zip-0248] being finalized. Until the ZIP Editors reach consensus on the
  precise definition and interpretation of the Extensible Transaction Format,
  the registry semantics assumed here (the `(bundleType, bundleVariant)` model,
  the `mValuePoolDeltas`/`mEffectBundles`/`mAuthBundles` flags, and the
  effecting/authorizing serialization) may change, and this proposal is blocked.
- **`bundleType` allocation.** The concrete `bundleType` value is to be assigned
  through the ZIP 248 registry once that registry exists.


# Reference implementation

Under this proposal, `librustzcash` and other implementations emit a Tachyon
bundle in the V6 TLV sequence rather than hardcoding a Tachyon field in a V6
transaction structure. The effecting and authorizing data are serialized as
specified above, `valueBalanceTachyon` is placed in the `mValuePoolDeltas` map
keyed by the Tachyon `bundleType`, and the txid/auth-digest contributions are
computed as specified.


# References

[^BCP14]: [Information on BCP 14 — "RFC 2119: Key words for use in RFCs to Indicate Requirement Levels" and "RFC 8174: Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words"](https://www.rfc-editor.org/info/bcp14)

[^protocol]: [Zcash Protocol Specification, Version 2025.6.3 [NU6.1] or later](protocol/protocol.pdf)

[^zip-0244]: [ZIP 244: Transaction Identifier Non-Malleability](zip-0244.rst)

[^zip-0248]: [ZIP 248: Extensible Transaction Format (draft, zcash/zips#1156)](https://github.com/zcash/zips/pull/1156)

[^tachyon-protocol]: [Tachyon Shielded Protocol (tachyon-zcash/tachyon issue #103)](https://github.com/tachyon-zcash/tachyon/issues/103)

[^tachyon-bundle]: [Tachyon Bundle (tachyon-zcash/tachyon issue #104)](https://github.com/tachyon-zcash/tachyon/issues/104)
