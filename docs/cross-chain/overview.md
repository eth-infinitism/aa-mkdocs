
# Overview

## What WeissChannels solve and why they matter.

WeissChannels is a protocol extension for ERC-4337 Smart Accounts that enables single-signature, atomic, cross-chain execution without custodial intermediaries.

---

## 🎯 The Problem

Ethereum's rollup-centric roadmap has successfully scaled throughput but has severely fragmented the user experience. Each Layer 2 functions as an isolated island with its own gas market, liquidity, and bridging delays.

Users are effectively "chain-bound," forced to choose between:

1. **Trust-minimized canonical bridges** with prohibitive latency (7+ days for Optimistic Rollups)
2. **Third-party intent solvers** with centralized trust assumptions

Current cross-chain solutions compromise Ethereum's core values:

- **Censorship Resistance** — Transacting through permissioned intermediaries
- **Security** — Trusting third parties with funds or state attestation
- **Privacy** — Exposing user IP addresses and intentions to centralized servers
- **Openness** — Relying on opaque, proprietary off-chain solving logic

---

## ⚠️ Why Intent Solvers Fail

At first glance, intent solvers seem like the path of least resistance. However, permissionless and decentralized solvers face structural DoS and griefing risks:

### The "Salmonella" Risk

A malicious user could deploy a contract on the destination chain that reverts only when a specific solver attempts to fulfill an intent. This causes the solver to burn gas without compensation.

### The Whitelist Response

To mitigate this, solvers inevitably create whitelists of "safe" contracts (e.g., only USDC and WETH). This degrades censorship resistance and hinders permissionless innovation.

### Verification Complexity

General purpose intents can be arbitrarily complex. If a solver executes an intent with marginal gas, causing an internal revert while the outer transaction succeeds, the user pays but receives nothing.

---

## 🔺 The Privacy/Safety/UX Trilemma

Submitting an intent reveals the user's desired outcome ahead of time. This creates a fundamental trilemma:

| Optimization | Example Approach | What is Lost |
|-------------|------------------|--------------|
| Privacy + UX | On-chain intent protocols (public mempool) | **Safety** — Transactions observed and sandwiched before inclusion |
| Safety + UX | Reputable off-chain solver networks | **Privacy** — User IP and intent metadata revealed to centralized solver |
| Safety + Privacy | Split flow: Bridge then execute | **UX** — High latency, multiple signatures, fragmented flow |

---

## ✅ The WeissChannel Solution

WeissChannels solve this trilemma by **putting the user in control**. By enabling the user to execute their own transactions on the destination chain (using the XLP only for liquidity/gas), we remove the need for a third party to understand or simulate the transaction logic.

### Security Model

The system follows an **Optimistic Rollup security model** with a 1-of-N honest assumption: the system is secure as long as *one* honest actor (a watcher, the user, or a rival XLP) is online to submit a fraud proof.

This is fundamentally different from:

| System | Trust Assumption |
|--------|------------------|
| Custodial Bridges | Honest Majority (>50%) |
| Intent Networks | Honest Federation |
| **WeissChannels** | **1-of-N Honest** |

### The Vision

**Multichain should feel like a single chain.**

If Alice on Arbitrum wants to send ETH to Bob on Base, she should:

1. Paste the address and click send
2. Her wallet abstracts the complexity
3. Pay fees in any asset
4. Sign once
5. Settle within seconds
6. Never trust a third party

---

## 🔗 Related Pages

- [Architecture](architecture.md) — How WeissChannels work under the hood
- [Security](security.md) — Deep dive into the trust model and attack mitigations
- [Paymasters](../paymasters/README.md) — Gas abstraction in ERC-4337

