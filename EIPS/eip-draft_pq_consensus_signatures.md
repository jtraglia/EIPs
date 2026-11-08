---
title: Post-Quantum Consensus Signatures
description: Replace BLS with XMSS
author: Justin Traglia (@jtraglia)
discussions-to: https://ethereum-magicians.org/t/TODO
status: Draft
type: Standards Track
category: Core
created: 2026-11-07
requires: 7732, 8025, 8205, 8483
---

## Abstract

This EIP replaces BLS with XMSS for consensus signatures while leaving the consensus mechanism unchanged. XMSS is a hash-based signature scheme that is believed to be post-quantum secure. Before the fork, validators and builders register XMSS keys through [EIP-8483](./eip-8483.md). At the fork, those that registered switch to their XMSS keys, and those that did not are exited.

## Motivation

The consensus layer relies on BLS signatures, which a sufficiently capable quantum computer could forge. Such an adversary could sign blocks and attestations on behalf of any validator, finalize conflicting checkpoints, and exit validators at will. Because the timeline for such a computer is uncertain, Ethereum should move its consensus layer off elliptic curves before one exists.

This EIP defines a deliberately narrow fork for that transition. It replaces the signature scheme and changes only what the replacement requires. This keeps the change small enough to specify, implement, and test in advance, so that it can be scheduled quickly when needed. A broader post-quantum redesign of consensus can follow separately.

## Specification

### Signatures

Validators and builders sign with XMSS instead of BLS. XMSS public keys are 32 bytes and signatures are about 2.4 KB, compared with 48 and 96 bytes for BLS.

XMSS is stateful. Each key covers approximately 4 billion one-time leaves, and signing two different messages with the same leaf compromises that leaf. The protocol therefore assigns a fixed leaf to each signed message, and verification checks that the signature uses it. Leaves are assigned as follows:

- Attestations use one leaf per target epoch.
- Messages signed regularly, such as block proposals, use one leaf per slot and message type.
- Messages signed at most once per key, such as voluntary exits, use reserved leaves.

Under this assignment, an honest validator never signs two different messages with the same leaf, and a single key covers many decades of duties without rotation.

### Attestation aggregation

XMSS does not support native aggregation, so aggregate attestations no longer carry a signature. Instead, an aggregator attaches a STARK proof of the signatures it collected, and the proposer merges these proofs into a single proof covering every attestation in its block.

### Randomness

Because XMSS signatures are not unique, a proposer could grind a signature-based RANDAO reveal. Instead, each validator commits to a VRF root: the root of a Merkle tree of secret per-slot values. A validator's RANDAO reveal for a slot is that slot's leaf and its Merkle branch, and the same value serves as its aggregator selection proof. Because the tree is fixed at registration, a proposer cannot grind its reveal and, as with BLS, can only choose whether to publish its block.

### Sync committees

Sync committees and the light client protocol are removed in favor of [EIP-8025](./eip-8025.md) optional proofs. Without native aggregation, the 512 sync committee signatures per slot would add either about 1.2 MB to each block or a second STARK proof competing with attestations for prover capacity, solely to serve light clients. Sync committee rewards are no longer issued; the resulting reduction in issuance, about 3%, is a side effect rather than a goal.

### Withdrawal credential changes

BLS-signed withdrawal credential changes are no longer accepted. This fork assumes that BLS signatures can no longer be trusted, so an attacker could forge such a change to redirect a validator's withdrawals to an address it controls. Validators with `0x00` withdrawal credentials must therefore set a withdrawal address before the fork. Those that do not can neither register an XMSS key through EIP-8483 nor withdraw their funds after the fork.

### Deposits and execution layer requests

Execution layer request formats are unchanged, so existing contracts continue to work. The 48-byte public key field of a request holds 16 zero bytes followed by the 32-byte XMSS public key. Because a compressed BLS public key always has its most significant bit set, the two cannot be confused. Deposits are no longer signed; instead, the first 32 bytes of the signature field hold the new validator's VRF root.

This EIP incorporates [EIP-8205](./eip-8205.md) preregistration, adapted to XMSS. Because deposits are unsigned, anyone who observes the first deposit for a new key could front-run it with a deposit to their own withdrawal credentials. To prevent this, a key holder can first submit a preregistration, signed with a reserved leaf, that binds the key to its withdrawal credentials and VRF root. While the binding is active, deposits for that key that do not match it are discarded.

Requests that still contain a BLS public key, such as those submitted before the fork or by outdated tooling, are treated as legacy requests:

- A legacy deposit creates a validator that is exited immediately and refunded after the withdrawability delay.
- A legacy builder deposit creates a builder whose exit is initiated immediately.
- Legacy withdrawal and consolidation requests are ignored.

[EIP-8483](./eip-8483.md) registration requests are no longer processed after the fork.

## Rationale

### Minimal change

Each departure from the current protocol follows from a property of XMSS: no native aggregation, non-unique signatures, statefulness, or larger signatures. Committee structure, rewards other than sync committee rewards, and [EIP-7732](./eip-7732.md) payload handling are unchanged, so clients can implement the fork as a delta on existing code and reuse most existing tests.

### XMSS

Hash-based signatures depend only on the security of hash functions, among the most conservative cryptographic assumptions. The XMSS variant used here is designed for Ethereum consensus, with small public keys and signatures that are efficient to prove in a STARK. Its main drawback, statefulness, is addressed by deriving every leaf from a slot or epoch, so validators keep no signing state beyond what slashing protection already records.

### Registering keys before the fork

Once BLS is broken, a BLS signature can no longer authenticate a new key. Keys must therefore be committed beforehand, through a channel that does not depend on BLS. EIP-8483 registration is authorized by the withdrawal address, which already controls exits and consolidations, and can begin long before this fork is scheduled.

### Exiting unregistered validators

A validator without a registered key has no key the protocol can trust. Retaining its BLS key would let a quantum adversary act on its behalf, and keeping it active without a usable key would only incur missed duties and inactivity penalties. Exiting it immediately, without waiting in the exit queue, returns its funds to its owner. This does not apply to validators that still have `0x00` withdrawal credentials at the fork: they have no withdrawal address, and setting one requires a BLS signature.

### Unsigned deposits

A BLS deposit signature is a proof of possession, which prevents rogue-key attacks on aggregate signatures. A STARK verifies each signature against its own key, so this attack does not apply and no proof of possession is needed. The remaining risk, deposit front-running, is addressed by EIP-8205 preregistration. Keeping the existing request formats avoids changes to the execution layer contracts.

## Backwards Compatibility

This EIP changes consensus rules and requires a hard fork. It affects participants and tooling as follows:

- Validators and builders that have not registered with EIP-8483 are exited at the fork.
- Validator clients, remote signers, and slashing protection must switch to XMSS and track signed leaves.
- Deposit tooling must produce the new deposit format and should preregister keys before depositing.
- Light clients and other consumers of sync committee data must switch to optional proofs.
- Software that parses or verifies BLS signatures on consensus objects must be updated.

## Security Considerations

### Leaf reuse

Signing two different messages with the same leaf compromises that leaf, allowing anyone to forge signatures with it. Running a key on multiple machines, or restoring a key without its slashing protection history, is therefore more dangerous than with BLS. Some protocol behavior also becomes a signing constraint. For example, a builder cannot bid on two different parents in the same slot, and a proposer cannot sign preferences for the same slot under two different dependent roots.

### Stake leaving at the fork

All unregistered stake exits at the fork epoch, bypassing churn limits. If registration is low, the active validator set shrinks abruptly, reducing the stake that secures finality and lowering the cost of attacks. Registration levels should be checked before the fork is scheduled. In addition, proposals and PTC votes already assigned to unregistered validators in the first epochs after the fork will be missed.

### Pre-fork offenses

BLS-signed slashing evidence from before the fork cannot be included after it, and unregistered validators bypass the withdrawability delay. A validator that commits a slashable offense shortly before the fork may therefore avoid its penalties.

### STARK proofs

Block validity depends on the soundness of the STARK proof system. The production proof system, its size bounds, and its proving costs are not yet specified.

### Scope

This EIP covers consensus signatures only. Execution layer transaction signatures, the keys that control withdrawal addresses, KZG blob commitments, and peer-to-peer networking keys still rely on elliptic curves and require separate upgrades. In particular, EIP-8483 registrations are authorized by the withdrawal address, so they must be made while its key can still be trusted.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
