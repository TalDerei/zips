    ZIP: Unassigned {to be folded into ZIP 209; see "Changes to ZIP 209"}
    Title: Tachyon Chain Value Pool Balance
    Owners: Tal Derei <talderei99@gmail.com>
    Status: Draft
    Category: Consensus
    Created: 2026-06-02
    License: MIT
    Discussions-To: <https://github.com/tachyon-zcash/tachyon/pull/118>
    Pull-Request: <https://github.com/tachyon-zcash/zips/pull/???>


# Terminology

The key words "MUST", "SHOULD", and "MAY" in this document are to be interpreted
as described in BCP 14 [^BCP14] when, and only when, they appear in all capitals.

The terms "block chain" and "network upgrade" are to be interpreted as defined in
ZIP 200. [^zip-0200]

The terms "Sprout chain value pool balance", "Sapling chain value pool balance",
and "Orchard chain value pool balance" are to be interpreted as defined in
ZIP 209. [^zip-0209]

The character § is used when referring to sections of the Zcash Protocol
Specification. [^protocol]

The terms below are specific to the Tachyon shielded protocol [^tachyon-protocol]
and are to be interpreted as follows.

`valueBalanceTachyon`

: The signed net value, in zatoshis, transferred between the Tachyon shielded
  value pool and the transparent value pool by a transaction, as defined by the
  Tachyon shielded protocol. [^tachyon-protocol] A positive value flows out of
  the Tachyon pool; a negative value flows in. This mirrors the role of
  `valueBalanceSapling` and `valueBalanceOrchard` for their pools.

Tachyon network upgrade

: The network upgrade that activates the Tachyon shielded protocol.
  [^tachyon-protocol]


# Abstract

ZIP 209 [^zip-0209] defines a consensus rule — the "turnstile" — that tracks the
running balance of each shielded value pool across the block chain and rejects
any block that would drive a pool's balance negative. It currently covers the
Sprout, Sapling, and Orchard pools.

This proposal extends that rule to the Tachyon shielded pool introduced by the
Tachyon shielded protocol [^tachyon-protocol]. It defines the "Tachyon chain
value pool balance" as the negation of the sum of all `valueBalanceTachyon`
fields across the block chain, and adds it to the set of balances that MUST NOT
become negative. The change is additive against ZIP 209 and mirrors how the
Orchard pool was added at NU5. It operates only on the public `valueBalanceTachyon`
field and therefore has no privacy implications.


# Motivation

ZIP 209 lets nodes monitor the total value shielded into, and unshielded out of,
each pool. If more value is unshielded from a pool than was ever shielded into
it, a balance violation has occurred — evidence of a bug or exploit in the
corresponding shielded protocol. ZIP 209 makes the network reject such blocks
outright, rather than relying on after-the-fact chain rollbacks.

The Tachyon shielded protocol [^tachyon-protocol] introduces a new shielded pool
whose transactions carry a signed `valueBalanceTachyon` field. Without extending
ZIP 209, a balance violation in the Tachyon pool would not be caught by
consensus, defeating the purpose of the turnstile for the new pool.


# Privacy Implications

The turnstile operates solely on the `valueBalanceTachyon` field, which is a
public field of every Tachyon-bearing transaction. This proposal adds no new
field and reveals no information that is not already public. It therefore has no
privacy implications beyond those already present in ZIP 209.


# Requirements

The extension to ZIP 209 must:

- define a Tachyon chain value pool balance using the same sign convention as the
  existing Sapling and Orchard pool balances, so that a net outflow exceeding the
  net inflow makes the balance negative;
- cause nodes to reject any block that would drive the Tachyon chain value pool
  balance negative, exactly as for the existing pools; and
- be an additive, editorial change to ZIP 209 that leaves the existing pool
  definitions and the rest of the consensus rule unchanged.


# Specification

## Changes to ZIP 209

The following changes are to be applied to ZIP 209 [^zip-0209]. They are
additive: no existing pool definition or clause is modified or removed.

In the Terminology section, alongside the existing chain value pool balance
definitions, add:

> The "Tachyon chain value pool balance" for a given block chain is the negation
> of the sum of all `valueBalanceTachyon` fields for transactions in the block
> chain. (Before the Tachyon network upgrade has activated, the Tachyon chain
> value pool balance is zero.)

In the Specification section, amend the consensus rule to read (addition in
**bold**):

> If any of the "Sprout chain value pool balance", "Sapling chain value pool
> balance", "Orchard chain value pool balance", or **"Tachyon chain value pool
> balance"** would become negative in the block chain created as a result of
> accepting a block, then all nodes MUST reject the block as invalid.

The remaining clause of ZIP 209 is unchanged: nodes MAY relay transactions even
if one or more of them cannot be mined due to this restriction.

## Folding into ZIP 209

This proposal is intended to be folded into ZIP 209's text by the ZIP Editors,
following the precedent by which the Orchard pool was added to ZIP 209 for NU5,
rather than deployed as a standalone numbered ZIP. The normative content above is
expected to be incorporated as part of the Tachyon shielded protocol
specification [^tachyon-protocol], which carries a "Changes to ZIP 209" section.
ZIP 209 would then gain an `Updated-By` header referencing that specification
once it reaches a Released status.


# Rationale

The Tachyon pool follows the same pattern as the Sapling and Orchard pools: the
bundle carries a signed `valueBalanceTachyon` field representing the net value
flowing between the Tachyon pool and the transparent value pool. Negating the sum
of these fields across the block chain gives the total value held in the pool, so
the same negative-balance check applies without modification. Reusing the
existing sign convention keeps the four pools uniform and lets the existing
balance-tracking logic in consensus nodes be extended with a single additional
accumulator.

The turnstile prevents a bug in the Tachyon shielded protocol from inflating the
pool's balance beyond what was deposited: even if such a bug allowed a proof to
verify for an unbalanced transaction, the chain-level check would reject the
block before the excess value could be unshielded.


# Deployment

This consensus rule applies to the Tachyon chain value pool balance from the
activation of the Tachyon network upgrade [^tachyon-protocol], analogously to how
ZIP 209's application to the Orchard chain value pool balance was deployed from
NU5 activation [^zip-0252]. Before the Tachyon network upgrade has activated, the
Tachyon chain value pool balance is zero and the rule has no effect.


# Reference implementation

Consensus nodes already track each shielded pool's chain value pool balance and
reject blocks that would drive any of them negative. Supporting this proposal
requires adding a Tachyon accumulator that sums `valueBalanceTachyon` (negated)
and including it in the existing negative-balance check, mirroring the Orchard
pool. No other change to the balance-tracking logic is required.


# References

[^BCP14]: [Information on BCP 14 — "RFC 2119: Key words for use in RFCs to Indicate Requirement Levels" and "RFC 8174: Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words"](https://www.rfc-editor.org/info/bcp14)

[^protocol]: [Zcash Protocol Specification, Version 2025.6.3 [NU6.1] or later](protocol/protocol.pdf)

[^zip-0200]: [ZIP 200: Network Upgrade Mechanism](zip-0200.rst)

[^zip-0209]: [ZIP 209: Prohibit Negative Shielded Chain Value Pool Balances](zip-0209.rst)

[^zip-0252]: [ZIP 252: Deployment of the NU5 Network Upgrade](zip-0252.rst)

[^tachyon-protocol]: [Tachyon Shielded Protocol (tachyon-zcash/tachyon issue #103)](https://github.com/tachyon-zcash/tachyon/issues/103)
