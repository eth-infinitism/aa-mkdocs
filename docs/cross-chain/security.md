
# Security

## Trust model, dispute resolution, and attack mitigations.

This page provides a comprehensive security analysis of WeissChannels, including the trust model, attack vectors, and how the protocol protects users and XLPs.

---

## 🔒 Trust Model: 1-of-N Honesty

WeissChannels operate under an **optimistic security model** with a 1-of-N honest assumption:

> The system is secure as long as at least ONE honest actor is online and willing to submit a fraud proof.

This is strictly weaker than:

| Model | Requirement | Examples |
|-------|-------------|----------|
| N-of-N (Unanimous) | All parties must be honest | Multisig bridges |
| M-of-N (Threshold) | Majority must be honest | PoS consensus |
| Trusted Third Party | One designated entity must be honest | Centralized exchanges |
| **1-of-N (WeissChannels)** | **Any ONE actor can prove fraud** | Optimistic Rollups |

### Who Can Be the "1"

- The user themselves
- A rival XLP (economically incentivized to catch cheaters)
- A public watcher service
- Any Ethereum user with the fraud proof data

### Implications

- Users who monitor their own transactions have sovereign security
- Even passive users benefit from the existence of watchers
- The cost of running a watcher is low (just monitoring events)

---

## 🛡️ The Optimistic Security Guarantee

The system relies on the following security flow:

1. **The Threat:** An XLP issues a Voucher but is insolvent on the Destination Chain
2. **The Defense:** Any watcher can submit a fraud proof to the L1 Stake Manager
3. **The Penalty:** The malicious XLP is slashed on L1
4. **User Safety:** The user never loses funds — their funds on the source chain simply unlock after expiration

### Security Inequality

The system remains secure against griefing attacks as long as:

```
S_XLP > U_grief
```

Where `S_XLP` is the slashed stake and `U_grief` is the utility gained from griefing.

Crucially, since the XLP must provide the voucher *before* claiming user funds, theft is impossible (U_theft = 0). Therefore:

```
S_XLP << V_tx
```

The stake does not need to cover the transaction value, allowing high capital efficiency.

---

## ⚔️ Dispute Types

Three types of misbehavior can trigger slashing:

| Type | Description | Trigger |
|------|-------------|---------|
| **INSOLVENT_XLP** | XLP issued a voucher but didn't have funds to honor it | `disputeInsolventXlp()` |
| **VOUCHER_OVERRIDE** | XLP falsely claimed to have provided liquidity | `disputeVoucherOverride()` |
| **UNSPENT_VOUCHER_FEE_CLAIM** | XLP claimed fees for a voucher that was consumed | `disputeXlpUnspentVoucherClaim()` |

### Fraud Proof Structure

Each dispute requires evidence from both origin and destination chains, matched on L1:

```
Origin Chain Evidence (ReportDisputeLeg):
├── requestIdsHash (identifies the disputed requests)
├── count (number of requests in batch)
├── firstRequestedAt / lastRequestedAt (timing bounds)
├── disputeTimestamp (when dispute was raised)
└── l1Beneficiary (who receives slashed funds)

Destination Chain Evidence (ReportProofLeg):
├── requestIdsHash (must match origin)
├── count (must match origin)
├── proofTimestamp (when proof was produced)
├── firstProveTimestamp (for pre/post window classification)
└── l1Beneficiary (who receives slashed funds)
```

L1 matches these legs by `requestIdsHash` and processes the slash atomically.

---

## 🏆 Longest-Array Algorithm

For insolvency disputes, multiple watchers may report the same misbehavior. To incentivize comprehensive reporting, the slashing system uses a **longest-array algorithm**:

1. **Pre/Post Windows:** Disputes are classified based on timing
2. **Longest Array Wins:** The reporter with the most request IDs wins their role's share
3. **Tie-Breaking:** On equal counts, the earliest destination proof timestamp wins
4. **Five Roles:** Pre-origin, pre-destination, post-origin, post-destination, L1-pull
5. **Distribution:** 90% of slashed stake is distributed

For non-insolvency disputes (VOUCHER_OVERRIDE, UNSPENT_VOUCHER_FEE_CLAIM), a single beneficiary receives the slashed funds.

---

## 🔄 Replay Attack Prevention

### Same-Chain Replay

**Threat:** Attacker replays a user's signed operation on the same chain.

**Mitigation:** Each operation includes the account's nonce. Once executed, the nonce increments, invalidating replays.

### Cross-Chain Replay

**Threat:** Attacker takes an operation signed for Chain A and submits it on Chain B.

**Mitigation:** Each Merkle leaf includes `chainId`. The leaf hash verification fails if `block.chainid` doesn't match.

### Voucher Replay

**Threat:** Attacker replays an old voucher to extract funds.

**Mitigation:**
- Vouchers reference a specific `requestId` (hash including nonce)
- Vouchers have `expiresAt` timestamps
- Consumed vouchers are marked in contract state

### Request Replay

**Threat:** Attacker replays a user's voucher request to lock their funds again.

**Mitigation:** The `senderNonce` in `SourceSwapComponent` prevents replay.

---

## 💸 Double-Spend Prevention

### XLP Double-Spend During Unstake

**Threat:** An XLP initiates unstake, waits until just before withdrawal, issues vouchers they don't honor, then withdraws stake.

**Mitigation:**

1. **Immediate Unregistration:** When `unlockStake()` is called, the XLP is immediately removed from the active registry
2. **8-Day Unstake Delay:** Exceeds maximum L2 finality (7 days)
3. **Stake Remains Slashable:** Until actually withdrawn

```
Timeline:
Day 0: XLP calls unlockStake() → Immediately unregistered
Day 0-7: Outstanding vouchers can still be disputed
Day 7: L2 finality reached for all pending transactions
Day 8: XLP can call withdrawStake() (if not slashed)
```

### User Double-Spend via Reorg

**Threat:** User's lock is included in Block N. The chain reorgs, and the user spends differently in Block N'.

**Mitigation:**

1. **Anti-Rugpull Lock:** Claimed funds remain locked briefly
2. **XLP Can Resubmit:** If the XLP holds the user's signed UserOp, they can resubmit to the new chain tip
3. **Nonce Consumption:** If the user already used that nonce in the reorg, the XLP's submission fails

---

## 🌊 Reorg Safety

### Deep Reorg Scenario

| Depth | Likelihood | Impact | Mitigation |
|-------|------------|--------|------------|
| 1-2 blocks | Common | XLP resubmits user's tx | XLP monitors and reacts |
| 3-12 blocks | Rare | XLP may lose if slow | Anti-rugpull lock period |
| 13+ blocks | Extremely rare | Potential loss | Finality assumptions |

### XLP Risk Management

XLPs SHOULD:
- Wait for sufficient confirmations before issuing vouchers for large amounts
- Set lower `expiresAt` on vouchers for chains with uncertain finality
- Price reorg risk into their fee calculations

---

## 🎭 Griefing Economics

### User Griefing XLP

**Attack:** User creates many requests, gets vouchers issued, never consumes them.

| Metric | Value |
|--------|-------|
| Cost to User | `unspentVoucherFee × amount × numberOfRequests` |
| Cost to XLP | Gas for `issueVouchers()` + capital lockup |

**Equilibrium:** If `unspentVoucherFee` is set appropriately, griefing is unprofitable.

### XLP Griefing User

**Attack:** XLP issues voucher but doesn't have destination liquidity.

| Metric | Value |
|--------|-------|
| Cost to XLP | Full stake slashed |
| Cost to User | Time delay (funds locked until expiration) |

### Concrete Calculation

```
Assume:
- Minimum stake per chain: 10 ETH (~$30,000)
- Maximum griefing utility: Delay user's $1M for 5 minutes
- User's opportunity cost: 10% APY = $1M × 0.10 × (5/525600) = $0.95

Griefing Ratio = Stake / Griefing Utility = $30,000 / $0.95 = 31,579x
```

The XLP would lose 31,579x more than they could possibly hurt the user.

---

## 🆘 XLP Insolvency Handling

### Scenario: XLP Becomes Insolvent Mid-Operation

An XLP might become insolvent due to:
- Multiple simultaneous voucher requests exceeding reserves
- Price movements making reserves insufficient
- Technical failure preventing destination funding

### Protocol Response

1. **Voucher Not Consumed:** User's funds remain locked until expiration, then unlock
2. **Dispute Window:** Anyone can call `disputeInsolventXlp()` with proof
3. **Slashing:** XLP's L1 stake is slashed
4. **Alternative Vouchers:** During dispute, other XLPs can issue `ALT` vouchers to rescue the user

### User Protection

Users never lose funds due to XLP insolvency. Worst case:
- Their funds are locked for the expiration period
- They can cancel after expiration and recover everything
- If another XLP issues an ALT voucher, they can complete their transaction

---

## 🌉 Cross-Chain Message Integrity

### Bridge Trust Assumptions

The protocol relies on L1 ↔ L2 messaging via **canonical rollup bridges** (e.g., Arbitrum's Outbox, Optimism's CrossDomainMessenger):

- **Optimistic Rollups:** Messages finalized after the challenge period (7 days)
- **ZK Rollups:** Messages finalized after proof verification

### Security Inheritance

```
WeissChannel Security ≤ min(L1 Security, Weakest Supported L2 Security)
```

The protocol cannot be more secure than the least secure bridge it uses.

---

## ⏱️ Timing Attacks

### Manipulate Block Timestamps

**Threat:** Miner/sequencer manipulates timestamps to expire vouchers prematurely or extend validity.

**Mitigation:**
- Timestamps only need ~minute accuracy
- `expiresAt` windows are typically 5-30 minutes (much larger than manipulation range)
- Fee ramp calculations are approximate; small manipulation has minimal impact

### Delay Dispute Submission

**Threat:** Censorship of dispute transactions to prevent fraud proofs.

**Mitigation:**
- Multiple chains can submit evidence
- L1 itself is censorship-resistant
- Dispute windows (`claimDelay`) provide buffer time
- Users can use Flashbots/private channels

---

## ⚠️ Known Limitations

1. **Not Atomic Across Chains:** Multi-leg transactions may partially succeed
2. **Finality Assumptions:** If an L2 experiences catastrophic failure, funds may be stuck
3. **Bridge Dependency:** Security is bounded by the canonical bridges
4. **Capital vs. Security Tradeoff:** Higher stakes mean more security but lower efficiency
5. **Heterogeneous Finality:** Different L2s have different finality times; `unstakeDelay` must exceed the maximum

---

## ✅ Summary: Trust Comparison

| Feature | Custodial Bridge | Intent Network | WeissChannels |
|---------|------------------|----------------|---------------|
| **Trust Analogy** | Alt-L1 | Federated Sidechain | Optimistic Rollup |
| **Verification** | External Consensus | Off-chain Reputation | L1 Adjudication |
| **Security Model** | Honest Majority (>50%) | Honest Federation | **1-of-N Honest** |
| **Worst Case** | Funds Stolen | Censorship / Bad Execution | **Revert (Funds Safe)** |

---

## 🔗 Related Pages

- [Architecture](architecture.md) — Protocol lifecycle and data structures
- [Contracts](contracts.md) — Dispute interfaces and slashing mechanics
- [Fees & Economics](fees-and-economics.md) — Griefing costs and equilibrium

