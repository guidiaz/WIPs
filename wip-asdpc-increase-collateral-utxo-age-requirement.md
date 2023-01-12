<pre>
  WIP: WIP-ASDPC-INCREASE-COLLATERAL-UTXO-AGE-REQUIREMENT
  Layer: Consensus (hard fork)
  Title: Increase the age requirement for using transaction outputs as collateral 
  Authors: Adán SDPC <adan@witnet.foundation>
  Discussions-To: `#dev-lounge` channel on Witnet Community's Discord server
  Status: Draft
  Type: Standards Track
  Created: 2022-10-26
  License: BSD-2-Clause
</pre>


## Abstract

This proposes an increase to the age requirement for using transaction outputs as collateral. The new required age
will be 13440 protocol epochs, i.e. 1 week. This improvement aims at strengthening the protocol's security and
economics.

## Motivation and rationale

### Current status

_Collateralization_ in Witnet means that nodes that become eligible to take part in resolving a oracle request need to
lock a certain amount of Wit coins upon publishing their commitment transaction. If their committed values pass the
filters in the eventual _tally_ of the request, they get the collateralized coins back. If they do not pass the filters
(aka _liars_), they lose the collateralized coins.

To use a transaction output (_UTXO_) as collateral, it must have existed for more than 2,000 epochs (25 hours). We often
call this the _UTXO age requirement_.

Its current value was set in the Witnet testnet times (2019) and was never adjusted to the mainnet reality.

Collateralized coins are refunded to their owners in the _tally_ transaction, with a new UTXO age of 0 epochs.

The point of collateralization is twofold: (1) deterring bribery attacks and (2) sybil resistance.

Bribery attacks on oracles like Witnet, and the role of collateralization for deterring them, have been discussed at
length in [this Medium post][bribery].

Collateralization provide sybil resistance because it creates exclusion of resources. The same coins cannot be
collateralized on multiple identities at the same time, and you can only move the coins from one identity to another
just as fast as the minimum UTXO age requirement allows.

Thus, the amount of identities that an actor can operate in parallel is bounded by the amount of Wit coins they own,
divided by the amount of coins that they could need to collateralize in each of them, over a period equal to the UTXO
age requirement.

A minimum collateral requirement for each oracle request is currently enforced by the protocol, and set to 1 Wit.
Requesters can freely decide to increase that requirement for any specific request.

### Problem statement

Some [ballpark estimations][estimations] suggested that, at sub-cent values for a Wit coin, the balance requirement for
running a node is around 8,000 Wits. A [further analysis][analysis] with an [ARS simulator tool][ARS] yielded
even lower figures, ranging from 3,000 to 5,000 Wits per identity.

Those numbers do not look specially good for credible resistance to bribery and Sybil attacks. As Witnet starts to
secure more TVL on its dozens of supported chains, we need to improve the economics of the protocol, and grow the amount
of value that node operators put into securing the network. In a decentralized oracle protocol like Witnet, any
economic improvement is also a security improvement.

### Considered solutions

Arguably, we could advocate for raising the minimum collateral amount. In theory, if we make it 100 Wit instead of 1
Wit, we could increase the total staked amount of Wit tokens (our own TVL, let's say) by the same factor. However, as
it is on the requester to choose the right collateral requirement for their use case, increasing the minimum would
only hinder the flexibility of posting cheap requests, may we adopt a [collateral-to-reward ratio][WIP-0022].

The main point in increasing the minimum collateral would be to force requesters to adjust this parameter instead of
defaulting to the minimum. However, increasing any consensus constant denominated in Wit can always have unexpected
side-effects if the price of the Wit coin goes up significantly.

Increasing the UTXO age requirement however has zero compromises to oracle queries. This change would be transparent to
end users, and would not affect the cost of the queries may we adopt a [collateral-to-reward ratio][WIP-0022]. The only
impact of this change would be on node operators, who would be urged to deposit more Wits into their identities in
fear of missing a chance to witness because they lacked old-enough UTXOs.

An extension to the UTXO age requirement should help multiply the total collateralized value proportionally. Therefore,
a proposal like this that extends this parameter, clearly aims to grow the total collateralized value as much as
possible to mitigate the undercollateralization of the protocol. That is, the more we grow the share of the circulating
supply that is locked into collaterals, the more the security of the protocol.

### Forethoughts when increasing the UTXO age requirement

The main downside to a longer UTXO age requirement is that new identities will not be able to witness for a longer
initial period, that is, it would extend the bootstrapping time of new nodes. Hence why the new proposed value for this 
requirement is only 1 week, which is significantly greater than the current 25 hours, but still a reasonable wait for a
node operator.

Another consideration when increasing the UTXO age requirement is the potential disruption to the the oracle quality
of service of the network that could happen right after activation of the increased requirement. If not enough nodes
in the network happen to meet the new requirements at that time, a number of oracle requests may go unresolved and just
get an `InsufficientCommit` error in their tally transactions. Moreover, this situation can also happen at any later
time. Special countermeasures are specified in this proposal to guarantee a smooth transition and to protect from a
generalized _everyone-is-lacking-collateral-age_ situation in the system.

## Specification

### Increase of the UTXO age consensus constant

**(1)** The UTXO age consensus constant MUST be increased to `13_440` epochs. That is, for an unspent transaction
output to be accepted as a collateral input in a commitment transaction, it MUST exist for at least 1 week prior to
acceptance of such commitment transaction into a block in the chain.

### Adaptive UTXO age requirement

To address any _everyone-is-lacking-collateral-age_ situation, the following changes are introduced: 

**(2)** Instead of applying the UTXO age consensus constant directly as a fixed value for the UTXO age requirement, the
lesser of this constant and another `safe_utxo_age` computed value MUST be used as the requirement.

```rust
/* RUST-ALIKE PSEUDOCODE */
let utxo_is_old_enough = now - utxo.timestamp >= std::cmp::min(UTXO_AGE_REQUIREMENT, safe_utxo_age);
```

**(3)** A new `SAFE_UTXOS_COUNT` protocol-level consensus constant is introduced, and set to `2_000`.

**(4)** `safe_utxo_age` MUST be computed as the age of the freshest UTXO among the `SAFE_UTXOS_COUNT` oldest
collateralizable UTXOs (>1 Wit) that are owned by identities that have a non-zero reputation and are active (i.e.
reputed ARS members).

**(5)** A new `SAFE_UTXOS_RETARGET_PERIOD` protocol-level consensus constant is introduced, and set to `2_400`.

**(6)** `safe_utxo_age` MUST be updated only at the beginning of epochs that are multiples of
`SAFE_UTXOS_RETARGET_PERIOD` (i.e. every 30 minutes). It MUST NOT be updated at any other time.

```rust
/* RUST-ALIKE PSEUDOCODE */
if current_epoch % 2_400 == 0 {
  let safe_utxo_age = current_epoch - ars
      .iter()
      .filter_map(|identity| if identity.reputation > 0 {
        Some(identity.utxos.filter(|utxo| utxo.value > 1_000_000_000))
      } else {
        None
      })
      .flatten()
      .take(SAFE_UTXOS_COUNT)
      .max_by_key(|utxo| utxo.epoch)
      .epoch;
}
```

## Backwards compatibility


### Consensus cheat sheet

Upon entry into force of the proposed improvements:

- Blocks and transactions that were considered valid by former versions of the protocol MUST continue to be considered valid.
- Blocks and transactions produced by non-implementers MUST be validated by implementers using the same rules as with those coming from implementers, that is, the new rules.
- Implementers MUST apply the proposed logic when evaluating transactions or computing their own data request eligibility.

As a result:

- Non-implementers MAY NOT get their blocks and transactions accepted by implementers.
- Implementers MAY NOT get their valid blocks accepted by non-implementers.
- Non-implementers MAY consolidate different blocks and superblocks than implementers.

Due to this last point, this MUST be considered a consensus-critical protocol improvement. An adoption plan MUST be proposed, setting an activation date that gives miners enough time in advance to upgrade their existing clients.

### Libraries, clients, and interfaces

The protocol improvements proposed herein will affect any library or client implementing the core Witnet block chain
 protocol, especially the commitment transaction validation rules.

Libraries, clients, interfaces, APIs, or any other software or hardware that do not implement or apply
commitment transactions and block validation will remain unaffected.

## Reference Implementation

This improvement proposal has no reference implementation yet.

## Adoption Plan

An activation date and [TAPI] signaling bit will be opportunely proposed upon moving this draft to the Proposed stage.

## Acknowledgements

This proposal has been cooperatively discussed and devised by many individuals from the Witnet development community.

[bribery]: https://medium.com/witnet/deterring-bribery-attacks-on-decentralized-oracle-networks-5bcf87d2cb22
[WIP-0022]: wip-0022.md
[estimations]: https://github.com/witnet/witnet-rust/discussions/2237#discussion-4258457
[analysis]: https://github.com/witnet/witnet-rust/discussions/2237#discussioncomment-3414051
[ARS]: https://github.com/drcpu-github/ARS-simulator
[TAPI]: wip-0014.md