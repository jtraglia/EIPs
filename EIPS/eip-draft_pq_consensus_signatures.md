---
title: Post-Quantum Consensus Signatures
description: Replace BLS with XMSS hash-based signatures
author: Justin Traglia (@jtraglia)
discussions-to: https://ethereum-magicians.org/t/TODO
status: Draft
type: Standards Track
category: Core
created: 2026-11-07
requires: 7732, 8025, 8205, 8483
---

## Abstract

This EIP replaces BLS signatures with XMSS signatures while leaving the consensus mechanism, Gasper, unchanged.  Post-quantum keys are registered ahead of time with [EIP-8483](./eip-8483.md). At the fork, those that registered switch to their new keys and those that didn't are forcibly exited.

## Motivation

Consensus relies on BLS signatures, which are not secure against a sufficiently capable quantum computer. An adversary that can forge BLS signatures can forge blocks and attestations for any validator, finalize conflicting checkpoints, and exit validators at will. Once such an adversary is credible, Ethereum needs a consensus layer that does not depend on elliptic curves, and it may need it quickly.

This EIP is meant to be that fork. It is deliberately narrow: it swaps the signature scheme and changes only what the swap forces. Keeping the rest of the protocol intact keeps the change small enough to specify, implement, and test in advance, so that it can be scheduled on short notice. A more ambitious post-quantum redesign of consensus can follow later.

## Specification

### Signatures

Validators and builders sign with XMSS instead of BLS. A public key is 32 bytes and a signature is about 2.4 KB, compared with 48 and 96 bytes for BLS.

Signing with XMSS is stateful. A key has about four billion one-time leaves, and signing two different messages at the same leaf exposes that leaf. Each signed message therefore uses a leaf fixed by the protocol, and verification checks that leaf. An honest validator never needs to sign two different messages at the same leaf, and one key covers many decades of duties without rotation. Leaves are assigned as follows:

- Attestations use one leaf per target epoch.
- Other duties, such as blocks, aggregates, payload timeliness committee (PTC) votes, bids, and payload envelopes, use one leaf per slot and message type.
- Messages that a key signs at most once, such as a voluntary exit or a preregistration, use reserved leaves.

### Attestation aggregation

Attestations no longer carry a signature because XMSS has no native aggregation. An aggregator instead attaches a STARK proof of the signatures it collected, and the proposer merges these into a single proof that covers every attestation in its block. Aggregates are larger than with BLS, so fewer validators are selected to aggregate in each committee.

Blocks no longer include payload attestations, which spares proposers from aggregating them. Members of the PTC still broadcast their votes, and fork choice uses them as before.

### Randomness

A signature-based RANDAO reveal could be ground because XMSS signatures are not unique. Each validator instead commits to a VRF root, the root of a Merkle tree of secret per-slot values. Its RANDAO reveal for a slot, which also serves as its aggregator selection proof, is the leaf for that slot together with its Merkle branch. The tree is fixed when it is registered, so the proposer cannot grind its reveal and, as with BLS, can only choose whether to publish its block.

### Removed features

Sync committees and the light client protocol are replaced with [EIP-8025](./eip-8025.md) optional proofs. Sync committees serve only light clients, and migrating them would add hundreds of large signatures per slot. Sync committee rewards are no longer issued, which lowers issuance by about 3%. Withdrawal credential changes signed with BLS are also disabled, because those signatures could be forged.

### Deposits and execution layer requests

Execution layer request formats do not change, so existing contracts keep working. A request's 48-byte public key field holds 16 zero bytes followed by a XMSS public key. A compressed BLS public key always has its top bit set, so the two cannot be confused. Deposits no longer carry a signature, and the first 32 bytes of a deposit's signature field instead hold the new validator's VRF root.

This EIP includes [EIP-8205](./eip-8205.md) preregistration, adapted to XMSS. Since deposits are unsigned, anyone who sees the first deposit for a new key could front-run it with their own withdrawal credentials. A key holder can prevent this by first binding the key to its withdrawal credentials and VRF root with a preregistration, signed at a reserved leaf. While the binding is active, deposits for that key that do not match it are discarded.

Requests that still name a BLS public key, such as those submitted before the fork or by outdated tooling, are treated as legacy requests. A legacy deposit creates a validator that is exited at once and refunded after the withdrawability delay, and a legacy builder deposit creates a builder whose exit is initiated at once. Legacy withdrawal and consolidation requests are ignored. [EIP-8483](./eip-8483.md) registration requests are no longer processed after the fork.

## Rationale

### Minimal change

The goal is a fork that is ready before it is needed. Each departure from the current protocol is forced by a property of XMSS: no native aggregation, non-unique signatures, statefulness, and larger signatures. The committee structure, rewards other than those of sync committees, and [EIP-7732](./eip-7732.md) payload handling are unchanged. Clients can implement the fork as a delta on existing code and reuse most existing tests.

### XMSS

Hash-based signatures rely only on the security of hash functions, the most conservative assumption available. The XMSS variant used here is designed for Ethereum consensus, with small public keys and signatures that are efficient to prove in a STARK. Its main drawback, statefulness, is handled by deriving every leaf from a slot or epoch, so a validator keeps no signing counter beyond what slashing protection already records.

### Registering keys before the fork

Once BLS is broken, a BLS signature can no longer prove who announced a new key. Keys must therefore be committed before then, through a channel that does not depend on BLS. Registration through EIP-8483 uses the withdrawal address, which already controls exits and consolidations, and registration can begin long before this fork is scheduled.

### Exiting unregistered validators

A validator without a registered key has no key the protocol can trust. Keeping its BLS key would let a quantum adversary act for it, and keeping it active without a usable key would only produce missed duties and inactivity penalties. Exiting it at once returns its funds to its owner without waiting in the exit queue. Validators with `0x00` withdrawal credentials cannot be handled this way, since setting a withdrawal address requires a BLS signature.

### Replacing sync committees

Sync committees add hundreds of signatures per slot that serve only light clients. With XMSS, each signature is about 2.4 KB, and proving them would compete with attestations for prover capacity. Relying on EIP-8025 optional proofs instead avoids migrating them and keeps the fork small.

### Unsigned deposits

A BLS deposit signature is a proof of possession, which prevents rogue-key attacks on aggregate signatures. A STARK proves each signature against its own key, so this attack does not apply and a proof of possession is not needed. The remaining risk, deposit front-running, is handled by EIP-8205 preregistration. Keeping the existing request formats avoids any change to the execution layer contracts.

## Backwards Compatibility

This EIP changes consensus rules and requires a scheduled hard fork. It affects participants and tooling as follows:

- Validators and builders that do not register with EIP-8483 before the fork are exited at the fork.
- Validator clients, remote signers, and slashing protection must switch to XMSS and track signed leaves.
- Deposit tooling must produce the new deposit format and should preregister keys before depositing.
- Light clients, and anything else that consumes sync committee data, must switch to optional proofs.
- Software that verifies or parses BLS signatures on consensus objects must be updated.

## Security Considerations

### Leaf reuse

Signing two different messages at the same leaf exposes that leaf, after which anyone can forge signatures at it. Running one key on two machines, or restoring a key without its slashing protection history, is therefore more dangerous than with BLS. Some protocol behavior also becomes a signing constraint. For example, a builder cannot bid on two different parents in the same slot, and a proposer cannot sign preferences for the same slot under two different dependent roots.

### Stake leaving at the fork

Unregistered stake leaves at the fork epoch all at once, without churn limits. If registration is low, the active validator set shrinks abruptly, which weakens finality and lowers the cost of attacks. Registration levels should be checked before the fork is scheduled. The first epochs after the fork also miss the proposals and PTC votes already assigned to unregistered validators.

### Pre-fork offenses

Slashing evidence signed with BLS before the fork cannot be included after it, and unregistered validators also skip the withdrawability delay. A validator that misbehaves shortly before the fork may therefore escape its penalties.

### STARK proofs

Block validity depends on the soundness of the STARK proof system. The production proof system, its size bounds, and its proving costs still need to be specified.

### Scope

This EIP covers consensus signatures only. Execution layer transaction signatures, the keys that control withdrawal addresses, KZG blob commitments, and peer-to-peer networking keys still rely on elliptic curves and need separate upgrades. In particular, EIP-8483 registrations are authorized by the withdrawal address, so they must be made while its key can still be trusted.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
