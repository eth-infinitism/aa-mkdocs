
# Fees & Economics

## Reverse Dutch auction, XLP incentives, and MEV dynamics.

This page explains the economic model for XLP compensation, including how fees are determined, how XLPs decide whether to participate, and the role of MEV.

---

## 🔧 Reverse Dutch Auction

To ensure users receive the best price without latency, the protocol utilizes a fee discovery mechanism based on a **reverse Dutch auction**.

### How It Works

1. **Start Fee:** The user sets a starting fee percentage, ideally close to the current market rate
2. **Ramping:** The fee increases over time according to the `feeIncreasePerSecond` parameter
3. **Competition:** Multiple XLPs monitor for requests; the first to issue a voucher locks in the current fee
4. **Cap:** The fee never exceeds `maxFeePercentNumerator`

### Fee Parameters

```solidity
struct AtomicSwapFeeRule {
    uint256 startFeePercentNumerator;   // Starting fee (1 = 0.01%, 100 = 1%)
    uint256 maxFeePercentNumerator;     // Maximum fee cap
    uint256 feeIncreasePerSecond;       // Fee increase per second
    uint256 unspentVoucherFee;          // Fee if voucher is not consumed
}
```

### Fee Calculation

```solidity
function calculateFee(
    AtomicSwapFeeRule memory feeRule,
    uint256 requestCreatedAt,
    uint256 currentTime,
    uint256 assetAmount
) internal pure returns (uint256 fee) {
    uint256 elapsed = currentTime - requestCreatedAt;
    uint256 currentFeePercent = feeRule.startFeePercentNumerator +
        (feeRule.feeIncreasePerSecond * elapsed);

    if (currentFeePercent > feeRule.maxFeePercentNumerator) {
        currentFeePercent = feeRule.maxFeePercentNumerator;
    }

    // Fee denominator is 10000 (100% = 10000, 1% = 100, 0.01% = 1)
    fee = (assetAmount * currentFeePercent) / 10000;
}
```

### Example Timeline

```
Time    Fee%    Behavior
─────────────────────────────────────────────────────
t+0s    0.10%   Request created. Aggressive XLPs may fulfill immediately
t+10s   0.15%   Fee increases. More XLPs become profitable
t+30s   0.25%   Typical fulfillment window for liquid pairs
t+60s   0.40%   Higher fee attracts slower/smaller XLPs
t+120s  0.50%   Max fee reached (capped)
t+300s  ----    Request expires if unfulfilled
```

---

## 💰 XLP Profitability

A rational XLP will issue a voucher if and only if their expected profit exceeds zero:

```
Expected Profit = F(t) - C_gas - C_capital - C_risk > 0
```

### Cost Components

| Cost | Description | Typical Range |
|------|-------------|---------------|
| `C_gas` | Gas on source (claim) + destination (validation) | $0.01 - $1.00 |
| `C_capital` | Opportunity cost of locked liquidity | ~5-15% APY prorated |
| `C_risk` | Expected loss from disputes, reorgs, failed txns | ~0.01% of volume |
| `C_inventory` | Cost of rebalancing across chains | Variable |

### Gas Cost Breakdown

```solidity
struct GasCosts {
    // Destination chain (paid during UserOp validation)
    uint256 destValidationGas;     // ~50,000 gas for voucher verification
    uint256 destExecutionGas;      // Variable based on user's callData

    // Source chain (paid when claiming)
    uint256 sourceClaimGas;        // ~80,000 gas for withdrawFromUserDeposit
    uint256 sourceOverrideGas;     // ~100,000 gas if override needed
}
```

### Capital Cost Calculation

The XLP's capital is locked from voucher issuance until source chain claim:

```
Lock Duration = T_dest_execution + T_source_finality + T_claim_delay
             ≈ 2 seconds + 15 minutes + variable
             ≈ 15-30 minutes typical
```

For a 10% APY opportunity cost:

```
C_capital = Principal × 0.10 × (LockDuration / 365 days)
         = $10,000 × 0.10 × (30 min / 525,600 min)
         = $0.057 per $10,000 for 30 minutes
```

Capital costs are typically negligible for small transactions but become significant for large volumes.

---

## ⚠️ Unspent Voucher Fee

If an XLP issues a voucher but the user never consumes it (e.g., user's destination transaction fails), the XLP can claim the `unspentVoucherFee`:

This compensates XLPs for:

- Gas spent issuing the voucher on-chain
- Opportunity cost of reserving liquidity
- Risk of users requesting vouchers they never use

### Anti-Griefing Properties

The unspent voucher fee creates a cost for users who request vouchers without intent to use them:

```
User Cost of Griefing = unspentVoucherFee × requestedAmount
XLP Cost of Being Griefed = C_gas(issue) + C_capital(reservation)
```

For the system to be griefing-resistant: `unspentVoucherFee ≥ XLP Cost of Being Griefed`

---

## 🎯 MEV Dynamics

Voucher requests create MEV (Maximal Extractable Value) opportunities that sophisticated actors can capture.

### XLP-Bundler Integration

An XLP that also operates a Bundler (or partners with one) can:

1. Observe user's `lockUserDeposit` transaction in mempool
2. Issue voucher
3. Bundle user's destination UserOp with voucher in same block
4. Achieve **1-block cross-chain swap**

```
Block N (Source Chain)          Block M (Dest Chain, ~same time)
┌─────────────────────────┐     ┌─────────────────────────────┐
│ 1. User lockUserDeposit │     │ 1. User's UserOp            │
│ 2. XLP issueVouchers    │     │    (with voucher attached)  │
│                         │     │ 2. Voucher validated        │
│                         │     │ 3. User execution           │
└─────────────────────────┘     └─────────────────────────────┘
         │                                    │
         └──────────── Same real-world ───────┘
                       time window
```

### Sandwich Attack Resistance

Unlike AMM swaps, WeissChannel transactions are resistant to sandwich attacks:

1. **No Price Impact:** The user specifies exact amounts, not a swap through a liquidity pool
2. **Private Execution:** The user's destination callData is only revealed when executed
3. **XLP Commitment:** The XLP commits to exact terms before seeing execution

However, the user's *intent* (e.g., wanting to buy a specific NFT) may be inferred from the destination chain and amount, enabling front-running of the underlying action.

### Priority Gas Auctions (PGA)

Multiple XLPs competing for the same request may engage in PGA:

```
XLP_A sees request, bids gas price G
XLP_B sees XLP_A's bid, bids G + δ
XLP_A responds with G + 2δ
...continues until one XLP's profit margin is exhausted
```

This competition benefits users (faster fulfillment) but erodes XLP margins. Sophisticated XLPs use private mempools or Flashbots-style infrastructure to avoid PGA.

---

## 📋 Fee Parameter Recommendations

### For Users (Wallet Implementations)

| Parameter | Conservative | Balanced | Aggressive |
|-----------|-------------|----------|------------|
| `startFeePercentNumerator` | 50 (0.5%) | 20 (0.2%) | 5 (0.05%) |
| `feeIncreasePerSecond` | 1 | 2 | 5 |
| `maxFeePercentNumerator` | 100 (1%) | 75 (0.75%) | 50 (0.5%) |
| `expiresAt` | +10 min | +5 min | +2 min |

- **Conservative:** Higher starting fee, filled quickly, pays premium
- **Balanced:** Market rate start, typical fill in 10-30 seconds
- **Aggressive:** Low start, may wait or fail, best price if filled

### For XLPs

XLPs SHOULD implement dynamic fee thresholds based on:

1. Current gas prices on source and destination chains
2. Available liquidity in their pools
3. Inventory balance (prefer routes that rebalance)
4. Historical dispute rates for the user/route
5. Current utilization of their capital

---

## ✅ Capital Efficiency

### O(Networks) Staking

In custodial bridges, relayers must lock collateral proportional to the value moved (e.g., $1M locked to move $1M). WeissChannels operate on a **Performance Bond** model:

- The XLP's L1 stake is a penalty bond, not a liquidity backing
- The stake scales with the number of networks: **O(Networks)**, not O(Volume)
- A single 10 ETH stake can secure unlimited volume

### The Security Inequality

The system remains secure as long as the slashed stake exceeds the utility gained from griefing:

```
S_XLP > U_grief
```

Crucially, since the XLP must provide the voucher (value) *before* claiming the user's funds, they cannot steal funds (U_theft = 0). Therefore:

```
S_XLP << V_tx
```

The stake does not need to cover the transaction value, allowing for high capital efficiency.

---

## 🔗 Related Pages

- [Architecture](architecture.md) — Protocol lifecycle and voucher flow
- [Security](security.md) — Griefing economics and attack analysis
- [Paymasters](../paymasters/README.md) — Gas abstraction in ERC-4337

