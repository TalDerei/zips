    ZIP: Unassigned {to be folded into ZIP 317; see "Changes to ZIP 317"}
    Title: Tachyon Action Fee Contribution
    Owners: Tal Derei <talderei99@gmail.com>
    Status: Draft
    Category: Standards / Wallet
    Created: 2026-06-02
    License: MIT
    Discussions-To: <https://github.com/tachyon-zcash/tachyon/pull/116>
    Pull-Request: <https://github.com/tachyon-zcash/zips/pull/???>


# Terminology

The key words "MUST", "SHOULD", "RECOMMENDED", and "MAY" in this document are to
be interpreted as described in BCP 14 [^BCP14] when, and only when, they appear
in all capitals.

The terms "conventional transaction fee", "logical action", "conventional fee",
and "marginal fee" are to be interpreted as defined in ZIP 317. [^zip-0317]

The character § is used when referring to sections of the Zcash Protocol
Specification. [^protocol]

The terms below are specific to the Tachyon shielded protocol [^tachyon-protocol]
and are to be interpreted as follows.

tachyaction

: A `{cv, rk, sig}` triple — a value commitment `cv` (32 bytes), a randomized
  verification key `rk` (32 bytes), and a RedPallas signature `sig` (64 bytes),
  for a total of 128 bytes. A tachyaction lives in the transaction regardless of
  stamp type.

stamp

: The aggregation-related fields of a Tachyon bundle (the proof, tachygrams, and
  anchor). A bundle that carries its stamp is "stamped"; a bundle whose stamp
  has been removed during aggregation is "stripped".

tachygram

: A 32-byte commitment published by a stamped bundle and subject to a
  duplicate check across a two-epoch window. Tachygrams are stamp fields.

autonome

: A stamped bundle produced by a wallet, carrying its own tachyactions, proof,
  tachygrams, and anchor.

aggregate

: A stamped bundle produced by an aggregator whose proof covers the tachyactions
  of one or more adjuncts. An aggregate MAY carry its own tachyactions (a "based"
  aggregate) or none (an "innocent" aggregate).

adjunct

: A stripped bundle produced by a miner. It carries tachyactions but no proof,
  no tachygrams, and no anchor; it references the covering aggregate by `wtxid`.

`nActionsTachyon`

: The number of tachyactions in a transaction, defined by the Tachyon shielded
  protocol. [^tachyon-protocol]


# Abstract

ZIP 317 [^zip-0317] defines a proportional fee mechanism in which the
conventional fee scales with the number of *logical actions* a transaction
imposes on the network, summed across protocols (transparent, Sprout, Sapling,
Orchard, ZSA issuance, and memos). The Tachyon shielded protocol
[^tachyon-protocol] introduces a new bundle type that is not accounted for in
that sum.

This proposal defines the Tachyon contribution to the ZIP 317 logical-action
count. A transaction's Tachyon contribution is its tachyaction count,
$\mathit{contribution}_{\,\mathsf{Tachyon}} = \mathit{nActionsTachyon}$, mirroring
the treatment of Orchard actions. The change is editorial against ZIP 317: it
adds one input variable and one term to the existing formula and does not alter
any consensus rule. As with ZIP 317 generally, the fee remains a wallet
convention rather than a consensus requirement, so this proposal has no direct
privacy implications beyond the existing recommendation that wallets pay the
conventional fee to reduce information leakage.


# Motivation

ZIP 317 prices the verification and state cost a transaction imposes on
validators, normalized into protocol-neutral logical actions, so that fees scale
with network impact and DoS vectors such as sandblasting become comparatively
more expensive. The formula is, by design, a pure function of a single
transaction's public fields.

Tachyon adds a bundle type that ZIP 317 does not cover, and it does so under an
aggregation model that has no precedent in the existing pools. The same logical
work — tachyactions plus tachygrams — can be split across two transactions: an
adjunct carries the tachyactions while the aggregate carries the proof that
covers them. Without an explicit Tachyon term, Tachyon transactions would
contribute zero logical actions and pay only the grace-window minimum,
regardless of size.

Following the precedent of ZIP 227 (ZSA issuance) [^zip-0227] and ZIP 231 (memo
bundles) [^zip-0231], which each defined their own ZIP 317 contributions that
were then folded into ZIP 317's text, this proposal defines
$\mathit{contribution}_{\,\mathsf{Tachyon}}$ so that Tachyon transactions are
priced consistently with the other pools.


# Privacy Implications

ZIP 317 fees are a wallet convention, not a consensus rule. This proposal does
not change that. It does not add or remove any transaction field, and it does
not change which fields are public. As recommended by ZIP 317, wallets SHOULD
create transactions that pay the conventional fee, in order to reduce
information leakage from fee selection, unless overridden by the user. No
additional privacy considerations are introduced.


# Requirements

The Tachyon contribution to the ZIP 317 logical-action count must:

- price the verification and state cost that a Tachyon transaction imposes on
  validators, consistently with how the existing pools are priced;
- preserve the property that $\mathit{conventional\_actions}()$ is a pure
  function of a single transaction's public fields, requiring no cross-
  transaction knowledge of which aggregate covers which adjuncts; and
- be an additive, editorial change to ZIP 317 that leaves every other term, and
  all downstream behaviour (the conventional fee, unpaid actions, and mempool
  checks), unchanged.


# Non-requirements

This proposal does not attempt to price the aggregator's proof work
(decompression, recursive composition, recompression). As in ZIP 317, that cost
is a market and protocol-design concern outside the scope of the fee formula.

This proposal does not introduce per-stamp-type fee differentiation, nor any
term that would require cross-transaction knowledge. See the Rationale for the
alternatives considered.


# Specification

## Changes to ZIP 317

The following changes are to be applied to the "Fee calculation" section of
ZIP 317 [^zip-0317-fee-calculation]. They are additive: no existing parameter,
input, or term is modified or removed.

Add the following input variable to the table of inputs taken from transaction
fields:

| Input             | Units  | Description                                       |
| ----------------- | ------ | ------------------------------------------------- |
| `nActionsTachyon` | number | the number of Tachyon actions (tachyactions)      |

Add the following term to the logical-action contributions:

$$\mathit{contribution}_{\,\mathsf{Tachyon}} = \mathit{nActionsTachyon}$$

such that the $\mathit{logical\_actions}$ sum becomes:

$$
\begin{array}{lcl}
  \mathit{logical\_actions} &=& \mathit{contribution}_{\,\mathsf{Transparent}} +
                                \mathit{contribution}_{\,\mathsf{Sprout}} +
                                \mathit{contribution}_{\,\mathsf{Sapling}} +
                                \mathit{contribution}_{\,\mathsf{Orchard}} \\
  & & +\; \mathit{contribution}_{\,\mathsf{Tachyon}} \\
  & & +\; \mathit{contribution}_{\,\mathsf{ZSAIssuance}} +
          \mathit{contribution}_{\,\mathsf{ZSACreation}} +
          \mathit{contribution}_{\,\mathsf{Memos}}
\end{array}
$$

The $\mathit{contribution}_{\,\mathsf{Tachyon}}$ term applies identically to all
Tachyon bundle types: a transaction's contribution is its tachyaction count
whether the bundle is an autonome, an aggregate, or an adjunct.

As with the rest of ZIP 317, it is not a consensus requirement that fees follow
this formula; however, wallets SHOULD create transactions that pay the resulting
conventional fee, in order to reduce information leakage, unless overridden by
the user.

## Folding into ZIP 317

This proposal is intended to be folded into ZIP 317's text by the ZIP Editors,
following the precedent of ZIP 227 [^zip-0227] and ZIP 231 [^zip-0231], rather
than deployed as a standalone numbered ZIP. The normative content above
(`nActionsTachyon` and $\mathit{contribution}_{\,\mathsf{Tachyon}}$) is expected
to be incorporated as part of the Tachyon shielded protocol specification
[^tachyon-protocol], which carries a "Changes to ZIP 317" section. ZIP 317 would
then gain an `Updated-By` header referencing that specification once it reaches a
Released status.


# Rationale

## Rationale for pricing tachyactions

A tachyaction is structurally analogous to an Orchard action. Each contains a
value commitment `cv` (32 bytes), a randomized verification key `rk` (32 bytes),
and a RedPallas signature `sig` (64 bytes), for 128 bytes per action. Validators
verify one RedPallas signature per tachyaction, regardless of stamp type.
Pricing the tachyaction count therefore mirrors
$\mathit{contribution}_{\,\mathsf{Orchard}} = \mathit{nActionsOrchard}$ and
preserves ZIP 317's protocol-neutral treatment of actions.

Tachygrams and the proof are stamp fields, not action fields. The stamp may be
stripped during aggregation (producing an adjunct), but the tachyactions remain
in the transaction. Pricing tachyactions alone preserves the property that
$\mathit{conventional\_actions}()$ is a pure function of a single transaction's
public fields, without requiring cross-transaction knowledge of which aggregate
covers which adjuncts.

The aggregator's proof cost (verification of one recursive proof per stamped
bundle) is not separately priced. This follows the same rationale as Orchard,
where the Halo 2 proof-verification cost is implicitly covered by the per-action
fee rather than charged as a separate term.

## Rationale against alternatives considered

Two alternatives were considered and rejected.

The first, $\mathit{contribution}_{\,\mathsf{Tachyon}} =
\mathsf{max}(\mathit{nActionsTachyon}, \mathit{nTachygrams})$, prices the
duplicate-check state cost of tachygrams explicitly. Because a spend publishes
two tachygrams (current- and next-epoch nullifiers) but only one tachyaction, it
charges spend-heavy transactions more, introducing a spend-versus-output
asymmetry that ZIP 317 does not have for the other pools.

The second differentiates by stamp type, charging stamped bundles for
proof, tachygrams, and actions while charging stripped adjuncts for actions
only. This creates an incentive for a wallet to have its transaction become an
adjunct in order to pay less, and — more fundamentally — the adjunct's omitted
cost (tachygrams plus proof) must be shifted onto the covering aggregate. The
aggregate would then either overpay or require cross-transaction knowledge,
breaking ZIP 317's per-transaction fee isolation.

Pricing tachyactions alone (the specification above) avoids both the
spend/output asymmetry and the cross-transaction dependency, at the cost of
treating the aggregate's proof verification as an unpriced network
optimization — consistent with how Orchard treats proof verification today.


# Reference implementation

In Zebra, `conventional_actions()`
[^zebra-zip317] computes the logical-action sum. A
$\mathit{contribution}_{\,\mathsf{Tachyon}}$ term equal to `nActionsTachyon`
is added to that sum. Everything downstream — `conventional_fee`,
`unpaid_actions`, and the mempool checks — operates on its output and is
unchanged.


# References

[^BCP14]: [Information on BCP 14 — "RFC 2119: Key words for use in RFCs to Indicate Requirement Levels" and "RFC 8174: Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words"](https://www.rfc-editor.org/info/bcp14)

[^protocol]: [Zcash Protocol Specification, Version 2025.6.3 [NU6.1] or later](protocol/protocol.pdf)

[^zip-0317]: [ZIP 317: Proportional Transfer Fee Mechanism](zip-0317.rst)

[^zip-0317-fee-calculation]: [ZIP 317: Proportional Transfer Fee Mechanism, Section "Fee calculation"](zip-0317.rst)

[^zip-0227]: [ZIP 227: Issuance of Zcash Shielded Assets](zip-0227.rst)

[^zip-0231]: [ZIP 231: Memo Bundles](zip-0231.md)

[^tachyon-protocol]: [Tachyon Shielded Protocol (tachyon-zcash/tachyon issue #103)](https://github.com/tachyon-zcash/tachyon/issues/103)

[^zebra-zip317]: [Zebra: `zebra-chain/src/transaction/unmined/zip317.rs`](https://github.com/ZcashFoundation/zebra/blob/main/zebra-chain/src/transaction/unmined/zip317.rs)
