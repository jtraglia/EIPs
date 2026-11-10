---
title: Post-Quantum Consensus Signatures
description: Replace BLS with XMSS
author: Justin Traglia (@jtraglia), Mikhail Kalinin (@mkalinin), Kamil Salakhiev (@kamilsa), Thomas Coratger (@tcoratger)
discussions-to: https://ethereum-magicians.org/t/TODO
status: Draft
type: Standards Track
category: Core
created: 2026-11-07
requires: 7732, 8025, 8205, 8483
---

## Abstract

This EIP replaces BLS with the eXtended Merkle Signature Scheme (XMSS) for consensus signatures while leaving the consensus mechanism unchanged. XMSS is a hash-based signature scheme that is believed to be post-quantum secure. Because XMSS has no native aggregation, STARK proofs replace aggregate signatures. Before the fork, validators and builders register XMSS keys through [EIP-8483](./eip-8483.md). At the fork, those that registered switch to their XMSS keys, and those that did not are exited.

## Motivation

The consensus layer relies on BLS signatures, which a cryptographically relevant quantum computer (CRQC) could forge, allowing an attacker to sign messages on behalf of any validator. The protocol must switch to a post-quantum signature scheme before there is a credible threat. This EIP defines the minimal set of changes required to switch to XMSS signatures.

## Specification

### Signatures

XMSS public keys replace BLS public keys in validator and builder structures, and XMSS signatures replace BLS signatures in signed messages. Each message is signed at a fixed leaf (one-time key pair). Most messages use a leaf derived from their slot and type, attestations use a leaf derived from their target epoch, and messages that a key signs at most once, such as voluntary exits, use reserved leaves. Each signature includes the index of the leaf that produced it. Verifiers derive the expected leaf from the message and reject any signature with a different leaf index. Aggregate attestations no longer carry a signature. Instead, aggregators attach a STARK proof to each aggregate they publish. Each block carries a single proof covering its attestations, which the proposer obtains by merging aggregate proofs or by reusing the proof of a single aggregate.

### RANDAO

A verifiable random function (VRF) root, the root of a Merkle tree of secret values with one per slot, is added to the validator structure. RANDAO reveals and aggregator selection proofs are no longer BLS signatures. Instead, each consists of the validator's secret value for the slot and a Merkle branch proving that value against the validator's VRF root.

### Key Migration

At the fork, validators that registered through [EIP-8483](./eip-8483.md) adopt their registered XMSS public key and VRF root, while builders that registered adopt only their XMSS public key. Validators and builders that did not register are exited and become withdrawable at the fork, bypassing the exit queue and the usual withdrawal delay. Slashed validators keep their original withdrawal delay, so their slashing penalties still apply. Their XMSS public key and VRF root are the SHA-256 hashes of their BLS public key followed by distinct domain separators. No one knows the secrets for these values. The fork leaves the proposer and payload timeliness committee lookaheads unchanged, so duties assigned to unregistered validators before the fork are missed.

### Execution Requests

In execution requests, the XMSS public key goes in the BLS public key field. In validator deposit requests, the VRF root goes in the BLS signature field, which is available because these requests are no longer signed. Requests that still contain a BLS public key are legacy requests, which are identified by a public key field that does not start with the 16 zero bytes that precede an XMSS public key. A legacy deposit uses the XMSS public key and VRF root derived from its BLS public key, as for validators and builders that did not register. A legacy deposit whose derived XMSS public key is new creates a validator or builder that is immediately exited, so its funds are returned after the usual delay. Legacy withdrawal and consolidation requests are ignored.

### Request Migration

At the fork, each pending deposit for an existing validator is updated to name that validator's new XMSS public key, so it still tops the validator up. A top-up for an unregistered validator does not consume churn, since the validator is already withdrawable, and is withdrawn with the rest of its balance. A pending deposit for a new validator keeps its BLS public key and is processed as a legacy deposit. Pending withdrawals and consolidations identify validators by index, so the fork leaves them unchanged.

### Removed Features

BLS-signed withdrawal credential changes are no longer accepted, since an attacker who can forge BLS signatures could use one to redirect a validator's withdrawals. Blocks no longer carry payload attestations, which spares proposers from aggregating them with a second STARK proof. Execution requests no longer carry [EIP-8483](./eip-8483.md) registrations, since registrations only take effect at the fork. Proposer and attester slashings of messages signed before the fork are no longer accepted, since verifying their BLS signatures would require BLS public keys that the state no longer holds. Sync committees and the light client protocol built on them are removed, since including their signatures in every block would be too costly.

### Networking

Attestation subnets are reduced from 64 to 16, so more of each committee's attestations share a subnet and aggregators collect them faster. The `attnets` bitfield keeps its length, so node metadata and Ethereum Node Records (ENRs) are unchanged, but only the bits of the first 16 subnets may be set. The target number of aggregators per committee is reduced from 16 to 4, which reduces the number of large aggregates propagated across the network.

## Rationale

### XMSS

Hash-based signatures depend only on the security of hash functions, among the most conservative cryptographic assumptions. This EIP uses leanXMSS, a variant of XMSS designed for Ethereum consensus, with small public keys, and with signatures that are efficient to prove in a STARK. XMSS is stateful, since signing two different messages at the same leaf compromises it, but assigning each message a fixed leaf makes this manageable. Compared with SPHINCS+, the stateless alternative, XMSS signatures are about a third of the size and about three times faster to prove in a STARK with current implementations.

### Contingency Plan

Replacing BLS with XMSS without changing the consensus mechanism would address the most critical vulnerability quickly. Even if this approach proves feasible but unattractive for other reasons, it can still serve as a contingency plan, ready to ship in an emergency while work on a new consensus mechanism continues. Evaluating its feasibility now avoids a situation where a new consensus mechanism must be shipped before it has been properly tested. If a new consensus mechanism turns out to be required to accommodate larger signatures, the work here is not wasted.

### Unsigned Deposits

A BLS deposit signature is a proof of possession, which prevents rogue-key attacks on aggregate signatures. A STARK proof verifies each signature against its own key, so this attack does not apply and no proof of possession is needed. Without signatures, anyone who observes the first deposit for a new key could front-run it with a deposit to their own withdrawal credentials. [EIP-8205](./eip-8205.md) preregistrations prevent this. Keeping the existing request formats avoids changes to the existing execution layer contracts.

### Sync Committees

Sync committees are removed because, without native aggregation, their 512 signatures per slot would add about 1.2 MB to each block, or require a second STARK proof competing with attestations for prover capacity, solely to serve light clients. Light clients can instead rely on an alternative such as [EIP-8025](./eip-8025.md) optional proofs.

### Scope

This EIP covers consensus signatures only. Execution layer transaction signatures, the keys that control withdrawal addresses, KZG blob commitments, and peer-to-peer networking keys still rely on elliptic curves and require separate upgrades. In particular, [EIP-8483](./eip-8483.md) registrations are authorized by the withdrawal address, so they must be made while its key can still be trusted.

## Backwards Compatibility

This EIP changes consensus rules and requires a hard fork. It fully replaces BLS with XMSS in the consensus layer, so any software that produces, parses, or verifies consensus signatures must be updated. Validators and builders that have not registered with [EIP-8483](./eip-8483.md) are exited at the fork. Light clients that follow the chain through sync committees stop working at the fork.

## Security Considerations

### Leaf Reuse

Reusing a leaf for two different messages lets anyone forge signatures at that leaf. Running a key on multiple machines, or restoring a key without its slashing protection history, is therefore more dangerous than with BLS. Some protocol behavior also becomes a signing constraint. For example, a builder cannot raise a bid it has already broadcast or bid on two different parents in the same slot, and a proposer cannot sign preferences for the same slot under two different dependent roots.

### Mass Exits

All unregistered stake exits at the fork. If registration is low, the active validator set shrinks significantly, reducing the stake that secures finality and lowering the cost of attacks. Registration levels should be checked before the fork is scheduled. In addition, proposals and payload timeliness committee votes already assigned to unregistered validators in the first epochs after the fork are missed.

### Instant Exits

Validators that do not register skip the exit queue and the withdrawal delay, so they can withdraw their full balance at the fork. Slashed validators are the exception: they keep their withdrawal delay, so their full slashing penalty still applies. Not registering also lets any validator, including one already in the exit queue, leave at the fork instead of at the rate the churn limit normally allows.

### Pre-Fork Offenses

Slashings of messages signed before the fork are not accepted after it, so an offense that is not slashed before the fork goes unpunished, whether or not the validator registered. An attacker could exploit this by attacking consensus in the final epochs before the fork, for example by trying to finalize conflicting checkpoints. If its validators do not register, they can also withdraw their full balance at the fork, so the attacker leaves with its stake intact. Any later upgrade that supports these slashings must stop accepting them before BLS signatures can be forged, or an attacker with a CRQC could forge slashings against any validator from before the fork.

### Denial of Service

With XMSS, most messages are larger, which makes the peer-to-peer network more vulnerable to denial-of-service attacks. Every invalid message an attacker sends costs peers far more bandwidth than it would with BLS.

### Prover Capacity

A block's proof normally comes from merging the proofs of all the aggregates it includes, which is relatively expensive. A proposer that cannot merge in time can still produce a valid block by reusing the proof of a single aggregate, but that block includes only one committee's attestations. This is harmless when it happens occasionally, since later proposers can include the attestations left out. If most proposers cannot merge, however, too few attestations reach the chain for it to justify and finalize.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
