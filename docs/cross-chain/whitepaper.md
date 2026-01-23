
# Whitepaper

## WeissChannels: Trust-Minimized Cross-Chain Execution via Account Abstraction

**Authors:** Yoav Weiss, Shahaf Nacson, Marissa Posner, Dror Tirosh, Alex Forshtat, Tom Teman

**Date:** December 25, 2025

---

## Abstract

Ethereum's rollup-centric roadmap has successfully scaled throughput but has severely fragmented the user experience (UX). Today, each Layer 2 (L2) functions as an isolated island with its own gas market, liquidity, and bridging delays. Users are effectively "chain-bound," forced to choose between the prohibitive latency of trust-minimized canonical bridges or the centralized trust assumptions of third-party intent solvers.

WeissChannels is a protocol extension for ERC-4337 Smart Accounts that enables single-signature, atomic, cross-chain execution without custodial intermediaries. By leveraging ephemeral, "just-in-time" state channels secured optimistically by Ethereum Layer 1, WeissChannels restore the seamless experience of a single chain while preserving the core Ethereum guarantees of censorship resistance, self-custody, and disintermediation.

---

## Key Contributions

### 1. The Interoperability Trilemma

We identify a fundamental **Privacy/Safety/UX Trilemma** in cross-chain protocols:

| Optimization | What is Lost |
|--------------|--------------|
| Privacy + UX | Safety (transactions sandwiched) |
| Safety + UX | Privacy (intent metadata exposed) |
| Safety + Privacy | UX (high latency, multiple signatures) |

WeissChannels solve this by putting the user in control—they execute their own transactions using XLPs only for liquidity.

### 2. Just-In-Time State Channels

Unlike traditional state channels (Lightning Network) which are long-lived and require pre-funding, a WeissChannel is:

- **Multilateral and ad-hoc** — spun up for exactly one transaction
- **Immediately dissolved** — no capital lockup
- **Dynamically funded** — liquidity sourced from permissionless XLP pool

### 3. L1 as the Supreme Court

The protocol adheres to an **Optimistic security model**:

- Ethereum Mainnet is never used for transaction execution
- L1 acts strictly as the judiciary, holding stakes and processing dispute proofs
- Normal case operates at L2 speed; worst case retains L1 security

### 4. 1-of-N Honest Assumption

The system is secure as long as ONE honest actor (watcher, user, or rival XLP) submits fraud proofs. This is analogous to how Optimistic Rollups work.

---

## Protocol Highlights

### Multichain UserOps

Users sign a **Merkle root** of a tree containing operations for multiple chains. This enables:

- Single signature from hardware wallet
- Per-chain Merkle proofs for validation
- Heterogeneous account support (ECDSA, Passkeys, BLS, multisig)

### Reverse Dutch Auction

Fees are discovered through a reverse Dutch auction where:

- User sets a starting fee and ramp rate
- Fee increases over time until an XLP fulfills
- Competition ensures best rates

### Zero-Overhead Claiming

The XLP does not submit a separate transaction on the destination chain. Voucher validation happens during the ERC-4337 validation phase of the user's own transaction.

### Capital Efficiency: O(Networks)

- XLP stake is a **performance bond**, not liquidity backing
- Stake scales with number of networks, not volume
- A single 10 ETH stake can secure unlimited volume

---

## Use Cases

### Seamless Cross-Chain Transfer

Alice on Arbitrum sends 100 USDC to Bob on Base:

1. Alice pastes Bob's address, clicks send
2. Wallet generates Multichain UserOp
3. Alice signs once
4. Bob receives funds in seconds

### Seamless Multichain Calls

Alice mints an NFT on Linea that costs 1 ETH. She has 0.8 ETH on Arbitrum and 0.5 ETH on Scroll:

1. Wallet detects insufficient funds on Linea
2. Constructs a tree aggregating liquidity from multiple chains
3. Alice signs the root
4. Protocol executes and mints the NFT

### Round-Trip Workflows

- Bridge to high-yield chain → deposit → receive receipt token → bridge receipt back
- Buy asset on Chain B → sell on Chain A in same atomic bundle

---

## Comparative Security Analysis

| Feature | Custodial Bridge | Intent Network | WeissChannels |
|---------|------------------|----------------|---------------|
| **Trust Analogy** | Alt-L1 | Federated Sidechain | Optimistic Rollup |
| **Verification** | External Consensus | Off-chain Reputation | L1 Adjudication |
| **Security Model** | Honest Majority (>50%) | Honest Federation | **1-of-N Honest** |
| **Worst Case** | Funds Stolen | Censorship | **Revert (Funds Safe)** |

---

## Future Work

### Native Account Abstraction (EIP-7701)

WeissChannels are designed to be forward-compatible with EIP-7701, which will further lower gas costs and improve censorship resistance.

### Trustless ZK Messaging

As rollups move toward ZK validity proofs with fast finality, we will introduce a `CrosschainMessagingPaymaster` that functions like a Uniswap LP (passive) rather than an atomic swap (active XLP).

### Generalized Merkle Root Authority

Future work will explore generalizing the `ChainID` into a `ContextID`, extending single-signature authorization to non-EVM domains or off-chain computation environments.

---

## Conclusion

WeissChannels restore the "Dark Forest" property of Ethereum—adversarial, permissionless, and secure—to the cross-L2 landscape. By embedding interoperability directly into the Account layer via ERC-4337 and leveraging the existing L1 validator set for dispute resolution, we dissolve the boundaries between chains.

This architecture offers a path to a unified "Superchain" experience where the complexity of the underlying infrastructure is completely abstracted from the user, finally fulfilling the promise of a scalable, singular Ethereum.

---

## References

1. Weiss, Y., et al., & Buterin, V. (2021). *ERC-4337: Account Abstraction Using Alt Mempool*. [EIP](https://eips.ethereum.org/EIPS/eip-4337)

2. Herlihy, M. (2018). *Atomic Cross-Chain Swaps*. PODC '18.

3. Kilbourn, Q., & Konstantopoulos, G. (2023). *Intent-Based Architectures and Their Risks*. [Paradigm](https://www.paradigm.xyz/2023/06/intents)

4. Daian, P., et al. (2019). *Flash Boys 2.0*. IEEE S&P.

5. Buterin, V. (2021). *An Incomplete Guide to Rollups*. [vitalik.eth.limo](https://vitalik.eth.limo/general/2021/01/05/rollup.html)

6. Poon, J., & Dryja, T. (2016). *The Bitcoin Lightning Network*. [lightning.network](https://lightning.network/lightning-network-paper.pdf)

---

## Reference Implementation

A reference implementation is available at: [github.com/eth-infinitism/eil-contracts](https://github.com/eth-infinitism/eil-contracts)

The implementation includes:

- `CrossChainPaymaster.sol` — L2 paymaster with origin/destination swap management
- `L1AtomicSwapStakeManager.sol` — L1 stake and dispute resolution
- Bridge connectors for Arbitrum and Optimism
- Integration tests demonstrating cross-chain flows

---

## 🔗 Related Pages

- [Overview](overview.md) — Problem space and motivation
- [Architecture](architecture.md) — Detailed technical architecture
- [Contracts](contracts.md) — Solidity interfaces
- [ERC-4337](../core-standards/erc-4337.md) — The underlying standard

