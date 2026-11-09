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

This EIP replaces BLS with XMSS for consensus signatures while leaving the consensus mechanism unchanged. XMSS is a hash-based signature scheme that is believed to be post-quantum secure. Before the fork, validators and builders register XMSS keys through [EIP-8483](./eip-8483.md). At the fork, those that registered switch to their XMSS keys, and those that did not are exited.

## Motivation

The consensus layer relies on BLS signatures, which a sufficiently capable quantum computer could forge. Such an adversary could sign messages on behalf of any validator. The protocol must switch to a post-quantum signature scheme before there is a credible threat. This EIP defines the minimal set of changes required to switch to XMSS signatures. A broader post-quantum redesign of consensus can follow separately. Limiting the scope allows the core vulnerability to be addressed sooner, independent of that larger effort.

## Specification

### Signatures

Validators and builders sign with leanXMSS, a variant of XMSS designed for Ethereum consensus, instead of BLS. XMSS is a hash-based scheme built from one-time signatures. A key consists of many one-time key pairs, called leaves, and the XMSS public key is the root of a Merkle tree over their public keys. A signature is a one-time signature from a single leaf, plus the Merkle path from that leaf to the root. To verify it, the verifier recomputes the leaf's public key from the one-time signature and checks that the path leads to the root. XMSS differs from BLS in a few key ways.

- XMSS signatures are much larger. A signature is 2404 bytes, compared to 96 bytes for BLS, which increases the size of every signed message. Public keys are 32 bytes, slightly smaller than the 48 bytes of a BLS public key.
- XMSS is stateful. Each key has about 4 billion leaves, and signing two different messages with the same leaf compromises that leaf. To prevent reuse, the protocol assigns a fixed leaf to each signed message, and verification checks that the signature uses it. Attestations use one leaf per epoch, other regular messages such as block proposals use one leaf per slot and message type, and one-time messages such as voluntary exits use reserved leaves. As a result, an honest validator never reuses a leaf, and a single key lasts many decades without rotation.
- XMSS does not support native aggregation. Instead of an aggregate signature, an aggregator attaches a STARK proof of the signatures it collected, and the proposer merges these proofs into a single proof covering every attestation in its block.

### Randomness

Because XMSS signatures are not unique, a proposer could grind a signature-based RANDAO reveal. Instead, each validator commits to a VRF root, which is the root of a Merkle tree of secret per-slot values. A validator's RANDAO reveal for a slot is that slot's leaf and its Merkle branch, and the same value serves as its aggregator selection proof. Because the tree is fixed at registration, a proposer cannot grind its reveal and, as with BLS, can only choose whether to publish its block.

### Execution Requests

Execution layer request formats are unchanged, so existing contracts continue to work. The 48-byte public key field of a request holds 16 zero bytes followed by the 32-byte XMSS public key. Deposits are no longer signed; instead, the first 32 bytes of the signature field hold the new validator's VRF root. Requests that still contain a BLS public key, such as those submitted before the fork or by outdated tooling, are treated as legacy requests. A legacy deposit creates a validator or builder that begins exiting immediately, so its funds are returned after the usual delay. Legacy withdrawal and consolidation requests are ignored.

### Preregistrations

This EIP incorporates [EIP-8205](./eip-8205.md) preregistrations, adapted to XMSS. Because deposits are unsigned, anyone who observes the first deposit for a new key could front-run it with a deposit to their own withdrawal credentials. To prevent this, a key holder can first submit a preregistration, signed with a reserved leaf, that binds the key to its withdrawal credentials and VRF root. While the binding is active, deposits for that key that do not match it are discarded.

### Sync Committees

Sync committees remain an open question. Without native aggregation, the 512 sync committee signatures per slot would add about 1.2 MB to each block, or require a second STARK proof competing with attestations for prover capacity, solely to serve light clients. The goal is to make sync committees practical with XMSS. If that is not possible, sync committees may need to be deprecated, and light clients would instead rely on an alternative such as [EIP-8025](./eip-8025.md) optional proofs.

## Rationale

### XMSS

Hash-based signatures depend only on the security of hash functions, among the most conservative cryptographic assumptions. The XMSS variant used here is designed for Ethereum consensus, with small public keys and signatures that are efficient to prove in a STARK. Its main drawback, statefulness, is addressed by deriving every leaf from a slot or epoch, so validators keep no signing state beyond what slashing protection already records.

### Contingency Plan

Replacing BLS with XMSS without changing the consensus mechanism would address the most critical vulnerability quickly. Even if this approach proves feasible but unattractive for other reasons, it can still serve as a contingency plan, ready to ship in an emergency while work on a new consensus mechanism continues. Evaluating its feasibility now avoids a situation where a new consensus mechanism must be shipped before it has been properly tested. If a new consensus mechanism turns out to be required to accommodate larger signatures, the work here is not wasted.

### Unregistered Keys

Validators and builders without a registered key are exited because the protocol has no key it can trust for them. Keeping their BLS keys would let a quantum adversary act on their behalf, and keeping them active without a usable key would only lead to missed duties. Exiting them returns their funds to their owners, except for `0x00` validators, whose funds remain locked because they have not set a withdrawal address.

### Unsigned Deposits

A BLS deposit signature is a proof of possession, which prevents rogue-key attacks on aggregate signatures. A STARK verifies each signature against its own key, so this attack does not apply and no proof of possession is needed. The remaining risk, deposit front-running, is addressed by [EIP-8205](./eip-8205.md) preregistrations. Keeping the existing request formats avoids changes to the execution layer contracts.

### Scope

This EIP covers consensus signatures only. Execution layer transaction signatures, the keys that control withdrawal addresses, KZG blob commitments, and peer-to-peer networking keys still rely on elliptic curves and require separate upgrades. In particular, [EIP-8483](./eip-8483.md) registrations are authorized by the withdrawal address, so they must be made while its key can still be trusted.

## Backwards Compatibility

This EIP changes consensus rules and requires a hard fork. It fully replaces BLS with XMSS in the consensus layer, so any software that produces, parses, or verifies consensus signatures must be updated. Validators and builders that have not registered with [EIP-8483](./eip-8483.md) are exited at the fork.

## Security Considerations

### Leaf Reuse

Signing two different messages with the same leaf compromises that leaf, allowing anyone to forge signatures with it. Running a key on multiple machines, or restoring a key without its slashing protection history, is therefore more dangerous than with BLS. Some protocol behavior also becomes a signing constraint. For example, a builder cannot bid on two different parents in the same slot, and a proposer cannot sign preferences for the same slot under two different dependent roots.

### Mass Exits

All unregistered stake exits at the fork. If registration is low, the active validator set shrinks abruptly, reducing the stake that secures finality and lowering the cost of attacks. Registration levels should be checked before the fork is scheduled. In addition, proposals and PTC votes already assigned to unregistered validators in the first epochs after the fork will be missed.

### Denial of Service

Most messages will be larger with XMSS. Larger messages make the peer-to-peer network more vulnerable to denial-of-service attacks. Every invalid message an attacker sends costs peers far more bandwidth than it would with BLS.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
