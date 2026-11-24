---
title: Staking Receiver Keys
description: Add validator receiver-key enrollment and expose the selected validator set and its keys for distributed key generation.
author: Category Labs (@category-labs)
discussions-to: https://forum.monad.xyz/
status: Draft
type: Standards Track
category: Core
created: 2026-11-23
---

## Abstract

This MIP lets a validator enroll an independent secp256k1 receiver key with Monad's native staking precompile. The precompile verifies that the validator controls the key and records it for use across epochs. Once enforcement begins, a validator needs an enrolled receiver key to qualify for the next validator set.

## Motivation

During distributed key generation (DKG), validators encrypt private shares for specific recipients. Each validator needs a receiver key distinct from its consensus secp256k1 signing key.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174 when, and only when, they appear in all capitals.

https://www.rfc-editor.org/rfc/rfc2119

https://www.rfc-editor.org/rfc/rfc8174

The staking precompile is at `0x0000000000000000000000000000000000001000`. Calls to a disabled selector follow the staking precompile's unsupported-method fallback. Each enabled selector has a fixed native gas charge. Transaction intrinsic and calldata charges apply separately.

The enrollment revision, `MONAD_NEXT_BVE_ENROLLMENT`, enables the three precompile methods below. The enforcement revision, `MONAD_NEXT_DKG`, MUST activate later to give existing validators time to enroll a receiver key.

### Add a receiver-key `addValidator` overload

The enrollment revision enables `addValidator(bytes,bytes,bytes,(bytes,bytes32,bytes32))` (`0x3e4616bf`). The overload takes the registration message, the 64-byte consensus secp256k1 signature, the 96-byte consensus BLS signature, and the receiver proof.

The original `addValidator(bytes,bytes,bytes)` selector (`0xf145204c`) can be disabled at either `MONAD_NEXT_BVE_ENROLLMENT` or `MONAD_NEXT_DKG`. This specification disables it at `MONAD_NEXT_BVE_ENROLLMENT` so new validators enroll their receiver key in the registration transaction. Only validators registered before enrollment need a separate `addReceiverKey` transaction.

Validators MUST generate their DKG receiver key independently of their consensus signing keys. The 198-byte registration message inserts the 33-byte compressed receiver key after the BLS key.

```text
consensusSecpKey[33] || consensusBlsKey[48] || receiverKey[33] ||
authAddress[20] || stake[32 big-endian] || commission[32 big-endian]
```

Both consensus signatures MUST cover the entire message using the existing validator-registration verification rules. Apply the existing stake, value, commission, and consensus-key checks. Verify the receiver proof and require an empty receiver-key ownership slot. On success, create the validator, make the initial delegation, and write the receiver key and its ownership entry atomically. Emit the existing validator-creation event and return the new `uint64` validator ID.

#### Proof of possession

Both write methods use the staking precompile's positional ABI decoding. The argument heads occupy their usual ABI words and the tails follow in argument order. The proof tuple contains a dynamic 33-byte image and two static 32-byte scalars. All declared byte-string lengths MUST match and the calldata MUST end after the padded tails.

Let `G` be the secp256k1 generator and `n` its order. `P` is the receiver public key. Let `validatorAddress` be the Ethereum address derived from the validator's **consensus** secp256k1 key and let `authAddress` be its staking authorization address. For creation these come from the signed registration message. For enrollment they come from the stored validator record.

Compute `Q` using RFC 9380 `secp256k1_XMD:SHA-256_SSWU_RO_` over the following bytes.

```text
chainId[32 big-endian] || validatorAddress[20] || authAddress[20] || P[33]
```

Use the chain ID of the executing network. The domain separation tags are exact byte strings. The first byte is `0x13` and the last byte is `0x0a` in both tags.

```text
base DST      = 0x13 || ASCII("btx-dkg/pop-base/1") || 0x0a
challenge DST = 0x13 || ASCII("btx-dkg/pop-chal/1") || 0x0a
```

Read the proof as `(I, c, z)`. Both `P` and `I` MUST be valid, non-identity, 33-byte compressed SEC 1 points. `c` and `z` MUST be canonical 32-byte big-endian scalars in `[0, n)`. Compute the commitments as follows.

```text
A = z G - c P
B = z Q - c I
```

Reject a proof when either commitment is the identity. Compute the expected challenge with RFC 9380 `hash_to_field` over the scalar field of order `n`, with count 1, extension degree 1, `L = 48`, `expand_message_xmd` using SHA-256 and the challenge DST. The message is the concatenation of the compressed encodings of `Q`, `P`, `I`, `A` and `B`, in that order. The proof MUST be accepted only if this challenge equals `c`.

A prover who knows the receiver secret scalar `x` constructs `P = x G` and `I = x Q`. With a fresh nonzero scalar `r`, it computes `A = r G` and `B = r Q`, hashes the commitments as above, and returns `z = (r + c x) mod n` with `I` and `c`.

https://www.rfc-editor.org/rfc/rfc9380

#### Receiver-key storage

In validator-metadata namespace `0x09`, write the receiver key at word offsets 8 and 9, immediately after the unclaimed-rewards field. Pad the second word with 31 zero bytes. Two zero words represent an unset key. Existing fields retain their offsets.

The ownership slot key consists of namespace byte `0x0d`, the 20-byte Ethereum address derived from `P`, and 11 zero bytes. The slot value is the owning validator ID, or zero if it has never been assigned. An enrollment MUST NOT overwrite a receiver key or a receiver ownership slot. Receiver ownership is separate from consensus-key ownership.

#### Gas

The legacy charge is 505,125 gas. It includes 352,125 gas for storage and events, a 3,000-gas allowance for consensus secp256k1 verification, and a 150,000-gas allowance for BLS verification. The overload charges **587,925 gas**.

| Work | Gas |
| --- | ---: |
| Existing validator creation | 505,125 |
| One cold read of the receiver-key ownership slot | 8,100 |
| One warm write to the ownership slot | 2,900 |
| Two cold writes for the receiver key | 21,800 |
| Receiver proof verification | 50,000 |
| **Total** | **587,925** |

On the same release build and hardware, a valid proof took 0.215 ms to verify and native `ecrecover` took 0.040 ms at a charge of 3,000 gas. The 50,000-gas proof charge leaves headroom above the estimate from that runtime comparison.

Standard ABI encoding takes 804 calldata bytes including the selector, compared with 548 bytes for the legacy call.

### Add `addReceiverKey`

From the enrollment revision, `addReceiverKey(uint64,bytes,(bytes,bytes32,bytes32))` (`0x3a3b9a22`) enrolls a key for an existing validator. The arguments are the validator ID, its 33-byte compressed receiver key, and the proof of possession.

The caller MUST be the validator's staking authorization address and the call MUST carry zero value. Require an existing validator with an empty receiver-key field and an empty ownership slot for the submitted key. Verify the proof using the stored consensus secp256k1 key and authorization address. On success, write the key and its ownership entry, return `true`, and emit `ReceiverKeyAdded(uint64 indexed validatorId)`. Callers can read the key with `getReceiverKey`. Eligibility is evaluated at validator selection.

#### Gas

The precompile charges **119,575 gas** using the existing staking storage and event prices and the same proof-verification budget as validator creation.

| Work | Calculation | Gas |
| --- | ---: | ---: |
| Read the authorization record, key presence word, three packed consensus-key slots, and ownership slot | 6 × 8,100 | 48,600 |
| Write the first key word and ownership slot after reading them | 2 × 2,900 | 5,800 |
| Write the second key word | 1 × 10,900 | 10,900 |
| Emit `ReceiverKeyAdded` | 1 × 4,275 | 4,275 |
| Verify the receiver proof | 1 × 50,000 | 50,000 |
| **Total** | | **119,575** |

Standard ABI encoding takes 388 calldata bytes including the selector.

### Add `getReceiverKey`

From the enrollment revision, `getReceiverKey(uint64)` (`0x83afcd10`) reads the receiver key for the supplied validator ID. The call MUST carry zero value and follow the staking precompile's existing call restrictions. Return ABI-encoded `bytes` containing the 33-byte compressed key. An unset key or unknown validator ID returns `bytes` containing 33 zero bytes. Use the existing ABI and 97,200-gas charge for `getValidator(uint64)`.

#### Gas

The precompile charges **16,200 gas**, budgeting two cold storage reads at 8,100 gas each for the receiver-key words. The same two-word read applies to an unset key. Standard ABI encoding takes 36 calldata bytes including the selector.

### Require receiver keys at validator selection

At the first `SNAPSHOT` under `MONAD_NEXT_DKG` and every subsequent `SNAPSHOT`, apply the existing validator status checks to each candidate. For a candidate that passes those checks, read the first receiver-key word and include the candidate only if its first byte is nonzero. Rank eligible candidates by descending stake, break ties by ascending validator ID, and apply the existing active-set limit.

A validator excluded solely for a missing receiver key stays in the execution set and can qualify at a later snapshot. Enrollment after a snapshot cannot add a validator to the already selected set. During the epoch delay, the snapshot view supplies the current set and the consensus view supplies the selected next set.

#### Gas

`SNAPSHOT` is a protocol system transaction with a zero gas limit and a zero-gas receipt. The eligibility check adds one receiver-key word read for each candidate that passes the existing status checks. A keyless candidate is skipped before its stake is read.

#### Finalized validator-set reads

The native finalized validator-set reader MUST return the membership and order of the requested epoch at the supplied finalized block. Each entry includes the consensus secp256k1 key, BLS key, receiver key if set and the stake of that validator in the selected view. The reader MUST use the same execution state for membership, stake and key data. Historical validator sets may contain entries with an unset receiver key.

## Rationale

Consensus signatures authorize validator registration. DKG recovery also requires proof that the registrant knows the secret scalar for the submitted receiver key. A validator can construct a public key related to another validator's key using public information alone. Because recovery shares ephemeral points across receivers, enrolling such a key could let the registrant use recovery openings to decrypt shares addressed to the other validator. The Chaum–Pedersen proof prevents this rogue-key registration by proving knowledge of the receiver secret. A receiver-key signature would need a separate security argument that establishes the same knowledge property for adversarially chosen keys.

Binding the proof to the chain ID, consensus identity, and authorization address prevents replay in a different registration context. A long-lived receiver key lets validators enroll once and use the same key across epochs.

## Backwards Compatibility

Disabling the original selector is a breaking ABI change. Calls to it revert starting at `MONAD_NEXT_BVE_ENROLLMENT`. Registration clients must switch to the overload, include the receiver key in the message signed by both consensus keys, and supply the receiver proof.

Validators registered before enrollment keep their existing IDs, stake and consensus keys. They can enroll with `addReceiverKey` and must do so before a snapshot under enforcement to be eligible for selection at that snapshot.

Finalized reads of pre-enforcement state may yield an absent receiver key. Consensus clients map this absence to an optional field, so a validator file or event encoded before enrollment can still be read. A validator record with a receiver key appends it to the prior RLP structure. Every consumer of keyed records must upgrade before receiving them.

## Test Cases

### Registration proof vector

The registration vector uses chain ID `1`, validator address `0x0000000000000000000000000000000000001234` and authorization address `0x0000000000000000000000000000000000004321`. The receiver secret scalar is `7` and the proof nonce is `11`. The calculated proof must verify.

```text
Q = 027327d0e06447e24ac9dfb8e4abd0cbd194b86cbcbd6a83ddcbdab76f86d80576
I = 03fe439a77eda1d6287267a6c256f2423dff626259ce7001976426ed3dae6701d8
c = 2d545bb7cffd24a37e384079d041a900faa82a1e2e006f775d75af3ab1cd561e
z = 3d4e8206afec00787389c354b1cb9f081fea49ec92ba6c07ce656c0e0c67199c
```

### Staking behavior

| Scenario | Expected result |
| --- | --- |
| Verify the vector with chain ID `2`, a different validator address, a different authorization address, or a different public key | Reject each proof |
| Submit a proof with an invalid compressed point, a scalar equal to `n`, or commitments at infinity | Reject the proof |
| Call the four-argument creation selector before enrollment or the three-argument selector from enrollment | Unsupported method |
| Enroll as a caller other than the authorization address | Revert without writes |
| Try to replace an enrolled key, or enroll an already-owned key for a second validator | Revert without writes |
| Read an unknown or keyless validator with `getReceiverKey` | Return 33 zero bytes as ABI `bytes` |
| Enroll an eligible-by-stake validator after an enforcing snapshot that excluded it | No membership change for the selected epoch. Include it as a candidate at the next snapshot. |
| Read the same finalized block and epoch from two nodes | Identical member order, keys and stake |

## Security Considerations

The verifier rejects scalar aliases, points outside the compressed secp256k1 encoding and identity commitments. The chain ID and both addresses are part of the proof context. Proof generators must use a fresh nonce for each proof and keep the receiver scalar and proof nonce private.

The receiver key is immutable. Operators must preserve its secret for as long as they use that validator identity. Losing or compromising it affects receipt of encrypted shares. Recovery or key rotation would require a separate protocol change.

Enforcing eligibility before enough validators enroll can reduce the active set and exclude substantial stake. Network rollout should verify enrollment coverage and schedule enforcement before activating the DKG consumer. Proof verification uses fixed-size inputs and the getter reads two storage slots.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
