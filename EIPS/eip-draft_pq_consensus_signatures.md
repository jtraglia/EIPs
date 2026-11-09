---
title: Post-Quantum Consensus Signatures
description: Replace BLS with XMSS
author: Justin Traglia (@jtraglia)
discussions-to: https://ethereum-magicians.org/t/TODO
status: Draft
type: Standards Track
category: Core
created: 2026-11-07
requires: 7732, 8205, 8483
---

## Abstract

This EIP replaces BLS with XMSS for consensus signatures while leaving the consensus mechanism unchanged. XMSS is a hash-based signature scheme that is believed to be post-quantum secure. Because XMSS has no native aggregation, STARK proofs replace aggregate signatures. Before the fork, validators and builders register XMSS keys through [EIP-8483](./eip-8483.md). At the fork, those that registered switch to their XMSS keys, and those that did not are exited.

## Motivation

The consensus layer relies on BLS signatures, which a sufficiently capable quantum computer could forge. Such an adversary could sign messages on behalf of any validator. The protocol must switch to a post-quantum signature scheme before there is a credible threat. This EIP defines the minimal set of changes required to switch to XMSS signatures.

## Specification

### Signatures

XMSS public keys replace BLS public keys in validator and builder structures, and XMSS signatures replace BLS signatures in signed messages. Each message is signed at a fixed leaf (one-time key pair). Most messages use a leaf derived from their slot and type, attestations use a leaf derived from their target epoch, and messages that a key signs at most once, such as voluntary exits, use reserved leaves. Each signature includes the index of the leaf that produced it. Verifiers derive the expected leaf from the message and reject any signature with a different leaf index. Aggregate attestations no longer carry a signature. Instead, aggregators attach a STARK proof to each aggregate they publish, and the proposer merges these proofs into a single proof covering every attestation in its block.

### Fork Transition

At the fork, validators that registered through [EIP-8483](./eip-8483.md) adopt their registered XMSS public key and VRF root, while builders that registered adopt only their XMSS public key. Validators and builders that did not register are exited.

### Randomness

A VRF root, the root of a Merkle tree of secret values with one per slot, is added to the validator structure. RANDAO reveals and aggregator selection proofs are no longer BLS signatures. Instead, each consists of the validator's secret value for the slot and a Merkle branch proving that value against the validator's VRF root.

### Execution Requests

In execution requests, the 48-byte public key field holds 16 zero bytes followed by the 32-byte XMSS public key. Deposits are no longer signed, and the first 32 bytes of a deposit's signature field instead hold the VRF root of the validator it creates. Requests that still contain a BLS public key, including those submitted before the fork, are legacy requests. A legacy deposit creates a validator or builder that is immediately exited, so its funds are returned after the usual delay. Legacy withdrawal and consolidation requests are ignored.

### Removed Operations

BLS-signed withdrawal credential changes are no longer accepted, and blocks no longer carry payload attestations.

## Rationale

### XMSS

Hash-based signatures depend only on the security of hash functions, among the most conservative cryptographic assumptions. This EIP uses leanXMSS, a variant of XMSS designed for Ethereum consensus, with small public keys, and with signatures that are efficient to prove in a STARK.

XMSS is built from one-time signatures. A key consists of many one-time key pairs, called leaves, and the XMSS public key is the root of a Merkle tree over their public keys. A signature is a one-time signature from a single leaf, plus the Merkle path from that leaf to the root. To verify it, the verifier recomputes the leaf's public key from the one-time signature and checks that the path leads to the root. XMSS differs from BLS in a few key ways.

- XMSS signatures are much larger, which increases the size of every signed message. Public keys are slightly smaller.
- XMSS is stateful. Each key has about 4 billion leaves, and signing two different messages at the same leaf compromises that leaf. Because each signed message has a fixed leaf, an honest signer never reuses a leaf and needs no signing state beyond what slashing protection already records. Under this assignment, a single key lasts many decades without rotation.
- XMSS does not support native aggregation. STARK proofs take the place of aggregate signatures.
- XMSS signatures are not unique, so a proposer could grind a signature-based RANDAO reveal. Instead, each validator commits to a VRF root, which is the root of a Merkle tree of secret values, one per slot. Because the tree is fixed in advance, a proposer cannot grind its reveal and, as with BLS, can only choose whether to publish its block. Builders do not need a VRF root, since they neither propose blocks nor aggregate attestations.

### Contingency Plan

Replacing BLS with XMSS without changing the consensus mechanism would address the most critical vulnerability quickly. Even if this approach proves feasible but unattractive for other reasons, it can still serve as a contingency plan, ready to ship in an emergency while work on a new consensus mechanism continues. Evaluating its feasibility now avoids a situation where a new consensus mechanism must be shipped before it has been properly tested. If a new consensus mechanism turns out to be required to accommodate larger signatures, the work here is not wasted.

### Unsigned Deposits

A BLS deposit signature is a proof of possession, which prevents rogue-key attacks on aggregate signatures. A STARK proof verifies each signature against its own key, so this attack does not apply and no proof of possession is needed. Without signatures, anyone who observes the first deposit for a new key could front-run it with a deposit to their own withdrawal credentials. [EIP-8205](./eip-8205.md) preregistrations prevent this. Keeping the existing request formats avoids changes to the existing execution layer contracts.

### Removed Operations

Blocks no longer carry payload attestations because, without native aggregation, proposers would have to aggregate them with another STARK proof. Fork choice still receives these votes through gossip.

BLS-signed withdrawal credential changes are disabled because, once BLS can be forged, an attacker could use one to redirect a validator's withdrawals to an address it controls. Validators with `0x00` withdrawal credentials must therefore set a withdrawal address before the fork.

### Unregistered Keys

Validators and builders without a registered key are exited because the protocol has no key it can trust for them. Keeping their BLS keys would let a quantum adversary act on their behalf, and keeping them active without a usable key would only lead to missed duties. Exiting them returns their funds to their owners, except for `0x00` validators, whose funds remain locked because they have not set a withdrawal address.

### Sync Committees

Sync committees remain an open question. Without native aggregation, the 512 sync committee signatures per slot would add about 1.2 MB to each block, or require a second STARK proof competing with attestations for prover capacity, solely to serve light clients. The goal is to make sync committees practical with XMSS. If that is not possible, sync committees may need to be deprecated, and light clients would instead rely on an alternative such as [EIP-8025](./eip-8025.md) optional proofs.

### Scope

This EIP covers consensus signatures only. Execution layer transaction signatures, the keys that control withdrawal addresses, KZG blob commitments, and peer-to-peer networking keys still rely on elliptic curves and require separate upgrades. In particular, [EIP-8483](./eip-8483.md) registrations are authorized by the withdrawal address, so they must be made while its key can still be trusted.

## Backwards Compatibility

This EIP changes consensus rules and requires a hard fork. It fully replaces BLS with XMSS in the consensus layer, so any software that produces, parses, or verifies consensus signatures must be updated. Validators and builders that have not registered with [EIP-8483](./eip-8483.md) are exited at the fork.

## Security Considerations

### Leaf Reuse

Reusing a leaf for two different messages lets anyone forge signatures at that leaf. Running a key on multiple machines, or restoring a key without its slashing protection history, is therefore more dangerous than with BLS. Some protocol behavior also becomes a signing constraint. For example, a builder cannot raise a bid it has already broadcast or bid on two different parents in the same slot, and a proposer cannot sign preferences for the same slot under two different dependent roots.

### Mass Exits

All unregistered stake exits at the fork. If registration is low, the active validator set shrinks significantly, reducing the stake that secures finality and lowering the cost of attacks. Registration levels should be checked before the fork is scheduled. In addition, proposals and payload timeliness committee votes already assigned to unregistered validators in the first epochs after the fork are missed.

### Denial of Service

With XMSS, most messages are larger, which makes the peer-to-peer network more vulnerable to denial-of-service attacks. Every invalid message an attacker sends costs peers far more bandwidth than it would with BLS.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
