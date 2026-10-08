
```
IIP: 64
Title: Post-Quantum Cryptographic Migration for IoTeX
Author: Xinxin Fan (@cryptoxfan)
Discussions-to: https://community.iotex.io/c/governance-proposals
Status: Draft
Type: Standards Track
Category: Core
Created: 2026-05-14
Updated: 2026-10-08
Requires: —
Replaces: N/A
```

> **Editor's revision note (Revision 2, 2026-10-08).** This revision incorporates the editorial and technical review of PR #72. Every change is enumerated in §19 (Changelog). Normative requirements live in §4 (Specification) and §§12–15; discussion of the open community debate about the *pace* of migration is confined to §1.5 and §18 and does not alter the specification. The parameter tables in §4.1 are the single source of truth for algorithm identifiers and sizes.

## Abstract

This IoTeX Improvement Proposal (IIP) specifies a comprehensive, phased migration of the IoTeX blockchain from quantum-vulnerable elliptic-curve cryptography (secp256k1/ECDSA) to NIST-standardized post-quantum cryptographic (PQC) algorithms. The proposal draws on published research and migration frameworks from Ethereum, Bitcoin (BIP-361 and Chaincode Labs), Sui, the Quantum Resistant Ledger (QRL), Google Quantum AI, the Coinbase Independent Advisory Board, Meta, and Paradigm, synthesizing their findings into a specification tailored to IoTeX's Roll-DPoS consensus, 5-second block time, 24-delegate architecture, EVM compatibility, and DePIN ecosystem.

The migration proceeds across five phases (2026–2033) and introduces: a PQ signature verification precompile; an EIP-2718-compatible PQ-authenticated transaction type; a PQ Key Registry; consensus-layer dual-signing with PQ checkpoint attestations; zkSTARK-based migration proofs for hidden-key accounts; a hash-hidden ("bunker") address strategy for accounts whose public keys are not yet exposed; the iPACT address-control timestamp scheme for pre-committed rescue; a five-level IoTeX PQC Maturity Model with on-chain metrics (§7); protocol guardrails that prevent new quantum-vulnerable deployments (§8); a quantum-milestone-triggered acceleration mechanism mapped to observable milestones (§9); and a cryptographic-agility framework with an explicit re-parameterization trigger (§4.11).

Upon completion, all IoTeX transactions, consensus attestations, and DePIN device identities will be secured exclusively by post-quantum cryptographic algorithms, achieving the highest maturity level (PQ-Enabled) before the projected arrival of cryptographically-relevant quantum computers (CRQCs).

## 1. Motivation

### 1.1 The Quantum Threat to the IoTeX Blockchain

Account security on the IoTeX blockchain relies on the intractability of the Elliptic Curve Discrete Logarithm Problem (ECDLP). Every Externally Owned Account (EOA) is associated with a private/public key pair generated on the secp256k1 curve, from which the IoTeX address is derived via public key hashing. Transaction authorization utilizes ECDSA signatures over secp256k1. While this establishes the same cryptographic foundation used by Bitcoin and Ethereum, it remains inherently vulnerable to quantum attacks.

[Google Quantum AI's whitepaper](https://quantumai.google/static/site-assets/downloads/cryptocurrency-whitepaper.pdf) published in March 2026 demonstrates that the resource estimates for breaking ECDLP on the secp256k1 curve have decreased by approximately 20× compared to prior literature. The paper presents two optimized Shor-based circuits. One circuit requires fewer than 1,200 logical qubits and 90 million Toffoli gates; another requires fewer than 1,450 logical qubits and 70 million Toffoli gates. These circuits could be executed on a superconducting CRQC with fewer than 500,000 physical qubits and complete within minutes. This places the threat horizon within the plausible engineering trajectory of the next 10–15 years.

> *These forward-looking resource estimates are treated in this IIP as an assumption (§1.5), not as an established result. The specification is designed so that its security does not depend on the estimates being exactly correct.*

### 1.2 Classification of IoTeX Quantum Vulnerabilities

Following the taxonomy from [Google Quantum AI's whitepaper](https://quantumai.google/static/site-assets/downloads/cryptocurrency-whitepaper.pdf) and the [Chaincode Labs' Bitcoin report](https://chaincode.com/bitcoin-post-quantum.pdf), we identify IoTeX's attack surfaces in order of severity:

* **At-Rest Attacks:** These attacks target accounts whose public keys are already exposed on-chain (i.e., any EOA that has ever signed a transaction). Since IoTeX addresses are derived from a Keccak-256 hash of the public key and the public key is recoverable from any signed transaction, the set of exposed accounts grows monotonically with network activity. A quantum adversary with a slow-clock CRQC (days of compute) can drain any such account.

* **On-Spend Attacks:** These attacks target transactions in the mempool. Although IoTeX's 5-second block time is significantly shorter than Bitcoin's 10 minutes, it is still vulnerable to fast-clock CRQCs capable of solving ECDLP in minutes, particularly if the attacker has precomputed the "primed" state that can halve the attack time to ~9 minutes per Google's estimates. Moreover, delegate collusion or adversarial delegate replacement could extend the effective attack window.

* **On-Setup Attacks:** These attacks target fixed public parameters. IoTeX's current protocol does not use trusted setups or pairing-based data availability sampling, making this vector less relevant at present. However, future integration with zk-rollup Layer-2 solutions or verifiable compute systems could introduce this risk.

* **Consensus-Layer Attacks:** These attacks target the 24 Roll-DPoS delegates. If a CRQC can derive delegate signing keys, it can produce valid block proposals, attestations, and governance votes, enabling chain reorganizations or protocol manipulation.

* **DePIN-Specific Attacks:** These attacks target long-lived device identities and data provenance proofs. IoT devices may remain operational for 10–20 years, which is well into the CRQC era. Device key compromise enables data falsification, unauthorized device impersonation, and fraudulent machine-to-machine payments.

### 1.3 Why Act Now?

[NIST](https://nvlpubs.nist.gov/nistpubs/ir/2024/NIST.IR.8547.ipd.pdf) mandates the deprecation of quantum-vulnerable algorithms by 2030 and their disallowance by 2035. [Google](https://blog.google/innovation-and-ai/technology/safety-security/cryptography-migration-timeline/) has announced a 2029 target for complete internal PQC migration and [Meta](https://engineering.fb.com/2026/04/16/security/post-quantum-cryptography-migration-at-meta-framework-lessons-and-takeaways/) has begun deploying PQ-TLS internally and published a detailed migration framework. The [Coinbase Independent Advisory Board](https://assets.ctfassets.net/sygt3q11s4a9/6EjYavuGdtJDYCqaJrASj9/9f464a8bf26f44bd6c85710fe7e4a29f/Quantum_Computing_and_Blockchain_v10.3_15April2026.pdf) urges all blockchain ecosystems to begin migration now, warning that "wallets that never upgrade may leave assets exposed." [Ethereum](https://pq.ethereum.org/) targets L1 PQ upgrades by approximately 2029 and [Bitcoin's BIP-361](https://github.com/bitcoin/bips/blob/master/bip-0361.mediawiki) proposes a two-phase soft-fork migration. IoTeX must act in concert with this industry-wide transition.

The case for acting now is **asymmetric cost and lead time**, not an assertion that a break is imminent. Wiring a PQ verifier, a key registry, and a dual-signed consensus into a live chain is a multi-year engineering and governance effort; starting it early is cheap relative to being late. A rushed, improvised migration after a break would be far more damaging (see §1.5).

### 1.4 Conventions and Normative Language

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** in this document are to be interpreted as described in RFC 2119.

This IIP uses two timeline labels defined in §2.2:

* **T-governance** — the point at which the protocol disables legacy ECDSA *by policy*, while ECDSA is still cryptographically sound.
* **T-break** — the point at which a CRQC (or a classical break) permits derivation of secp256k1 private keys from exposed public keys.

Mechanisms are labeled with respect to which timeline they remain valid in. Unless stated otherwise, a requirement applies to mainnet after the referenced phase activates. This IIP is a **Standards Track / Core** proposal; it does not itself create a token or funding mechanism, and any subsidy referenced in §13 MUST be separately authorized by governance.

### 1.5 Threat-Model Assumptions (and What This IIP Does Not Claim)

This IIP is a risk-management action, and it deliberately separates **assumptions** from **findings**:

* It does **not** claim that ECDLP has been broken, that a classical break of elliptic curves is imminent, or that lattice problems are known to be weak. There is, as of this writing, no public evidence of a break in the underlying hardness assumptions.
* It **does** assume that (a) a cryptographically-relevant quantum computer is plausible within the 2035+ horizon per the Coinbase Independent Advisory Board, and (b) AI-accelerated mathematical discovery could eventually degrade the concrete security of *structured* assumptions — including both elliptic curves and lattices — faster than linear extrapolation suggests. These are assumptions about risk, adopted because the cost of being wrong in the "too late" direction is catastrophic and one-directional.

The current public debate (Drake's "bunker mode" call for maximal acceleration and hash-only cryptography; Buterin's argument that lattices — not hashes — are the newly exposed assumption and that parameter sizes should be treated with extra paranoia; Lindell's rebuttal that there is no evidence of a break and that such claims constitute FUD) does **not** change this IIP's specifications, but it directly informed two design decisions that are stated normatively elsewhere:

1. **Hash-hidden addresses (§2.6)** are supported as a first-class, zero-cryptography-risk defensive posture, independent of any protocol change.
2. **The hash-based fallback is elevated to a co-primary, not a last resort (§4.1, §4.11):** SLH-DSA is offered as an equal-status option wherever signature size is tolerable, so that a deployer who wishes to avoid lattice assumptions entirely can do so without waiting for a protocol change. See §4.13 (Rationale).

## 2. Threat Model and Risk Assessment

### 2.1 Adversary Model

This IIP considers the following three classes of quantum adversary, ordered by increasing capability.

* **Class 1: Harvest-Now-Decrypt-Later (HNDL).** A classical adversary records all on-chain public keys and transaction data today, intending to extract private keys once a CRQC becomes available. This is the most immediate threat because it requires no quantum hardware today.

* **Class 2: Offline CRQC.** An adversary with access to a CRQC but without real-time capability. This adversary can break any exposed secp256k1 public key given hours to days of quantum computation. All at-rest keys with prior on-chain exposure are vulnerable.

* **Class 3: Real-Time CRQC.** An adversary with a CRQC capable of solving ECDLP-256 within the network's block confirmation window (5 seconds for IoTeX). Such an adversary can intercept transactions in the mempool, extract the signing key from the revealed public key, and broadcast a competing transaction before the original one is finalized. This is the most severe threat and the one that requires the network to have fully completed migration.

### 2.2 Two Timelines: T-governance and T-break

Migration mechanisms must be evaluated against two distinct events. Conflating them is a common and dangerous error, because several widely proposed "rescue" mechanisms are forged by a CRQC that already knows the exposed private key.

| Event | Definition | What is true at this point |
| --- | --- | --- |
| **T-governance** | The protocol disables legacy ECDSA by policy. A CRQC may or may not exist. | Exposed keys are still secret to everyone but their owner, **in principle**. A mechanism that requires knowledge of the private key *can* still distinguish owner from attacker. |
| **T-break** | A CRQC (or a classical algorithmic break) allows derivation of `sk` from any exposed public key. | For an **already-exposed** key, the attacker knows `sk`. Any predicate the owner can satisfy *from `sk` alone* is equally satisfiable by the attacker. |

**Normative consequence.** A rescue or registration mechanism MUST be labeled **T-governance-only** or **T-break-sound**, and the network MUST NOT rely on a T-governance-only mechanism to recover funds after T-break. Specifically:

* Mechanisms that prove ownership using an **ECDSA signature** (or otherwise using an exposed `sk`) are **T-governance-only** and MUST be labeled as such wherever they appear.
* Mechanisms that prove ownership using a secret **not derivable from the exposed public key** — an off-chain salt/commitment (iPACT, §4.10), or a mnemonic/seed preimage that is not invertible to `sk` (§4.9) — are **T-break-sound** for the accounts that committed in advance.

### 2.3 IoTeX-Specific Risk Matrix

The IoTeX components vulnerable to quantum computers are summarized below. "Likelihood (15 years)" is the probability that the component's protective assumption is *concretely* weakened enough to matter within the horizon, under the assumptions of §1.5.

| **Component** | **Vulnerability** | **Attack Type** | **Likelihood (15 Years)** | **Impact** | **Risk** |
| --- | --- | --- | --- | --- | --- |
| **EOA keys (secp256k1, exposed)** | ECDLP via Shor's | At-rest / On-spend | High | High | **High** |
| **EOA keys (secp256k1, unexposed)** | Preimage of Keccak-256 | Grover's (preimage) | Low | High | Low |
| **Delegate signing keys** | ECDLP via Shor's | At-rest | High | High | **High** |
| **Smart contract admin keys** | ECDLP via Shor's | At-rest | High | High | **High** |
| **DePIN device keys** | ECDLP via Shor's | At-rest | High | High | **High** |
| **Keccak-256 addresses** | Grover's (preimage) | Brute-force | Low | High | Low |
| **SHA-256 (Merkle trees)** | Grover's (collision, BHT) | Collision | Medium | Medium | Medium |
| **TLS/P2P networking** | ECDH key exchange | Harvest-now | Medium | Medium | Medium |
| **Lattice-based primitives (proposed)** | Unknown structure exploitation | AI-accelerated cryptanalysis | Medium | High | **Medium–High** |

The final row is new in this revision and is intentional: per §1.5, lattice-based post-quantum primitives are treated as carrying a *concrete-security* risk (not a break) that must be managed through parameter sizing and agility, which is why §4.1 sets a Category-3 floor and §4.11 defines a re-parameterization trigger.

### 2.4 Quantitative Exposure Analysis

IoTeX addresses that have initiated at least one transaction have an on-chain-recoverable public key. Based on IoTeX mainnet state, the majority of active accounts fall into this category. The proportion of value secured by exposed public keys represents the primary risk surface. Accounts that have never transacted (receive-only) retain hash-hiding protection, analogous to Bitcoin's P2PKH. However, as noted in the [Kiraz–Kardas framework](https://eprint.iacr.org/2026/352.pdf), this protection is reduced (not eliminated) in the quantum setting. Specifically, Grover's algorithm reduces the effective preimage security of Keccak-256 from 256 bits to approximately 128 bits, which remains substantial but warrants proactive migration rather than reliance on hash security alone.

> **Action item (normative deliverable, not normative rule):** the pre-migration baseline MUST include a published on-chain census quantifying (a) the share of value held in exposed-key accounts, (b) the share held in never-transacted accounts, (c) the distribution by balance bucket, and (d) the share held in contracts whose admin/owner keys are EOA-exposed. This census is the denominator for every migration-progress metric in §7.

### 2.5 What Is Not Threatened

Hash functions (e.g., Keccak-256, SHA-256) used in address derivation, Merkle trees, and block hashing provide modest quantum resistance via Grover's algorithm (which halves the effective security bits of symmetric primitives, yielding ~2^128 preimage work for a 256-bit digest) and BHT-style collision algorithms. These algorithms do not require immediate replacement, but outputs SHOULD be sized so that preimage resistance is at least 256-bit (→ ≥512-bit digest) and collision resistance at least 256-bit (→ ≥384-bit digest, since BHT gives ~2^(n/3)). IoTeX's existing Keccak-256 usage meets the *preimage* requirement but provides only ~2^85 collision resistance; this is acceptable for current uses but is the reason §4.11 permits raising hash output sizes and round counts, *before* changing byte widths, if concern grows.

### 2.6 Hash-Hidden Address ("Bunker") Strategy

A separate, protocol-independent defensive layer exists and this IIP adopts it explicitly. An account whose secp256k1 public key has **never been published on-chain** is protected against at-rest attacks by the ~2^128 preimage resistance of Keccak-256 (§2.5). This protection requires no new cryptography and no protocol change.

The IIP therefore specifies the following operational guidance and one **hard warning about the migration path itself**:

* **Bunker posture.** Holders SHOULD keep the bulk of long-term balances in addresses that have never signed a transaction. When such an address must sign, its holder SHOULD, in the same or subsequent transaction, move the remaining balance to a fresh never-signed address (optionally derived from the same seed).
* **Do not de-anonymize on registration.** The dual-signature registration path of §4.5 publishes an ECDSA public key and therefore **destroys** hash-hiding. Accounts whose public key is not already exposed MUST use the zkSTARK registration path (`registerWithProof`) described in §4.5 and §4.9, not the dual-signature path.
* **Rotation.** Load-bearing signers (oracles, bridges, security councils) SHOULD rotate keys off-chain where possible and, where on-chain signing is unavoidable, minimize the number of signatures a single long-lived key ever produces.

This section exists because a migration path that moves a currently-safe account onto a signature scheme using a dual-signed registration would *reduce* that account's security during the transition. The warning is normative: **a migration MUST NOT silently convert a hash-hidden account into an exposed one.**

## 3. Comparative Analysis of Existing Proposals

### 3.1 Ethereum Foundation PQ Roadmap

The Ethereum Foundation's Post-Quantum team defines a [multi-layer, multi-fork migration](https://pq.ethereum.org/):

* **Execution Layer:** evolve through three stages — PQ signature precompiles (Fork J\*), PQ transactions (longer-term), and PQ signature aggregation (Fork M\*). The strategy relies on account abstraction ([ERC-4337](https://eips.ethereum.org/EIPS/eip-4337) / [EIP-8141](https://eips.ethereum.org/EIPS/eip-8141)) as the central upgrade mechanism, allowing smart contract accounts to define custom PQ verification logic without protocol changes. Signature candidates under evaluation include Falcon (FN-DSA/FIPS 206), [ML-DSA](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.204.pdf) (FIPS 204), and [SLH-DSA](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.205.pdf) (FIPS 205).
* **Consensus Layer:** PQ key registry (Fork I\*), PQ attestations with real-time CL proofs using leanXMSS and leanVM (Fork L\*), and full PQ consensus (longer-term). This replaces BLS with hash-based signatures (i.e., [leanXMSS](https://github.com/leanEthereum/leanSig)) and uses SNARK-based aggregation via a minimal zkVM ([leanMultisig](https://github.com/leanEthereum/leanMultisig)) to restore aggregation properties lost by abandoning BLS.
* **Data Layer:** leanVM (Fork L\*) and PQ blobs (Fork M\*). PQ-safe data availability is an emerging research area.

Ethereum L1 protocol upgrades could be completed by 2029, with full execution-layer migration extending years beyond. **Relevant to IoTeX:** the EF roadmap has moved decisively toward *hash-based* signing (leanXMSS/WOTS) rather than lattice signing for its lean path. This IIP does not mandate that choice, but per §4.1 it elevates SLH-DSA to co-primary status so an IoTeX deployer can adopt a hash-only posture without a protocol change.

### 3.2 Google Quantum AI Whitepaper

The [Google whitepaper](https://quantumai.google/static/site-assets/downloads/cryptocurrency-whitepaper.pdf) introduces a critical architectural distinction between "fast-clock" CRQCs (superconducting, photonic, silicon — minutes for ECDLP) and "slow-clock" CRQCs (neutral atom, ion trap — days for ECDLP). Key findings: (1) updated resource estimates (fewer than 500K physical qubits) represent a ~20× reduction over prior work; (2) a zero-knowledge proof methodology for responsible disclosure; (3) identification of five distinct Ethereum vulnerability categories (Account, Admin, Code, Consensus, Data Availability); (4) a "digital salvage" policy framework for dormant assets.

The paper recommends two contingency scenarios: (1) **Fast-clock first** — on-spend and at-rest attacks arrive simultaneously; and (2) **Slow-clock first** — at-rest attacks precede on-spend by years. This IIP's phase plan (§6) and milestone triggers (§9) are designed to remain safe under either ordering.

### 3.3 ZKP-Augmented Ethereum Transaction Migration (pQCee/IoTeX)

Researchers from [IoTeX](https://www.iotex.io/) and [pQCee](https://www.pqcee.com/) propose [two migration paths](https://link.springer.com/chapter/10.1007/978-3-031-77095-1_1) for Ethereum-compatible chains:

* **Proposal 1 (Layer-1 Hard Fork):** a new transaction type augments legacy transactions with a `proofUri` parameter linking to an off-chain quantum-safe zero-knowledge proof (zkSTARK or MPCitH). The proof demonstrates that the sender knows a secret (e.g., a mnemonic phrase) that deterministically generates the transaction's signing address. Validators verify both the ECDSA signature and the PQ ZKP.
* **Proposal 2 (Layer-2 zk-Rollups):** off-chain rollup nodes aggregate individual zkSTARK proofs into a single recursive proof via a tree-based aggregation process, submitting the batch with a final compressed proof via blob transactions ([EIP-4844](https://eips.ethereum.org/EIPS/eip-4844)).
* **Initial benchmarks on Azure F32s v2:** individual proof generation ~304 s with ~42 GB memory for SP1; proof size ~5.3 MB at 100-bit security. With recursive aggregation of 10 proofs via SP1, per-proof size drops to ~177 KB and per-proof time to ~80 s.

### 3.4 Dual Mode Signatures for EdDSA Chains (Mysten Labs/Sui)

[This work](https://eprint.iacr.org/2025/1368) formalizes the "Dual Mode Signature" (DMS) security model and demonstrates a structural advantage of EdDSA-based chains (Sui, Solana, Near). EdDSA's deterministic seed-based key derivation (RFC 8032) allows the seed to serve as a witness in a PQ-NIZK proof, enabling post-quantum authentication without address changes or asset transfers, even for accounts with already-exposed public keys.

The core protocol: the EdDSA seed (sk₂) is a private witness in a STARK proof that (1) `pk = HashToScalar(SHA-512(seed)[:32]) · G`, and (2) `hx = Hash(msg, seed, rx)`. This "one-time proof certification" binds a PQ public key to the legacy address, after which all subsequent signatures use standard PQ verification. Benchmarks (MacBook Pro M4): 6.2 s proof generation, 2.3 s verification, 5.4 MB proof size using the Ligetron zkVM.

The authors show that ECDSA-based chains (Bitcoin, Ethereum) lack this structural property because ECDSA keys are not sampled with deterministic seed structure. BIP-32 hardened derivation is a partial workaround but introduces domain-separation issues and does not achieve quantum safety without modifying BIP-32 itself. **Relevant to IoTeX:** because IoTeX is secp256k1/ECDSA, the seed-based DMS route is not available; the mnemonic-preimage circuit of §4.9 is the analogous (but weaker) construction and is labeled accordingly.

### 3.5 zkSTARK-Based Bitcoin/Ethereum Address Migration

[Kiraz and Kardas](https://eprint.iacr.org/2026/352) present an end-to-end migration framework with dual paths based on key exposure status:

* **Scenario A (Revealed Public Keys):** a hybrid dual-signature migration that combines ECDSA attestation of the PQ key with ML-DSA counter-attestation, followed by a one-way transition to PQ-only authorization.
* **Scenario B (Hidden Public Keys):** zkSTARK-based migration proofs that bind legacy addresses to PQ keys without revealing the legacy EC public key on-chain. The STARK circuit verifies (1) address derivation (`RIPEMD-160(SHA-256(Q)) = h160` for Bitcoin; `Keccak-256(Q[1:65])[12:32] = aeth` for Ethereum) and (2) ECDSA ownership proof.

Both are implementable via two new on-chain primitives: `OP_CHECKQUANTUMSIG` and `OP_CHECKSTARKPROOF`.

> **T-break caveat (this revision, per §2.2).** Kiraz–Kardas Scenario A is **T-governance-only**. Its ownership predicate ("an ECDSA signature by the exposed key") is satisfiable by any CRQC that knows the exposed `sk`. This IIP therefore adopts Scenario A exclusively for the pre-break migration window and adopts Scenario B plus iPACT (§4.10) as the T-break-sound paths.

### 3.6 BIP-361 — Post-Quantum Migration and Legacy Signature Sunset (Bitcoin)

[BIP-361](https://github.com/bitcoin/bips/blob/master/bip-0361.mediawiki) introduces a pre-announced sunset of ECDSA/Schnorr signatures in two phases:

* **Phase A (160,000 blocks ≈ 3 years after activation):** disallows sending funds to quantum-vulnerable address types, forcing adoption of PQ address types.
* **Phase B (2 years after Phase A):** restricts ECDSA/Schnorr spends by encumbering them with a quantum-safe rescue protocol (e.g., proving knowledge of parent XPriv via zkSTARK for BIP-32 HD wallets).

The proposal estimates that over 34% of Bitcoin's supply has exposed public keys.

### 3.7 Chaincode Labs — Bitcoin Post-Quantum Report

The [Chaincode report](https://chaincode.com/bitcoin-post-quantum.pdf) proposes a dual-track strategy: short-term contingency measures (~2 years) deployable as emergency response, and a long-term comprehensive path (~7 years). Key elements include Lamport signatures via `OP_CAT`, quantum-secure Taproot scripts, pay-to-quantum-resistant-hash (P2QRH), and commit-delay-reveal migration mechanisms. The report estimates ~6.26 million BTC (~US$650 billion) as vulnerable and frames the "burn vs. steal" dilemma for dormant assets.

### 3.8 Paradigm — Provable Address-Control Timestamps (PACTs)

[Paradigm](https://www.paradigm.xyz/2026/05/pacts-protecting-your-bitcoin-from-a-quantum-sunset) proposed Provable Address-Control Timestamps (PACTs), a privacy-preserving rescue mechanism for dormant Bitcoin holders. PACTs allow holders to secretly timestamp their knowledge of private keys using Bitcoin itself (via OpenTimestamps and `OP_RETURN` outputs) before CRQCs arrive. A future sunset fork could accept STARK proofs of these timestamps as an alternative spending path, enabling dormant holders to prove ownership without publicly moving coins. The protocol exploits the temporal asymmetry between current private-key knowledge and future quantum key derivation, creating unforgeable evidence that a quantum attacker fundamentally cannot produce. **This is the only mechanism in this survey that is T-break-sound for already-exposed keys** (it requires an off-chain secret, the salt, that a CRQC cannot derive from the public key). This mechanism is adapted for IoTeX as iPACT (§4.10).

### 3.9 Quantum Resistant Ledger (QRL)

[QRL](https://www.theqrl.org/) is a production blockchain that launched (2018) with NIST-approved post-quantum signature [XMSS](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-208.pdf), a stateful hash-based scheme. It is now migrating to [SLH-DSA](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.205.pdf) (FIPS 205, stateless) to eliminate XMSS state-management complexity. [Project Zond](https://qrl-zond.com/) introduces Proof-of-Stake consensus with EVM compatibility. QRL is the existence proof that post-quantum blockchain operation is practical, with several years of uninterrupted PQ-secured operation.

### 3.10 Coinbase Independent Advisory Board

The [position paper](https://assets.ctfassets.net/sygt3q11s4a9/6EjYavuGdtJDYCqaJrASj9/9f464a8bf26f44bd6c85710fe7e4a29f/Quantum_Computing_and_Blockchain_v10.3_15April2026.pdf) from the Coinbase Independent Advisory Board provides the most authoritative current threat assessment. The board concludes that large-scale fault-tolerant quantum computers are "not imminent but could arrive by 2035+." They define four quantum milestones: (1) fault-tolerant two-qubit gates; (2) fault-tolerant Shor's factoring; (3) indefinitely stable logical qubits; and (4) verifiable quantum-simulation advantage. The paper rates PQC signature schemes by confidence and size: ML-DSA-44 (1,312-byte pubkey, 2,420-byte sig), FN-DSA-512 (897-byte pubkey, 666-byte sig, 3× signing cost), SLH-DSA-128s (32-byte pubkey, 7,856-byte sig, 14,000× signing), and SLH-DSA-128f (32-byte pubkey, 17,088-byte sig, 720× signing). Confidence is highest for hash-based SLH-DSA; aggregation remains research-intensive.

For consensus, the paper recommends periodic PQ checkpoints: a super-majority of delegates signs block metadata (`block_hash`, `state_root`, `tx_root`, etc.) with ML-DSA-65 signatures (~3,309 bytes each), providing PQ integrity without full execution-layer migration. IoTeX's 24-delegate Roll-DPoS can adopt full PQ signing per block, eliminating the need for aggregation optimizations. The paper also urges early governance decisions on dormant wallets and recommends public decisions on freeze, burn, or other policies before they become market-impacting. **This IIP adopts the four milestones as the trigger basis for §9.**

### 3.11 Meta

Meta's [engineering blog](https://engineering.fb.com/2026/04/16/security/post-quantum-cryptography-migration-at-meta-framework-lessons-and-takeaways/) outlines a six-step PQC migration strategy: (1) prioritize high-risk applications (offline-attack vectors first); (2) build a cryptographic inventory through automated discovery and developer reporting; (3) address external dependencies (NIST standards, hardware vendor support, production implementations); (4) select algorithms (ML-KEM, ML-DSA, HQC); (5) implement guardrails to block new quantum-vulnerable keys and APIs; and (6) integrate PQC components via replacement or hybrid schemes. Meta introduces PQC Migration Levels — PQ-Unaware, PQ-Aware, PQ-Ready, PQ-Hardened, and PQ-Enabled. This framework is the organizational maturity model adopted (and concretized for IoTeX) in §7.

### 3.12 Key Insights for IoTeX

* Ethereum's account-abstraction approach is directly applicable to IoTeX given EVM compatibility. IoTeX's Roll-DPoS consensus with 24 delegates is simpler than Ethereum's PoS, dramatically simplifying consensus-layer PQ migration.
* IoTeX's 5-second block time makes on-spend attacks more challenging but not impossible (an attacker could collude with or replace delegates). The DePIN use case amplifies at-rest risk because device keys are long-lived. Google's dual-scenario planning framework directly informs the phased migration.
* IoTeX uses secp256k1/ECDSA, not EdDSA, so the seed-based DMS approach is not directly applicable. The mnemonic-based approach (proving knowledge of a BIP-39 mnemonic) remains viable for HD wallets following the IoTeX/pQCee construction. For non-mnemonic accounts (e.g., HSM-based institutional custody), the Kiraz–Kardas dual-path framework is more appropriate — subject to the T-break caveat in §3.5.
* The dual-path approach of Kiraz and Kardas is directly applicable: IoTeX accounts that have transacted (revealed keys) follow Scenario A; accounts that have only received (hidden keys) follow Scenario B. The Ethereum-specific STARK circuit design can be adopted with minimal modification for IoTeX's Keccak-256 address derivation.
* The "private incentive" model in BIP-361 (a clear deadline to motivate migration) is applicable to IoTeX but must be calibrated to IoTeX's governance model (24 delegates, community voting) and the DePIN device lifecycle.
* **New in this revision:** the EF's hash-based lean path, Drake's hash-only argument, and Buterin's lattice-parameter warning all point the same way — treat the hash-based option as co-primary, and treat lattice parameter sizes as provisional and revisable. This is reflected in §4.1, §4.11, and §4.13.

### 3.13 Comparative Summary

| **Dimension** | **Ethereum (EF)** | **Bitcoin (BIP-361)** | **IoTeX & pQCee** | **Sui** | **Kiraz-Kardas** | **QRL** | **This IIP** |
| --- | --- | --- | --- | --- | --- | --- | --- |
| **Signature Scheme** | leanXMSS/WOTS (lean) + Falcon/ML-DSA/SLH-DSA (evaluating) | ML-DSA (proposed) | Scheme-agnostic (ZKP-based) | EdDSA DMS + PQ-NIZK | ML-DSA + zkSTARK | XMSS → SLH-DSA | ML-DSA-65 + SLH-DSA (co-primary) + zkSTARK |
| **Security Level Target** | Not stated | Not stated | Not stated | Not stated | Not stated | Level 1–5 (per param) | **≥ Category 3 for long-lived keys** |
| **Migration Model** | Account abstraction (opt-in) | Flag-day sunset (forced) | New tx type + off-chain proof | Seed-based DMS (opt-in) | Dual-path (revealed/hidden) | Hard fork (PQ-native) | Phased hybrid (incentivized) + bunker track |
| **Consensus Layer** | leanXMSS + SNARK aggregation | N/A (PoW unchanged) | Not addressed | Not addressed | Not addressed | XMSS/SLH-DSA native | ML-DSA-87 delegate keys + registry |
| **Backward Compatibility** | High (AA-based) | Moderate (soft fork) | High (new tx type) | High (same address) | High (one-way transition) | Low (new chain) | High (phased, incentivized) |
| **Aggregation** | SNARK-based (leanVM) | N/A | Recursive zkSTARK | One-time proof | Not addressed | N/A | Recursive zkSTARK + precompile |
| **ZK Proof System** | General-purpose | zkSTARK (proposed) | SP1/RISC0 zkVM | Ligetron zkVM | SP1 zkVM (STARK-native) | N/A | SP1 zkVM (STARK-native) |
| **IoT/DePIN** | Not addressed | Not addressed | Partially (IoTeX context) | Not addressed | Not addressed | Not addressed | Fully addressed |
| **T-break Rescue** | Not addressed | Rescue unclear | Not addressed | Not addressed | Scenario B only | N/A | iPACT + Scenario B (explicitly labeled) |
| **Timeline** | L1 by 2029 & full migration 2032+ | ~5 years post-activation | Not specified | Not specified | Not specified | Already deployed | 2026–2033 (7-year plan) |

## 4. Specification

### 4.1 Post-Quantum Signature Algorithms

#### 4.1.1 Security Target

This IIP adopts an explicit, normative security target, rather than selecting parameter sets ad hoc:

* **Long-lived keys** (consensus/delegate keys, smart-contract admin keys, DePIN device identities) MUST achieve **NIST Security Category 3** or higher.
* **User-transaction keys** MUST achieve **Category 3** by default. A Category-2 construction MAY be used only for explicitly bounded, low-value, or compatibility contexts, and MUST be identified as Category 2 in all wallet and RPC metadata.
* Because Grover's algorithm reduces the effective key-search security of a Category-1 assumption to a level comparable to ~2^64 quantum work (§2.5), a Category-1 construction MUST NOT be described as delivering 128-bit post-quantum security, and MUST NOT be used for long-lived keys.

This floor is set deliberately above the minimum available, because (a) the horizon is a decade-plus, (b) lattice parameter sizes are provisionally revisable (§4.11), and (c) signature-size headroom is cheaper than a second migration.

#### 4.1.2 Supported Algorithms

* **ML-DSA (FIPS 204, Module-Lattice):** the **primary** PQ signature scheme.
  * **ML-DSA-65 (Category 3)** — the default for user transactions. Public key 1,952 B; signature 3,309 B.
  * **ML-DSA-87 (Category 5)** — REQUIRED for consensus/delegate keys and RECOMMENDED for high-value smart contract admin keys. Public key 2,592 B; signature 4,627 B.
  * **ML-DSA-44 (Category 2)** — MAY be used only for bounded/low-value contexts or explicit compatibility, and MUST be labeled Category 2. Public key 1,312 B; signature 2,420 B.
* **SLH-DSA (FIPS 205, Stateless Hash-Based):** the **co-primary**, hash-only scheme. SLH-DSA relies solely on hash-function security and is offered as an equal-status option, not a fallback, so deployers may avoid lattice assumptions entirely.
  * **SLH-DSA-SHA2-192s (Category 3)** — REQUIRED option for long-lived DePIN device identities and RECOMMENDED where signature size is tolerable. Public key 48 B; signature 16,224 B.
  * **SLH-DSA-SHA2-256s (Category 5)** — OPTIONAL for the most conservative long-horizon identities. Public key 64 B; signature 29,792 B.
  * **SLH-DSA-SHA2-128s (Category 1)** — permitted only for short-lived or size-constrained contexts; MUST be labeled Category 1.

> **Correction from Revision 1.** SLH-DSA-128s was described as "the most conservative security assumption." That was a category error: for a conservative posture the correct choice is a *larger* parameter set (192s/256s), not the smallest. Category 1 is now explicitly excluded from long-lived keys.

**Excluded or reserved algorithms:**

* **FN-DSA / FALCON (FIPS 206 draft):** **NOT active in this IIP.** It requires complex floating-point arithmetic and Gaussian sampling, which is unsuitable for IoT devices. The identifier `0x80` is **reserved** to prevent future collision; it MUST NOT be accepted by the precompile until a future IIP activates it.
* **SQIsign:** compact signatures but immature and computationally expensive; not supported.
* **XMSS (stateful):** battle-tested (QRL) but operationally complex for a general-purpose chain; not supported as a primary scheme.

#### 4.1.3 Canonical Algorithm-Identifier Table

This table is the **single source of truth** for algorithm identifiers and sizes. Any other section of this document (including inline comments in §4.3) MUST agree with it; in case of conflict, this table governs.

| **Algorithm** | **Identifier (uint8)** | **NIST Category** | **Public key (B)** | **Signature (B)** | **OID** | **Status** |
| --- | --- | --- | --- | --- | --- | --- |
| ML-DSA-44 | 0x01 | 2 | 1,312 | 2,420 | 2.16.840.1.101.3.4.3.17 | Active (bounded/low-value only) |
| ML-DSA-65 | 0x02 | 3 | 1,952 | 3,309 | 2.16.840.1.101.3.4.3.18 | Active (default) |
| ML-DSA-87 | 0x03 | 5 | 2,592 | 4,627 | 2.16.840.1.101.3.4.3.19 | Active (consensus/admin) |
| SLH-DSA-SHA2-128s | 0x10 | 1 | 32 | 7,856 | 2.16.840.1.101.3.4.3.20 | Active (short-lived only) |
| SLH-DSA-SHA2-128f | 0x11 | 1 | 32 | 17,088 | 2.16.840.1.101.3.4.3.21 | Active (short-lived only) |
| SLH-DSA-SHA2-192s | 0x12 | 3 | 48 | 16,224 | 2.16.840.1.101.3.4.3.22 | Active (device identity) |
| SLH-DSA-SHA2-192f | 0x13 | 3 | 48 | 35,664 | 2.16.840.1.101.3.4.3.23 | Active (device identity) |
| SLH-DSA-SHA2-256s | 0x14 | 5 | 64 | 29,792 | 2.16.840.1.101.3.4.3.24 | Active (optional, max) |
| SLH-DSA-SHA2-256f | 0x15 | 5 | 64 | 49,856 | 2.16.840.1.101.3.4.3.25 | Active (optional, max) |
| FN-DSA-512 | 0x80 | (pending FIPS 206) | 897 | 666 | (pending) | **Reserved — not active** |

> ML-DSA-65 signature size is **3,309 bytes** (FIPS 204 Table 2), correcting the "3,293 bytes" figure used in Revision 1.

### 4.2 Domain Separation and Signing Preimages

Every PQ signature in this IIP is computed over an explicitly defined, domain-separated preimage. ML-DSA is invoked with a FIPS 204 *context string* (`ctx`) that binds the purpose of the signature; independent of `ctx`, the signed message binds the chain and the object being authorized. Concretely:

* **Transaction signing (Type `0x05`).** Let

  ```
  DOMAIN_TX  = "IoTeX-PQ-Tx-v1"
  unsigned   = rlp([chain_id, nonce, max_priority_fee_per_gas, max_fee_per_gas,
                    gas_limit, to, value, data, access_list,
                    pq_algorithm_id, pq_public_key])
  pq_signing_hash = Keccak256( bytes(DOMAIN_TX) || unsigned )
  ```

  The PQ signature MUST verify over `pq_signing_hash` with `ctx = DOMAIN_TX`. Because `chain_id` is inside `unsigned`, a signature is not replayable on another chain; because `DOMAIN_TX` prefixes the digest, it is not replayable as a registry or consensus signature.
* **Registry registration.** With `registry_address` the deployed PQ Key Registry address and `registration_nonce` a per-address monotonically increasing counter,

  ```
  DOMAIN_REG = "IoTeX-PQ-Register-v1"
  registration_hash = Keccak256( bytes(DOMAIN_REG) || to_bytes(chain_id, 32) ||
                                 registry_address || sender_address || algorithm_id ||
                                 keccak256(pq_public_key) || registration_nonce )
  ```

  Both the PQ signature(s) and (where applicable) the ECDSA signature MUST verify over `registration_hash`. `registration_nonce` provides replay protection across the two registration paths.
* **Consensus signing (delegate proposals and attestations).** With `bp` the block-proposal/attestation object,

  ```
  DOMAIN_CONS = "IoTeX-PQ-Consensus-v1"
  consensus_digest = Keccak256( bytes(DOMAIN_CONS) || to_bytes(chain_id, 32) ||
                                to_bytes(height, 8) || block_hash || state_root ||
                                tx_root || epoch || validator_set_root )
  ```

* **Generic precompile use (dapps).** The precompile (§4.3) performs **no implicit domain separation**: callers MUST pass a message whose *first* bytes are a caller-chosen domain tag, and the precompile MUST reject a message shorter than the tag plus a chain-id field. The protocol's own uses above are the reference patterns.

All multi-byte integers in this section are big-endian unless stated otherwise. All `rlp` encodings follow the standard Ethereum RLP rules.

### 4.3 PQ Signature Verification Precompile

A precompiled contract is deployed at address `0x000000000000000000000000000000000000000B` (shorthand `0x0B`). It exposes generic PQ signature verification:

```
// Input (all integers big-endian):
//   [0]      version        (1 byte, MUST be 0x01)
//   [1]      algorithm_id   (1 byte; see §4.1.3 — the canonical table)
//   [2:4]    pk_len         (2 bytes)
//   [4:4+pk_len]      public_key
//   [.. : ..+2]       msg_len (2 bytes)
//   [.. : ..+msg_len] message
//   [.. : ..+2]       sig_len (2 bytes)
//   [.. : ..+sig_len] signature
//
// Output: a 32-byte word, 0x...01 if the signature is valid, 0x...00 otherwise.
//
// Error handling: malformed framing, an unknown/!active algorithm_id, a pk_len or
// sig_len inconsistent with the declared algorithm, or a msg_len of zero MUST cause
// a REVERT (not a 0x00 result). A well-formed input with a failing signature returns 0x00.
```

The input is length-prefixed so it is unambiguously parseable for any algorithm (including the variable-size SLH-DSA `s`/`f` variants). The precompile MUST reject `0x80` (reserved FN-DSA) and any algorithm_id not marked "Active" in §4.1.3.

**Domain separation.** As specified in §4.2, the precompile itself does not add a domain tag; the protocol-level callers bind `DOMAIN_TX` / `DOMAIN_REG` / `DOMAIN_CONS` and `chain_id` inside `message`, and dapp callers MUST follow the pattern in §4.2.

**Provisional gas schedule.** Verification gas is benchmarked against `ecrecover` (3,000 gas). The following values are **provisional defaults** and are final unless changed by a future IIP; they MUST be revisited once client benchmarks are published (see §16, Test Vectors, and §18, Open Questions).

| Algorithm | Relative verify cost vs `ecrecover` | Provisional gas |
| --- | --- | --- |
| ML-DSA-44 (0x01) | ~0.9× | 2,700 |
| ML-DSA-65 (0x02) | ~1.5× | 4,500 |
| ML-DSA-87 (0x03) | ~2.0× | 6,000 |
| SLH-DSA-SHA2-128s (0x10) | ~37× | 111,000 |
| SLH-DSA-SHA2-192s (0x12) | ~85× | 255,000 |
| SLH-DSA-SHA2-256s (0x14) | ~140× | 420,000 |

> **Correction from Revision 1.** The same operation was described as both "0.5× (faster)" and "0.9× the cost of ECDSA." Only the 0.9× figure is retained, since it is the one from which the gas schedule is derived. A single benchmark methodology (hardware, library version, batch size) MUST be cited when the final numbers are published.

### 4.4 PQ-Authenticated Transaction (Type `0x05`)

Following [EIP-2718](https://eips.ethereum.org/EIPS/eip-2718) (Typed Transaction Envelope), a new transaction type `0x05` is introduced:

```
0x05 || rlp([chain_id, nonce, max_priority_fee_per_gas, max_fee_per_gas,
             gas_limit, to, value, data, access_list,
             pq_algorithm_id, pq_public_key, pq_signature,
             legacy_v, legacy_r, legacy_s])
```

* During Phase 2 (Hybrid Mode), both `pq_signature` and the legacy ECDSA signature (`legacy_v, legacy_r, legacy_s`) MUST be present and valid.
* During Phase 4 (PQ-Only Mode), the legacy fields MUST be zero-filled and are ignored for sender recovery.

**Validation** proceeds as follows:

1. Compute `pq_signing_hash` per §4.2 and verify `pq_signature` over it with the precompile (§4.3), using `pq_algorithm_id` and `pq_public_key`.
2. Look up `pq_public_key` in the PQ Key Registry (§4.5). The registered entry MUST exist and be `active`.
3. **Sender resolution.**
   * **Hybrid mode (Phase 2):** recover the sender from the legacy ECDSA signature via `ecrecover`; the recovered address MUST equal the registry entry's bound address.
   * **PQ-only mode (Phase 4):** the legacy fields are zero-filled and no `ecrecover` is performed. The sender is defined as `registry[pq_public_key].bound_address`. The registry MUST enforce that a `pq_public_key` binds to **exactly one** address (§4.5, binding invariant), so sender resolution is unambiguous.
4. If all checks pass, the transaction is valid. If the registry shows the sender has an active PQ key and the transaction is a legacy type (`0x00`/`0x01`/`0x02`), validation fails per the Phase 3 rule (§6).

> **Binding invariant (normative).** The registry MUST NOT allow the same `(algorithm_id, pq_public_key)` to be bound to more than one IoTeX address, and MUST NOT allow an address to have more than one active PQ key. Violating this invariant enables address-confusion and replay-style attacks.

### 4.5 PQ Key Registry

A singleton contract, deployed at a protocol-designated address, maintains the mapping between IoTeX addresses and their registered PQ public keys.

```solidity
contract PQKeyRegistry {
    struct PQKey {
        uint8   algorithmId;
        bytes   publicKey;
        uint64  registeredBlock;
        uint64  registrationNonce;
        bool    active;
    }

    mapping(address => PQKey) public registry;
    // Invariant: keccak256(algorithmId || publicKey) => bound address is injective.
    mapping(bytes32 => address) public keyOwner;

    event KeyRegistered(address indexed addr, uint8 algorithmId, bytes32 keyHash, uint64 nonce);
    event KeyRotated(address indexed addr, uint8 algorithmId, bytes32 keyHash, uint64 nonce);
    event PQOnlyActivated(address indexed addr, uint64 blockNumber);

    // Path 1 (T-governance-only): dual-signature registration for
    // accounts whose ECDSA public key is ALREADY exposed. For unexposed accounts
    // this path MUST NOT be used (it de-anonymizes the account; see §2.6, §4.6).
    function registerDualSignature(
        uint8   algorithmId,
        bytes calldata pqPublicKey,
        bytes calldata pqSignature,     // over registration_hash (§4.2)
        bytes calldata ecdsaSignature   // over registration_hash (§4.2)
    ) external;

    // Path 2 (T-break-sound; RECOMMENDED default for unexposed accounts):
    // zkSTARK proof binding the address to the PQ key without revealing the EC key.
    function registerWithProof(
        uint8   algorithmId,
        bytes calldata pqPublicKey,
        bytes calldata starkProof,      // per §4.9
        bytes32 addressHash             // Keccak-256 address binding (public input)
    ) external;

    // Path 3: native-PQ registration for entities that never had secp256k1 keys
    // (e.g., newly manufactured DePIN devices, §4.8). The bound address is derived
    // from the PQ key per the derivation rule below.
    function registerNativePQ(
        uint8   algorithmId,
        bytes calldata pqPublicKey,
        bytes calldata ecdsaAttestation  // OPTIONAL: a manufacturer/attester signature; see §4.8
    ) external;

    // Activation of PQ-only mode for a single account. Caller MUST be the account
    // itself (msg.sender == addr) or an authorized recovery delegate. Reverts if the
    // account has no active registered PQ key. Irreversible for the account, but the
    // contract exposes no permissionless, network-wide flip.
    function activatePQOnly() external;

    // Key rotation: requires a valid signature by the CURRENT registered PQ key.
    function rotateKey(uint8 newAlgorithmId, bytes calldata newPqPublicKey,
                       bytes calldata rotationSignature) external;
}
```

**Native-PQ address derivation (Path 3).** For entities with no prior secp256k1 account, the bound IoTeX address is defined as:

```
native_addr = keccak256( bytes("IoTeX-PQ-Addr-v1") || chain_id || algorithm_id ||
                         keccak256(pq_public_key) )[12:32]
```

This derivation is domain-separated from secp256k1 address derivation, so a native-PQ address can never collide with (or be confused for) a legacy account, and no ECDSA key ever exists for it. `registerNativePQ` MUST reject a `pq_public_key` already bound to another address.

**Notes.**

* The `registration_nonce` in `registration_hash` (§4.2) MUST be the current `registrationNonce` for the address and MUST be incremented on every successful registration/rotation, preventing replay of a registration signature.
* `activatePQOnly` MUST be access-controlled as written (self or recovery delegate) and MUST NOT be callable permissionlessly. An irreversible, permissionless state flip would be a protocol-level hazard.
* The dual-signature path (`registerDualSignature`) follows Kiraz–Kardas Scenario A and is **T-governance-only** (§2.2). The zkSTARK path (`registerWithProof`) follows Scenario B and is the **RECOMMENDED default** for any account whose public key is not already on-chain.
* **De-anonymization warning (normative).** `registerDualSignature` publishes a recoverable ECDSA public key. Using it for a hash-hidden account converts a bunker-protected account into an at-rest-exposed one. Wallets MUST warn and MUST default to `registerWithProof` for unexposed accounts.

### 4.6 Hash-Hidden Account Migration Path

The defensive posture for currently-safe accounts is specified in §2.6. Normatively:

* An account whose secp256k1 public key is not already published on-chain SHOULD register via `registerWithProof` (§4.5, Path 2), which reveals neither the EC public key nor the address's linkage in a way that enables at-rest ECDLP recovery.
* Wallets, exchanges, and the SDK MUST NOT use the dual-signature path by default for such accounts.
* Migration tooling SHOULD surface, per account, whether its public key is already exposed (a "key-exposure status" indicator) and SHOULD recommend the bunker posture of §2.6 for exposed and unexposed accounts alike.

### 4.7 Consensus Layer: Delegate PQ Key Migration

IoTeX's Roll-DPoS consensus with 24 delegates simplifies consensus PQ migration compared to Ethereum's hundreds of thousands of validators:

* **Phase 1 — PQ Key Registry for Delegates.** Each delegate registers a PQ public key in a dedicated delegate registry contract. Registration requires both the delegate's current secp256k1 signing key and the new PQ key, or (for new delegates) the native-PQ path. **ML-DSA-87 (Category 5) is REQUIRED for delegate keys** (§4.1.1).
* **Phase 2 — Dual-Signed Blocks.** Block proposals include both ECDSA and PQ signatures from the proposing delegate; attestations from other delegates include both signature types. Clients MUST verify both. During this phase, block *integrity* is PQ-protected, but delegate *identity* still depends on the ECDSA half until Phase 3.
* **Phase 3 — PQ-Only Consensus.** After a governance-approved activation height, only PQ-signed block proposals and attestations are accepted; ECDSA-only delegates are excluded from the active set.

**Bandwidth and storage sizing (new in this revision).** With 24 delegates at ML-DSA-87:

```
per-block attestation payload  = 24 × 4,627 B      = 111,048 B ≈ 108 KiB
per-day   (5 s blocks, 17,280) = 111,048 × 17,280  ≈ 1.92 GB/day
per-year                            ≈ 700 GB/year (attestations only)
```

A delegate that cannot yet run ML-DSA-87 and falls back to SLH-DSA-192s (16,224 B) yields 24 × 16,224 = 389,376 B/block ≈ **6.7 GB/day**, which is why the IIP (a) RECOMMENDS ML-DSA for the consensus hot path, and (b) treats aggregation as an optimization to be introduced in Phase 2 rather than a prerequisite. Node operators MUST plan storage and archival policy (pruned attestation retention, light-client attestation) accordingly.

With only 24 delegates, native BLS-style aggregation is not required. Individual ML-DSA-87 signatures cost ~108 KiB/block, manageable within IoTeX's block capacity. For future scalability (e.g., larger delegate sets), STARK-based aggregation following the EF's leanMultisig approach MAY be adopted.

### 4.8 DePIN Device Identity Migration

A specialized migration path for IoT device identities in the DePIN ecosystem:

* **Device PQ Key Provisioning.** New devices SHOULD be provisioned with PQ key pairs at manufacture, using **SLH-DSA-SHA2-192s (Category 3, hash-only)** where signature size is tolerable, or ML-DSA-65 otherwise. Devices MUST be registered via `registerNativePQ` (§4.5, Path 3); they have no secp256k1 key and MUST NOT be required to produce an ECDSA signature to bootstrap.
* **Legacy Device Migration.** Existing devices with secp256k1 keys register PQ keys via a TEE-assisted proving service. The device authenticates to the TEE via its legacy key, the TEE generates a PQ key pair and a zkSTARK migration proof, and the proof is submitted to the PQ Key Registry on behalf of the device. Because the legacy key is used here, this path is **T-governance-only** (§2.2); devices SHOULD complete migration and iPACT commitment (§4.10) as early as practical.
* **Lightweight Verification.** Resource-constrained IoT devices that cannot perform PQ signature verification locally MAY delegate verification to trusted gateway nodes holding PQ-verified certificates, following the IoTeX W3bstream off-chain compute model.

**Application-layer credential carve-out (new in this revision).** The precompile (`0x0B`) and the PQ Key Registry are **general-purpose primitives**. A downstream protocol MAY consume them for **application-layer credential issuance, verification, and renewal independently of account-level enforcement timing**. Specifically, a DePIN credential layer (device-bound attestations, verified-hardware proofs, presence/challenge-response credentials) MAY be PQ-deployable on the Phase 1 timeline without waiting for Phase 2 hybrid enforcement on user accounts. This IIP imposes no account-level dependency on such application-layer use; the registry's `registerNativePQ` path exists precisely to give such credentials a first-class, non-ECDSA anchor.

### 4.9 zkSTARK Proof System for Migration

Following the IoTeX/pQCee and Kiraz–Kardas frameworks, the migration proof system uses SP1 (Succinct's STARK-native zkVM) to generate proofs that remain entirely within the STARK domain (avoiding a quantum-vulnerable Groth16 final recursion step).

**Path A — Address-binding circuit (T-governance-only).** Adapted from Kiraz–Kardas §6.2:

```
Public inputs:  IoTeX address a_iotex (20 bytes), PQ public key pk_pq
Private inputs: Uncompressed secp256k1 public key Q_u (65 bytes), ECDSA signature σ
Constraints:
  1. Address binding: Keccak-256(Q_u[1:65])[12:32] == a_iotex
  2. Ownership proof: ECDSA.Verify(Q_u, Keccak-256(a_iotex || pk_pq), σ) == 1
```

> **T-break caveat (normative).** Constraint 2 is satisfiable by anyone who knows the exposed private key. After T-break, a CRQC can forge σ and bind its own `pk_pq` to the address. This circuit MUST be labeled **T-governance-only** and MUST NOT be relied upon to recover funds after T-break.

**Path B — Mnemonic/seed-preimage circuit (T-break-sound *only if the witness is a true preimage*).** For wallets using BIP-39/BIP-32 HD derivation:

```
Public inputs:  IoTeX address a_iotex, PQ public key pk_pq
Private inputs: ρ = the mnemonic-derived entropy/pre-image (NOT sk)
Constraints:
  1. sk = KDF(ρ)             (e.g., PBKDF2/BIP-32 derivation from ρ)
  2. sk · G = Q              (secp256k1 point multiplication)
  3. Keccak-256(Q_uncompressed[1:65])[12:32] == a_iotex
```

> **Security-critical:** the witness MUST be `ρ`, the value from which `sk` is *derived*, and MUST NOT be `sk` itself. If `sk` is the witness, the circuit is T-governance-only (a CRQC derives `sk` and satisfies it). Implementers MUST verify that the derivation is not invertible in the circuit and MUST NOT accept any structure in which `sk` alone suffices. It is RECOMMENDED that this circuit be restricted to derivation schemes (e.g., hardened BIP-32) whose preimage remains hidden after `sk` is known, and this SHOULD be documented per wallet type.

Multiple migration proofs MAY be aggregated via tree-based recursive STARK aggregation, reducing on-chain verification cost. A batch of N proofs aggregates into a single proof of approximately constant size (~1.8 MB for SP1 compressed). A STARK verification contract is deployed on IoTeX, accepting the proof, public inputs, and verification key and returning a boolean. Verification gas is provisionally ~200,000–500,000 gas depending on proof parameters (§4.3 and §13).

### 4.10 iPACT: IoTeX Provable Address-Control Timestamps

Adapted from [Paradigm's PACTs proposal](https://www.paradigm.xyz/2026/05/pacts-protecting-your-bitcoin-from-a-quantum-sunset), iPACTs allow holders of quantum-vulnerable IoTeX addresses to silently and cheaply prove, **before** CRQCs arrive, that they control their private keys. Unlike the T-governance-only paths of §4.9, an iPACT is **T-break-sound**: a CRQC that later derives `sk` cannot reconstruct the holder's off-chain `salt`, and cannot backdate a commitment.

> **Timing is the entire value.** An iPACT is worth exactly as much as its timestamp predates T-break. This revision therefore moves iPACT generation into **Phase 0** (§6.1) and requires the cutoff to be set as early as participation allows.

#### 4.10.1 Commitment Generation

The holder generates:

```
salt          = random(32 bytes)
msg           = Encode("iPACT/v1", "iotex-mainnet", ioAddr, salt)
control_proof = ECDSA.sign(secp256k1_privkey, keccak256(msg))
commitment    = keccak256("iPACT/v1 commitment" || salt || keccak256(control_proof))
```

`ioAddr` is the IoTeX address (20 bytes, derived from `keccak256(secp256k1_pubkey)[12:32]`), and `Encode` is a canonical ABI-encoding function. The holder stores the salt, the ECDSA control proof, and the timestamp proof in a secure location. This requires no on-chain transaction when using Method A and reveals nothing about the control proof, salt, public key, address, or which coins the holder owns.

#### 4.10.2 Timestamping Methods

* **Method A: OpenTimestamps via Bitcoin.** The commitment hash is submitted to the OpenTimestamps calendar server, which aggregates it into a Merkle tree and embeds the root in a Bitcoin `OP_RETURN` output. Strongest timestamp guarantee; the holder retains the `.ots` proof file; no IoTeX transaction required.
* **Method B: IoTeX On-Chain Commitment.** The commitment hash is submitted to a `PACTRegistry` contract on IoTeX, storing `mapping(bytes32 => uint64)` from commitment to block number. IoTeX-native timestamping; requires one transaction (~41,000 gas); reveals nothing (the commitment is opaque).
* **Method C: Ethereum On-Chain Commitment.** The commitment hash is submitted to an Ethereum contract, providing an independent timestamp anchored to Ethereum's security.

Holders SHOULD use multiple methods for redundancy. The rescue protocol accepts any method whose timestamp predates the governance-defined cutoff.

```solidity
contract PACTRegistry {
    mapping(bytes32 => uint64) public commitments;
    event CommitmentRegistered(bytes32 indexed commitment, uint64 blockNumber);
    function register(bytes32 commitment) external {
        require(commitments[commitment] == 0, "Already registered");
        commitments[commitment] = uint64(block.number);
        emit CommitmentRegistered(commitment, uint64(block.number));
    }
    function getTimestamp(bytes32 commitment) external view returns (uint64) { return commitments[commitment]; }
}
```

#### 4.10.3 Cutoff Date

The governance-defined cutoff is the timestamp before which an iPACT MUST have been created to be accepted. It MUST be set conservatively early — before any credible evidence that a CRQC could derive secp256k1 keys — and is a Phase 0/Phase 2 governance decision. The recommended default is the earliest block at which commitments can be reliably timestamped (i.e., Phase 0 activation), **not** Phase 2 activation as in Revision 1.

#### 4.10.4 iPACT Rescue Protocol

Upon activation of the legacy sunset (Phase 3), a `PACTRescue` contract is deployed alongside the `PQRescue` contract. To rescue funds, the claimant submits a STARK proof of the statement:

> "I know values `salt`, `control_proof`, and `secp256k1_privkey` such that: (1) `pk = secp256k1.pubkey(secp256k1_privkey)`; (2) `ioAddr = keccak256(pk)[12:32]`; (3) `ioAddr == claimed_address`; (4) `msg = Encode("iPACT/v1", "iotex-mainnet", ioAddr, salt)`; (5) `ECDSA.verify(pk, keccak256(msg), control_proof) == true`; (6) `commitment = keccak256("iPACT/v1 commitment" || salt || keccak256(control_proof))`; (7) `commitment` was timestamped before `cutoff_date`; and (8) `pq_destination` is the post-quantum address to which rescued funds are sent."

Public inputs: `(claimed_address, commitment, timestamp_proof_root, cutoff_block, pq_destination, pq_algorithm_id)`. Private inputs: `(secp256k1_privkey, pk, salt, control_proof)`. The STARK proof is post-quantum secure. On success, `PACTRescue` transfers the balance to `pq_destination`, which MUST have a registered PQ key. The rescue transaction is bound to the commitment and MUST be single-use (see §4.10.6).

#### 4.10.5 iPACT-DePIN Extension

For DePIN devices, an extended format is defined:

```
msg = Encode("iPACT-DePIN/v1", "iotex-mainnet", deviceAddr, deviceType, salt)
```

where `deviceType` is a standardized identifier (e.g., `"obd-dongle"`, `"pebble-tracker"`). Device manufacturers SHOULD generate iPACTs for all deployed devices during provisioning and MAY refresh them during firmware updates, storing salts and proof files in secure enclaves or manufacturer KMS.

#### 4.10.6 Refresh, Replay, and Renewal Semantics (new in this revision)

To make the iPACT-DePIN extension buildable by downstream protocols, the following semantics are normative:

* **Refreshable, not one-time.** A device or holder MAY publish multiple iPACT commitments over its lifetime (e.g., on each firmware update or re-attestation). Each commitment incorporates a distinct `salt` and a `nonce`.
* **Earliest-valid rule.** The rescue predicate accepts the **earliest** commitment whose timestamp is strictly before the cutoff, and rejects any commitment at or after the cutoff. Refreshing after the cutoff produces a commitment that cannot be used for rescue (it is evidence of continuity, not entitlement).
* **Single-use and monotonic binding.** Each commitment is single-use: once redeemed, it MUST be marked consumed on-chain and MUST NOT authorize a second transfer. Commitment preimages MUST be domain-separated by an increasing `nonce` so that publishing a refresh cannot be linked to, or substitute for, an earlier entitlement.
* **Interaction with hardware re-attestation.** Where a device periodically re-attests (verified-hardware-proof renewal), the renewal is independent of the iPACT cadence: the iPACT proves *control of the legacy key at time T*, while hardware re-attestation proves *the device is genuine at time T'*. The two MAY share a provisioning channel but MUST NOT share a preimage; renewal MUST NOT invalidate an earlier, pre-cutoff iPACT.
* **DePIN refresh MUST be derivable from the secure element** and MUST NOT require the legacy private key to leave the device.

### 4.11 Cryptographic Agility and Re-parameterization

Cryptographic agility is a first-class design principle. The precompile supports multiple algorithm identifiers and the PQ Key Registry supports rotation. If an algorithm is weakened — as happened with isogeny-based SIKE in 2022, and as Buterin warns could happen to lattices — accounts can rotate to another active algorithm without protocol changes.

Agility is given teeth by an explicit **re-parameterization procedure**:

* **Trigger.** A re-parameterization IIP MUST be opened within 90 days if either (a) published cryptanalysis reduces the concrete core-SVP (or equivalent) cost of the deployed parameter set by more than a threshold set by governance (proposed: 32 bits), or (b) NIST issues a revised or superseding specification for a deployed algorithm. The threshold is a governance parameter, not a protocol constant.
* **Owner.** The 24 delegates MAY propose, but re-parameterization for a *deployed* algorithm REQUIRES a community governance vote (§14); the cryptographic review committee (§7) MUST publish an impact assessment first.
* **Dry-run.** The network MUST exercise at least one non-critical algorithm-identifier rotation on testnet before mainnet PQ-only mode activates. Agility that has never been exercised is not agility.
* **Substitution policy.** In line with §1.5, if concern about lattice assumptions grows, the RECOMMENDED response is (1) increase ML-DSA parameter sizes within the existing precompile, and only then (2) migrate signatures to SLH-DSA; and for hash functions, (1) increase round counts before (2) changing output byte-widths.

### 4.12 Multisig and Account Abstraction

Contract accounts (Gnosis-Safe-style multisigs, ERC-4337 accounts) hold a substantial share of value and are **not** automatically covered by EOA-key migration. This IIP therefore specifies:

* **Protocol PQ module.** A reference, protocol-endorsed PQ verifier module (`PQVerifierLib.sol`) and a multisig/AA migration contract SHOULD be provided as public goods (§17), letting existing contract accounts add a PQ spending path without redeployment where possible, or via a defined migration transaction where not.
* **Off-chain confirmations.** Multisig confirmations SHOULD be collected off-chain (signed messages, not on-chain confirmation transactions) wherever the wallet exposes that option, so signer public keys are not published on-chain. This is a direct hedge: if ECDSA degrades faster than expected, an off-chain multisig "degrades gracefully" instead of exposing every signer key.
* **Deadline.** Contract accounts SHOULD be migrated before Phase 3, and a registry of un-migrated, high-value contract accounts SHOULD be published to prioritize outreach.

### 4.13 Rationale

* **Why ML-DSA-65 (not -44) as the user default?** Because the horizon is a decade-plus and lattices carry a *concrete-security* risk (§2.3) that is best managed with headroom; Category 2 is retained only for bounded/low-value contexts. The cost is ~1 KiB more per transaction — far cheaper than a second migration.
* **Why ML-DSA-87 for consensus/admin?** These keys are few, long-lived, and control the chain; the size premium is immaterial for the value at risk.
* **Why is SLH-DSA co-primary rather than a fallback?** Per §1.5 and §3.1, a deployer may reasonably prefer *no structured assumptions*. Making SLH-DSA an equal-status option lets that deployer act without waiting for a protocol change.
* **Why not FN-DSA/Falcon?** Floating-point Gaussian sampling and side-channel sensitivity make it a poor fit for IoT and for constant-time client code; it is reserved for a future IIP.
* **Why not an EdDSA-style seed migration?** IoTeX is secp256k1/ECDSA; the deterministic-seed property that makes Sui's DMS work is absent (§3.4). The mnemonic-preimage circuit (§4.9, Path B) is the closest analogue and is explicitly bounded.
* **Why is there a separate bunker track?** Because it is the only defensive measure that is (a) free, (b) protocol-independent, and (c) effective specifically for the largest, most-exposed holders, who are also the ones the ecosystem cannot afford to lose. Round-efficiency of a migration is measured in *value protected per unit of change*, and the bunker track dominates on that metric.

## 5. Technology Stack

### 5.1 Cryptographic Libraries

| **Component** | **Library** | **Justification** |
| --- | --- | --- |
| ML-DSA | liboqs (Open Quantum Safe) | NIST-aligned, audited, C with Go/Rust bindings |
| SLH-DSA | liboqs | Same library for consistency |
| zkSTARK prover | SP1 (Succinct) | STARK-native, Rust, WASM-compatible, no quantum-vulnerable final recursion |
| zkSTARK verifier (on-chain) | SP1 Solidity verifier | Deployable on EVM-compatible chains |
| Hash functions | SHA-256, SHA-512, Keccak-256 | Existing IoTeX hashes; sized per §2.5 |
| Formal verification | Lean4 / Coq | Following EF's formal-verification approach |

All client-side ML-DSA and SLH-DSA code MUST use constant-time, masked implementations, and MUST support the FIPS 204 *hedged* (randomized) signing mode (see §12.3).

### 5.2 Client Modifications

* **Transaction pool:** accept and propagate Type `0x05`; validate both PQ and legacy signatures, and enforce the §13 mempool policy, before admission.
* **Block validation:** verify PQ signatures on proposals and attestations (Phase 2+); reject ECDSA-only blocks after Phase 3 activation.
* **State management:** integrate PQ Key Registry lookups (including the §4.5 binding invariant) into the transaction-validation pipeline.
* **P2P networking:** upgrade to PQ-TLS (ML-KEM key encapsulation) for peer connections, following the Open Quantum Safe provider for OpenSSL 3. Note: this protects the *transport*; it is not a substitute for PQ transaction and consensus signatures.
* **RPC endpoints:** expose methods for PQ key registration, migration-proof submission, key lookup, and key-exposure status (§4.6).

### 5.3 Wallet and SDK Modifications

* **iotex-antenna (JS/TS SDK):** ML-DSA/SLH-DSA key generation, signing, verification; Type `0x05` construction; SP1 WASM prover for client-side migration proofs; key-exposure status surfaced by default.
* **ioPay (mobile wallet):** PQ key generation and secure-enclave storage; TEE-based proving service for migration proofs; MUST default to `registerWithProof` for unexposed accounts (§4.5).
* **Hardware wallet integration:** add ML-DSA support. In the interim, use a hybrid approach where the hardware wallet signs the ECDSA component and a software companion generates the PQ signature — with an explicit warning that this hybrid provides no on-device PQ protection for the PQ half.

## 6. Migration Roadmap

### 6.1 Five-Phase Migration Plan

| **Phase** | **Time Period** | **Objective** | **Key Deliverables** |
| --- | --- | --- | --- |
| **Phase 0: Foundation** | 2026 Q3 – 2027 Q2 | Deploy infrastructure without protocol changes | Publish this IIP and complete review; implement ML-DSA/SLH-DSA in `iotex-core` as an optional module; deploy PQ Key Registry and STARK verifier contracts on mainnet (voluntary registration only); **launch iPACT commitment generation and publish the recommended cutoff** (§4.10); launch TEE proving service; integrate PQ key generation into ioPay (opt-in); formal verification of the PQ precompile; hybrid PQ-TLS; publish DePIN PQ Device Identity Standard; **publish the §2.4 exposure census** |
| **Phase 1: Precompile Deployment** | 2027 Q3 – 2028 Q2 | Deploy PQ verification as a native protocol capability | Hard fork deploying precompile `0x0B`; enable Type `0x05` (hybrid required); activate registry enforcement for Type `0x05`; all 24 delegates MUST register ML-DSA-87 keys; begin dual-signed block production; gas-subsidy program for migration transactions (first 12 months, subject to §14); ecosystem migration toolkit |
| **Phase 2: Hybrid Enforcement** | 2028 Q3 – 2030 Q2 | Make PQ the primary authentication mechanism while retaining compatibility | Soft deadline (2028 Q3): new accounts in official wallets default to PQ-capable; hard deadline (2029 Q3): transactions to PQ-registered accounts MUST use Type `0x05`; consensus transition (2030 Q1): PQ-only consensus, ECDSA-only delegates excluded; deploy recursive STARK aggregation; expand DePIN migration; publish full security audit |
| **Phase 3: Legacy Sunset** | 2030 Q3 – 2031 Q4 | Restrict and eventually eliminate classical ECDSA | Soft fork (2030 Q3): legacy types accepted only from senders without a registered PQ key; 18-month grace period with aggressive outreach (automated migration, exchange partnerships, device OTA); iPACT rescue and zkSTARK rescue remain open; **governance vote** on unmigrated accounts (preserve / time-locked freeze / burn) per §14 |
| **Phase 4: PQ-Native** | 2032 Q1 – 2033 | Complete the transition to PQ-only operation | Hard fork (2032 Q1): legacy ECDSA transactions no longer processed; rescue paths remain for late migrants with valid pre-cutoff iPACTs; remove ECDSA from the critical consensus path (retain `ecrecover` only under the §11 deprecation schedule); evaluate STARK aggregation for execution-layer verification; evaluate emerging PQ algorithms via §4.11; publish final migration report |

### 6.2 Emergency Track

If credible evidence emerges of a CRQC capable of at-rest attacks before Phase 3 completion:

* **Immediate (within 48 hours):** issue an ecosystem advisory; activate PQ-only processing for all PQ-registered accounts; coordinate emergency key rotation with exchanges and custodians.
* **Short-term (within 2 weeks):** emergency hard fork requiring PQ authentication for high-value transactions (> 10,000 IOTX); a *gated* hold on accounts with exposed keys that have not registered a PQ key — **the hold MUST be released only by a PQ-sound proof or a pre-cutoff iPACT** (a hold releasable by an ECDSA registration signature is forgeable post-break; see §14); deploy a commit-delay-reveal protocol for legacy transactions.
* **Medium-term (within 3 months):** accelerate Phase 4; implement the BIP-361-style rescue for all accounts with provable ownership; **explicitly announce that exposed keys are, from this point, unrecoverable by cryptographic means** and that only pre-committed iPACTs survive.

### 6.3 Dependency Order

Phase 1 depends on the Phase 0 registry and verifier being live and audited; Phase 2's consensus transition depends on delegate key registration completing in Phase 1; Phase 3's sunset depends on the iPACT/rescue contracts being live and the exposure census (§2.4) being published so that the "who is left" population is known. A phase MUST NOT activate if its dependencies are unmet.

## 7. IoTeX PQC Maturity Model

This IIP defines five maturity levels for IoTeX ecosystem participants (clients, wallets, dapps, devices, custodians), concretizing Meta's framework (§3.11). A participant's level is determined by **on-chain-observable** metrics so that progress is verifiable, not self-reported.

| **Level** | **Name** | **Definition** | **On-chain metric** |
| --- | --- | --- | --- |
| **0** | PQ-Unaware | Uses only secp256k1/ECDSA; no PQ capability. | `pq_registered = false`; no Type `0x05` history. |
| **1** | PQ-Aware | Can verify PQ signatures; publishes a key-exposure status; supports reading Type `0x05`. | Registry lookup supported; exposure status RPC called. |
| **2** | PQ-Ready | Has generated a PQ key and can register; supports constructing Type `0x05`. | Has registered a PQ key (`registry[addr].active = true`). |
| **3** | PQ-Hardened | Actively signs Type `0x05` for new activity; contract accounts expose a PQ verifier path. | ≥ 90% of last-N outbound transactions are Type `0x05`; PQ verifier path present in contract account. |
| **4** | PQ-Enabled | Fully PQ-authenticated; no reliance on ECDSA for authentication. | All authentication is PQ; legacy types rejected for this account; for delegates, PQ-only attestations. |

**Aggregate metrics** (published per epoch, for the network): share of value in PQ-Hardened+ accounts; share of transaction *count* that is Type `0x05`; delegate PQ-readiness (target: 24/24 by Phase 1); device PQ-readiness; contract-account PQ-readiness; and the §2.4 exposure census over time. These metrics are the scoreboard for the phase deadlines in §6.

## 8. Guardrails

To prevent new quantum-vulnerable deployments as the network migrates:

* **New key creation (Phase 0+):** official wallets MUST generate a PQ key alongside any new secp256k1 key and MUST surface the exposure status of any imported key.
* **New contract accounts (Phase 1+):** a protocol-checked advisory (and, by Phase 3, a hard rule for protocol-endorsed templates) SHOULD flag any contract whose access control is ECDSA-only (`ecrecover`, `ecrecover`-based multisig), and the reference module (§4.12) SHOULD be the default template.
* **Blocked after Phase 3:** the protocol MUST reject a legacy-type transaction from an account with a registered PQ key (already stated in §6). After Phase 4, the protocol MUST reject all legacy-type transactions.
* **New DePIN devices (Phase 0+):** the DePIN PQ Device Identity Standard MUST require PQ-native provisioning (§4.8); a device shipped after a published date without a PQ identity is out of compliance.
* **Deployment checklist:** new smart contracts that manage value SHOULD use a published "PQ-ready" checklist, and tooling SHOULD warn on `ecrecover` usage.

## 9. Quantum-Milestone-Triggered Acceleration

The following table maps the Coinbase Independent Advisory Board's four milestones (§3.10) to concrete, pre-specified protocol actions, so that acceleration is triggered by observation rather than improvised.

| **Observable milestone** | **Trigger condition** | **Pre-specified IoTeX action** |
| --- | --- | --- |
| **M0: No CRQC** | Steady state | Execute the §6.1 phase plan. |
| **M1: Fault-tolerant two-qubit gates** | Independently confirmed | Complete Phase 0; begin Phase 1 engineering; publish the exposure census. |
| **M2: Fault-tolerant Shor's factoring (of a small integer)** | Independently confirmed | Jump to Phase 1 precompile deployment; mandate delegate PQ registration; activate the commit-delay-reveal option for high-value legacy transactions. |
| **M3: Indefinitely stable logical qubits / plausible ECDLP** | Credible expert consensus | Enter the §6.2 emergency track; require PQ authentication for high-value transactions; open the §3.14 rescue paths. |
| **M4: Verifiable quantum-simulation advantage** | Independently confirmed | Assume imminent T-break; accelerate Phase 4; publish the §14 governance decision on unmigrated accounts. |

Milestone assessment is performed by the cryptographic review committee (§7) with a published, dated report; the governance body (§14) authorizes the corresponding action.

## 10. Emergency Track Governance

Emergency powers (§6.2) are the most contentious part of any PQ migration. This IIP requires that they be:

* **narrowly scoped** — limited to the specific actions in §6.2, and only on a credible M3/M4 trigger (§9);
* **time-limited** — any emergency hold MUST expire automatically after a fixed window unless renewed by a governance vote;
* **subject to post-hoc review** — a public report on the use of emergency powers MUST be published within 30 days of their deactivation;
* **structurally controlled** — executed by a guardian multisig whose composition, threshold, and rotation are published, with individual signer keys PQ-migrated first.

## 11. Backward Compatibility

* **EVM Compatibility:** the PQ precompile extends the precompile set without modifying existing opcodes. Existing contracts continue to function; `ecrecover` is retained only per the deprecation schedule below.
* **Transaction Format:** Type `0x05` is additive under EIP-2718; existing types remain valid through Phase 3 (§6).
* **Address Format:** no address changes are required for existing accounts; PQ keys are bound to existing addresses via the registry. Native-PQ accounts use a domain-separated address derivation (§4.5) that cannot collide with legacy addresses.
* **Smart Contract Compatibility:** existing contracts do not require redeployment, but contracts performing on-chain signature verification SHOULD adopt the PQ verifier path (§4.12). A migration library (`PQVerifierLib.sol`) is provided.
* **Delegate Compatibility:** non-upgraded delegates can produce blocks through Phase 2 using ECDSA-only signatures; after the Phase 2 consensus transition, only PQ-capable delegates are in the active set.
* **`ecrecover` deprecation (new in this revision).** Retaining `ecrecover` "permanently" is a permanent forgery surface, not a compatibility guarantee. The protocol MUST: (a) publish a deprecation schedule with target deactivation no later than Phase 4 + 2 years; (b) introduce a gas disincentive for `ecrecover`-dependent value paths in Phase 3; and (c) publish a registry of known `ecrecover`-dependent contracts so users can be warned. "Retained for compatibility" MUST NOT be presented as a mitigation.

## 12. Security Considerations

### 12.1 Cryptographic Agility

See §4.11. Agility is coupled to an explicit re-parameterization trigger, owner, and dry-run requirement.

### 12.2 Rescue Soundness Across Timelines (new in this revision)

The single most important security consideration in this IIP is that **"rescue" means different things before and after a break** (§2.2). The following table is normative for how each mechanism MUST be labeled and relied upon:

| **Mechanism** | **Effective before T-governance?** | **Effective after T-break?** | **Requires prior holder action?** | **Label** |
| --- | --- | --- | --- | --- |
| ECDSA dual-signature registration (§4.5 Path 1) | Yes | No (forgeable by CRQC) | No | T-governance-only |
| Kiraz–Kardas Scenario A address binding (§4.9 Path A) | Yes | No (forgeable by CRQC) | No | T-governance-only |
| Mnemonic/seed-preimage proof (§4.9 Path B), witness = ρ | Yes | Yes, if ρ is not derivable from sk | No (but needs the mnemonic) | T-break-sound (conditional) |
| Mnemonic/seed-preimage proof (§4.9 Path B), witness = sk | Yes | No | No | T-governance-only |
| iPACT commitment (§4.10) | Yes | Yes | **Yes — must commit before cutoff** | T-break-sound |

The network MUST NOT advertise any T-governance-only mechanism as a post-break rescue. Post-break, an exposed key is recoverable only through a pre-committed iPACT (or the conditional Path B), which is why §4.10.3 sets the cutoff early.

### 12.3 Side-Channel and Signing-Mode Security

NIST PQC algorithms are susceptible to side-channel attacks (power/EM analysis, fault injection), as documented in the [Baseri et al. STRIDE analysis](https://arxiv.org/pdf/2501.11798). The IoTeX client MUST use constant-time, masked ML-DSA/SLH-DSA implementations from liboqs, and hardware wallets MUST include side-channel protections. In addition:

* Implementations MUST support and SHOULD use FIPS 204 **hedged (randomized)** signing; deterministic signing has a documented fault-attack exposure.
* During the hybrid window, the legacy ECDSA half MUST use RFC-6979 deterministic nonces (or equivalent); the hybrid is only as strong as its classical half until Phase 4.

### 12.4 Migration Proof Security

zkSTARK proofs rely on the soundness of the STARK system and the quantum preimage resistance of the underlying hashes. STARKs use no trusted setup and rely only on collision-resistant hash functions, making them plausibly post-quantum secure. SP1 is STARK-native end to end, avoiding a quantum-vulnerable Groth16 final recursion. As stated in §4.9, the security of Path B depends **entirely** on the witness being a true preimage rather than `sk`.

### 12.5 Mempool Privacy and Denial-of-Service

During the transition, Type `0x05` transactions that include an ECDSA signature reveal the public key in the mempool. Two mitigations are required:

* **Commit-delay-reveal** for high-value transactions: the sender commits to a hash, waits N blocks, then reveals — preventing on-spend attacks even for unmigrated accounts.
* **Mempool policy (new in this revision):** because Type `0x05` transactions are ~60–250× larger than legacy transactions (§13), nodes MUST enforce bounded PQ public-key sizes (rejecting any not matching a declared active algorithm), per-peer rate limits on PQ transactions, and an admission-time PQ-verification cost budget. Without these, PQ rollout becomes a bandwidth and CPU amplification vector. The precompile's per-algorithm gas (§4.3) also bounds the worst-case verification cost a caller can impose.

### 12.6 Quantum-Safe Consensus Integrity

With 24 delegates, Roll-DPoS is vulnerable if a CRQC can compromise multiple delegate keys simultaneously. Migrating all 24 delegate keys in Phase 2 eliminates this risk. During the hybrid period, the dual-signature requirement means an attacker must compromise both the ECDSA and PQ keys of a delegate to produce valid attestations. Note the inverse caveat: during hybrid, an attacker who can forge the *ECDSA* half can still impersonate a delegate's identity even if the PQ half is secure, which is why consensus PQ-only mode (Phase 2) is a required end-state and not optional.

### 12.7 DePIN Device Security

IoT devices often have limited compute and long lifetimes. SLH-DSA-SHA2-192s is RECOMMENDED for devices that can tolerate its 16,224-byte signature (hash-only security); ML-DSA-65 is the alternative where size is critical. For extremely constrained devices, the gateway-mediated verification model (§4.8) applies. Because device keys are long-lived, devices SHOULD also publish an iPACT (§4.10.5) at provisioning time.

## 13. Performance Analysis and Gas Economics

### 13.1 Transaction Size Impact

| **Transaction Type** | **Current (ECDSA)** | **Type 0x05 (ML-DSA-65 + ECDSA)** | **Type 0x05 (ML-DSA-87 + ECDSA)** | **Type 0x05 (SLH-DSA-192s + ECDSA)** |
| --- | --- | --- | --- | --- |
| Signature size | ~65 B | ~3,374 B (3,309 + 65) | ~4,692 B (4,627 + 65) | ~16,289 B (16,224 + 65) |
| Public key size | 0 (recovered) | ~1,952 B | ~2,592 B | ~48 B |
| Total overhead | ~65 B | ~5,326 B | ~7,284 B | ~16,337 B |
| Increase factor | 1× | ~82× | ~112× | ~251× |

> **Correction from Revision 1.** Revision 1 tabulated ML-DSA-44 (57–58×). Under the §4.1 Category-3 floor the overhead is ~82× for the default user scheme. This is the primary cost of raising the floor and MUST be budgeted.

### 13.2 Block Size and Throughput Impact

At IoTeX's current parameters, transaction-size growth reduces transactions per block substantially if block-size limits are unchanged. Mitigations: increase the block-size limit (straightforward given the 5-second block time and delegate consensus); introduce blob-style data availability for PQ signature data (EIP-4844-style); and implement STARK-based signature aggregation in later phases. Consensus-attestation bandwidth and storage are quantified in §4.7 (~700 GB/year at ML-DSA-87).

### 13.3 Gas Cost Comparison

| **Operation** | **Current Gas** | **PQ Gas (Phase 1–2)** | **PQ Gas (Phase 4)** |
| --- | --- | --- | --- |
| Simple transfer (ECDSA) | ~21,000 | N/A | Deprecated |
| Simple transfer (ML-DSA-65) | N/A | ~25,500 (21,000 + 4,500) | ~25,500 |
| Simple transfer (ML-DSA-87) | N/A | ~27,000 (21,000 + 6,000) | ~27,000 |
| Simple transfer (SLH-DSA-192s) | N/A | ~276,000 (21,000 + 255,000) | ~276,000 |
| PQ Key Registration | N/A | ~100,000 (one-time) | N/A |
| zkSTARK Migration Proof | N/A | ~300,000–500,000 (one-time) | N/A |

### 13.4 Migration Incentives

* **Gas subsidy (Phase 1, 12 months):** PQ key registration and the first Type `0x05` transaction receive a gas discount, funded from a migration subsidy pool that MUST be separately authorized by governance (§14). The funding source and cap MUST be stated in the authorization proposal.
* **Priority inclusion:** Type `0x05` transactions receive priority ordering in the block (above same-fee legacy transactions) during Phases 1–2.
* **Delegate rewards:** delegates completing PQ migration before the Phase 2 deadline receive a bonus reward multiplier for 6 months, subject to the same authorization.

## 14. Governance of Contentious Powers

Several decisions in this IIP are genuinely contentious and MUST be made explicitly and publicly, before they become market-impacting:

* **Dormant/unmigrated accounts (preserve / time-locked freeze / burn):** the community MUST decide via a governance vote, with the options, thresholds, and consequences published well in advance (target: before Phase 3). This IIP does not presume the outcome.
* **Voting mechanism and thresholds:** phase activations, the iPACT cutoff, the re-parameterization threshold (§4.11), and any emergency action MUST specify (a) who may propose, (b) the approval threshold (e.g., delegate super-majority plus community signal), and (c) the minimum notice period.
* **Emergency powers:** governed per §10.
* **T-break disclosure (normative):** the network MUST publicly state that, after T-break, accounts with already-exposed public keys and no pre-committed iPACT are **not recoverable** by cryptographic means, and MUST NOT imply otherwise in any advisory or UI.

## 15. Activation and Parameters

| **Parameter** | **Proposed value** | **Set by** |
| --- | --- | --- |
| Precompile address | `0x000000000000000000000000000000000000000B` | This IIP |
| Precompile version byte | `0x01` | This IIP |
| Domain tags | `IoTeX-PQ-Tx-v1`, `IoTeX-PQ-Register-v1`, `IoTeX-PQ-Consensus-v1`, `IoTeX-PQ-Addr-v1` | This IIP |
| Native-PQ address derivation | `keccak256("IoTeX-PQ-Addr-v1" || chain_id || algorithm_id || keccak256(pk))[12:32]` | This IIP |
| PQ Key Registry address | *(to be specified at deploy)* | Governance |
| Delegate PQ scheme | ML-DSA-87 | This IIP |
| iPACT cutoff block | *(to be specified; set as early as possible)* | Governance (§4.10.3) |
| Re-parameterization risk threshold | 32 bits (proposed) | Governance (§4.11) |
| Phase activation heights | *(per phase, by fork)* | Governance (§14) |

Activation of each phase REQUIRES a passed governance vote (§14) and, for Phase 1, the Phase 0 dependencies (§6.3). Fork activation heights are intentionally left to governance; this IIP fixes the *rules*, not the *clock*.

## 16. Test Vectors

This section defines the test-vector format. Concrete vectors MUST be generated by the reference implementation (§17) and published in the `iotextestv` test-vector repository before Phase 1 activation; they are normative for conformance.

Each vector is a JSON object:

```json
{
  "name": "<descriptive id>",
  "algorithm": "ML-DSA-65 | ML-DSA-87 | SLH-DSA-SHA2-192s | ...",
  "algorithm_id": "0x02",
  "ctx": "IoTeX-PQ-Tx-v1",
  "public_key": "<hex, from reference impl>",
  "message_or_preimage_fields": { "...": "..." },
  "signature": "<hex, from reference impl>",
  "expected": "valid",
  "vector_source": "reference implementation, version X"
}
```

Required coverage: at least one valid and one invalid vector per active algorithm; a Type `0x05` transaction hash/preimage vector; a registry `registration_hash` vector including the `registration_nonce`; a consensus `consensus_digest` vector; and a precompile input/output vector (including a malformed-framing vector that MUST revert). **No vector in this document is fabricated**; the values above are placeholders to be filled from the reference implementation.

## 17. Reference Implementation

To be developed as open-source public goods (repository links MUST be added here before Phase 1):

* **`iotex-core` modifications:** PQ precompile, Type `0x05` processing, dual-signed consensus.
* **`iotex-pq-registry`:** PQ Key Registry, including the zkSTARK verifier integration and the §4.5 binding invariant.
* **`iotex-pq-sdk`:** TypeScript/Go SDK extensions for key management, Type `0x05` construction, and proof generation.
* **`iotex-pq-prover`:** Rust/SP1 circuits for address binding (§4.9 Path A) and mnemonic-preimage migration (§4.9 Path B).
* **`iotex-pq-tee-service`:** TEE proving service for mobile/IoT migration.
* **`iotex-pq-depin-sdk`:** DePIN device identity SDK with ML-DSA/SLH-DSA support and the §4.10.6 iPACT refresh semantics.
* **`PQVerifierLib.sol`:** migration library for contract accounts (§4.12).

## 18. Open Questions

1. **Lattice vs. hash-based posture:** should IoTeX follow the EF lean path toward hash-only signing, or keep ML-DSA primary with SLH-DSA co-primary as specified? (§1.5, §4.1, §4.13)
2. **Security-level floor:** is Category 3 the right floor, or should consensus/admin use Category 5 exclusively and user keys stay Category 2 for size? (§4.1.1)
3. **iPACT cutoff timing:** what is the earliest block at which commitments can be reliably timestamped on both Bitcoin (OpenTimestamps) and IoTeX? (§4.10.3)
4. **Post-break rescue at scale:** if T-break arrives before mass iPACT commitment, what (if anything) should happen to un-migrated exposed accounts? (§14)
5. **Aggregation recall:** at what delegate-set size does STARK aggregation (leanMultisig-style) become necessary? (§4.7)
6. **Contract-account coverage:** is a protocol-enforced PQ path for multisigs/AA acceptable, or does it violate account-owner autonomy? (§4.12)
7. **Gas schedule finalization:** which benchmark methodology and hardware define the §4.3 constants?

## 19. Changelog (Revision 2, 2026-10-08)

**Blocking corrections**
1. Reconciled the algorithm-identifier mismatch: §4.3's inline comment now matches the canonical §4.1.3 table (`0x10`/`0x11` for SLH-DSA).
2. Added §4.2 defining every PQ signing preimage, with domain separation and chain binding.
3. Defined sender resolution in PQ-only mode and added the §4.5 **binding invariant**.
4. Fixed the native-PQ bootstrapping gap: added `registerNativePQ` (§4.5) and a native-PQ address derivation rule (§4.5, §4.8).
5. Corrected ML-DSA-65 size to **3,309 B** and removed the "0.5× vs 0.9×" verification contradiction.
6. Added the **T-governance vs T-break** distinction (§2.2), the rescue-soundness table (§12.2), and relabeled ECDSA-based registration/binding as T-governance-only.
7. Reconciled the Abstract with the body: added §7 (Maturity Model), §8 (Guardrails), §9 (Milestones).
8. Resolved FN-DSA: removed from active support, reserved as `0x80`.

**Design-level changes**
9. Added an explicit security target (Category 3 floor) and raised the floor from ML-DSA-44/SLH-DSA-128s to ML-DSA-65/ML-DSA-87 and SLH-DSA-192s/256s (§4.1).
10. Added the hash-hidden ("bunker") strategy (§2.6, §4.6) and made `registerWithProof` the default for unexposed accounts, with a de-anonymization warning.
11. Moved iPACT generation to Phase 0 and defined refresh/replay/renewal semantics (§4.10.3, §4.10.6).
12. Added the re-parameterization trigger, owner, and dry-run requirement (§4.11).
13. Added the mempool DoS policy (§12.5), consensus bandwidth/storage quantification (§4.7), and the `ecrecover` deprecation schedule (§11).
14. Added the multisig/AA section and off-chain confirmation guidance (§4.12).
15. Added the application-layer credential carve-out for DePIN (§4.8) and the iPACT-DePIN refinements (§4.10.6) — resolving both open questions raised in the PR review thread.

**Structural and editorial**
16. Renumbered duplicate sections 3.9/3.11; fixed the `[Google whitepaper}` link and the typos ("Likelyhood"→Likelihood, "at leas"→at least, "translation size"→transaction size, "Compatability"→Compatibility).
17. Added §1.4 (Conventions), §4.13 (Rationale), §14 (Governance), §15 (Activation), §16 (Test Vectors), §18 (Open Questions), and §6.3 (Dependency Order).
18. Added §1.5 separating threat-model **assumptions** from **findings**, in light of the public debate (Drake, Buterin, Lindell), and added the lattice concrete-security row to the §2.3 risk matrix.
19. Strengthened §12.3 (hedged signing, RFC-6979 for the classical half) and §12.6 (hybrid is only as strong as its ECDSA half until Phase 2).

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
