
# Types of Paymasters

## Explore different Paymaster patterns in ERC-4337, from whitelists to token gas payments.

Paymasters in ERC-4337 can be implemented in various ways depending on how gas is sponsored and what validation logic is required. Below are the most common patterns.

---

## ✅ Whitelist Paymaster

Simplest model. The Paymaster maintains a list of allowed users or operations.

**Use case:** Free gas for early users, partners, or test campaigns.

**How it works:**

- Checks the sender address or calldata
- Accepts or reverts in `validatePaymasterUserOp`
- Optionally restricts frequency or nonce range

---

## 🔐 Verifying Paymaster (Off-Chain Signed)

Requires an off-chain component to approve each transaction.

**Use case:** Controlled sponsorship via API, rate limits, user-specific logic

**How it works:**

- User gets a signed message from a backend
- `paymasterAndData` includes the signature and metadata
- Contract verifies the signature before approving sponsorship

This model is extensible and secure — popular in production wallets.

---

## 💰 ERC-20 Paymaster (Token Paymaster)

Lets users pay for gas in an ERC-20 token, which the Paymaster converts to ETH.

**Use case:** Gasless UX for apps where users hold tokens but no ETH

**How it works:**

- Contract checks token balance or transfer approval
- Performs a token-to-ETH swap off-chain or via internal pool
- Uses ETH deposit in EntryPoint to cover the gas

Caution: swaps can be complex and require slippage/timing protection.

---

## 🔗 CrossChainPaymaster

Enables atomic cross-chain execution for Smart Accounts via the [WeissChannels](../cross-chain/README.md) protocol.

**Use case:** Single-signature transactions that span multiple L2 chains

**How it works:**

- User locks funds on the source chain via the CrossChainPaymaster
- XLP (Crosschain Liquidity Provider) issues a voucher for destination chain funds
- User executes on destination chain, consuming the voucher
- XLP claims locked funds on source chain

This model enables seamless cross-chain transfers and multichain operations without custodial bridges. See [Cross-Chain Architecture](../cross-chain/architecture.md) for details.

---

## 🧩 Hybrid and Custom Patterns

- **Quota-based Paymasters:** Allow N txs per user/day
- **Multi-condition checks:** Combine whitelist + signatures
- **Action-specific Paymasters:** Only subsidize known `callData`

The Paymaster model is extremely flexible. The key constraint is ensuring `validatePaymasterUserOp()` remains **safe, bounded, and deterministic**.

---

## ✅ Summary

Paymasters enable powerful UX flows, from gasless onboarding to pay-in-token systems. Choose a pattern based on your app’s risk tolerance, business model, and infrastructure capabilities.
