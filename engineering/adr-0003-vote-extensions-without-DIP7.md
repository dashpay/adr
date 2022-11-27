# Remove DIP7 format support from Tenderdash vote extensions

* Date: 2022-11-10
* Participants: @shumkov, @jawid-h, @knst, @PastaPastaPasta, @QuantumExplorer, @shotonoff, @lklimek
* Scope(s): Core, Platform, Tenderdash

## Status

Proposed

## Context

The initial plan to sign Asset Unlock (withdrawal) transactions with tenderdash validator
set quorum assumed that signature payload will follow [DIP7](https://github.com/dashpay/dips/blob/master/dip-0007.md) format:
```
requestID = SHA256("plwdtx", index)

sigHash = SHA256(transaction without quorumSig)

SignID = SHA256(quorumHash, requestID, sigHash)
```

Asset Unlock `index` and `quorumHash` are part of the transaction payload, so any consumer can easily
verify this signature.

Later, when new tenderdash functionality "vote extensions" were discovered, the decision to utilize vote extension
instead of custom signing logic was made between the group. The potential "multi round signature shares"
attack vector was also brought up during the discussion. Basically, if signature payload doesn't contain round specific
information and payloads can be different between rounds, an attacker can collect shares from different (faulty) rounds
and recover valid signature. As a solution, @lklimek and @shotonoff incorporated a protection mechanism.
They added [platform block height and round](https://github.com/dashpay/tenderdash/blob/dca73910c74bb8b80605e66b4a4b3a9c36c02e80/types/vote.go#L464) to the `requestID`:

```
requestID = SHA256("dpbvote", height, round)

sigHash = SHA256(transaction without quorumSig)

SignID = SHA256(quorumHash, requestID, sigHash)
```

Although this approach protects against the attack, there is no way for consumers to verify this signature
since they don't have platform height and round.

On other hand, Drive implements a special two block commit approach to construct and sign asset unlock transactions in safe way.
With this solution the transactions aren't signed immediately but incomplete form of them (without block specific information in payload)
go to special queue and will be pulled for signing on the next block. This guarantee on code level that for specific height only those
exact transactions will be used as vote extensions for every round. There are three types of asset unlock transactions, additional queue
in platform state and logic in Drive implemented to support this solution.

There two ways to proceed with credits withdrawal implementation:
1. Add height and round to asset lock transaction payload and remove unnecessary in this case "two block commit" solution to simplify the system
2. Change tenderdash to sign the whole transaction instead of DIP7 format.

Both approaches have benefits and downsides. In this ADR we will look closer to the second solution.
There is [ADR0002](./adr-0002-height-and-round-in-unlock-tx-payload.md) that is dedicated to the first approach. 

## Decision

To simplify Asset Unlock transaction signature verification for all consumers, Tenderdash should sign the whole transaction (without `quorumSig`)
with `Sign` Core RPC method instead of DIP7 format. This will also allow Core to validate the transaction during the signing.

To verify the signature, asset unlock transaction consumer should just use the whole transaction without `quorumSig` as signature message:

```
SignID = transaction without quorumSig
```

## Implementation

* Change vote extension signing logic to pass the whole vote extension payload instead of DIP7 format (Tenderdash team)
* Pass the whole transaction to vote extensions instead of transaction hash (Platform team)
* Update Asset Unlock Transaction signature verification function in rust-dashcore (Platform team)
* Update Asset Unlock Transaction signature verification function in Core (Core team)
* Add a new RPC method to validate and sign Asset Unlock Transaction (Core team)
* VoteExtension should use special Core RPC method in case if vote extended with Asset Unlock Transaction (Tenderdash team)

## Consequences

### Pros

* Very simple rules on how to sign and verify signatures.
* Additional validation for Asset Unlock Transactions. Technically, such validation already performed by miners. Also, it won't
  prevent an attacker to sign arbitrary data with Core because we still sign tenderdash blocks with `Sign` RPC method. But it adds an extra
  security on top of the existing functionality.

### Cons

* Tenderdash's vote extension functionality become vulnerable for "multi round signature shares" attack. ABCI app developers
  should always keep this in mind and implement own protection mechanisms for each case on ABCI app side. 
* It introduces an additional dependency between Tenderdash and Core
* It requires Tenderdash to rely on another component to ensure the solution's security. 
* Vote extension implementation becomes part of business logic that "belongs" to a higher layer, instead of just providing well-defined,
  independent services to that layer.
* With withdrawal pooling mechanism, asset unlock transaction could be pretty big and will be interchanged between tenderdash
  validators via p2p according to the current implementation. This can be optimized but additional changes in Tenderdash is required.
* Tenderdash should know what it's signing with vote extension and use special Core RPC method. This logic can be moved to Core, then
  Core should detect what it's signing in `Sign` method.
* Bigger amount of work comparing with [ADR0002](./adr-0002-height-and-round-in-unlock-tx-payload.md)
