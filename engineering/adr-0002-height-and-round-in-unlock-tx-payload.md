# Add platform block round and height to asset unlock transaction payload

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

Both approaches have benefits and downsides. In this ADR we will look closer to the fist solution.
There is [ADR0003](./adr-0003-vote-extensions-without-DIP7.md) that is dedicated to the second approach.

## Decision

To allow asset unlock transaction consumers verify signature, concatenated prefix `ve` (Dash Platform Vote Extension), platform height (uint64)
and platform round (uint32) as `signatureRequest` (14 bytes) field should be added to asset unlock transaction payload. Having a complete request
message as part of the transaction payload will allow not just tenderdash but other L2 systems to utilize Asset Lock/Unlock Core functionality for
their needs in the future.

To verify the signature, asset unlock transaction consumer should hash `signatureRequest` from payload a combine with `quorumHash` and `sigHash`
according to DIP7:

```
requestID = SHA256(signatureRequest from transaction payload)

sigHash = SHA256(transaction without quorumSig)

SignID = SHA256(quorumHash, requestID, sigHash)
```

The two block commit logic that mitigates "multi round signature shares" attack using additional step with a queue become unnecessary and can be removed from the codebase to simplify the withdrawal process.

An additional security layer could be added to [Sign](https://dashcore.readme.io/docs/core-api-ref-remote-procedure-calls-evo#quorum-sign) Core's RPC.
List of quorums types should be limited to platform only. It will guarantiee that Sign RPC method won't be used to attack Layer 1 functionality like
Instant Send and Chain Lock. 


## Implementation

* Revert commit with additional asset lock payload types form rust-dashcore library (Platform team)
* Remove withdrawal transactions queue and related logic from Drive (Platform team)
* Add `requestID` to Asset Unlock Payload and update signature verification function in rust-dashcore (Platform team)
* Add `requestID` to Asset Unlock Payload and update signature verification function in Core (Core team)
* Change request ID prefix to `ve` in Tenderdash (Tenderdash team)

## Consequences

### Pros

* An Asset Unlock transaction will be signed and broadcasted to Core on the same block when the withdrawal was requested.
  It speeds up withdrawal up to 3 minutes (empty block interval).
* Simpler withdrawal functionality. Unnecessary logic, data and IO operations will be removed that will simplify
  the system and reduce the load.

### Cons

* Additional 14 bytes will be added to Asset Unlock transaction
