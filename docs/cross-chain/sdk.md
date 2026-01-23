
# SDK

!!! info "Testnet Preview"
    The SDK is under active development. APIs may change before mainnet launch.

## TypeScript SDK for building cross-chain operations.

The EIL SDK provides tools to integrate WeissChannels into wallets and dApps, allowing users to execute complex multi-chain operations with a single signature.

---

## 🔧 Installation

```bash
yarn add @eil-protocol/sdk
```

---

## 📖 Quick Start

### Example: Buy an NFT on Arbitrum using USDC on Optimism

```typescript
import { CrossChainSdk, ApproveAction, FunctionCallAction } from '@eth-infinitism/eil-sdk'

const sdk = new CrossChainSdk()
const builder = sdk.createBuilder()

const usdc = sdk.createToken('USDC', tokensFile.USDC)

const executor = await builder
  .startBatch(10n)  // Optimism chain ID
  .addVoucherRequest({
    tokens: [{ token: usdc, amount: 90n }],
    destinationChainId: 42161n,  // Arbitrum
    ref: 'voucher1'
  })
  .endBatch()
  .startBatch(42161n)  // Arbitrum chain ID
  .useVoucher('voucher1')
  .addAction(new ApproveAction({ token: usdc, amount: 90n }))
  .addAction(new FunctionCallAction({
    functionName: 'purchaseNft',
    args: [123]
  }))
  .endBatch()
  .buildAndSign()
  .execute(callback)
```

---

## 🏗️ Core Components

### CrossChainSdk

The main entry point for building cross-chain operations.

```typescript
import { CrossChainSdk } from '@eth-infinitism/eil-sdk'

const sdk = new CrossChainSdk(config?)

// Create a builder for cross-chain operations
const builder = sdk.createBuilder()

// Create a multichain token reference
const usdc = sdk.createToken('USDC', {
  10: '0x...',      // Optimism
  42161: '0x...',   // Arbitrum
  8453: '0x...'     // Base
})
```

### CrossChainBuilder

Builds a cross-chain session from multiple batches. Each batch runs on a specific chain and can create voucher requests to move tokens to other chains.

```typescript
const builder = sdk.createBuilder()

builder
  .useAccount(smartAccount)   // Set the smart account
  .startBatch(chainId)        // Start a batch on a chain
  .endBatch()                 // End the current batch
  .buildAndSign()             // Build and sign all batches
```

### BatchBuilder

Builds operations within a single chain batch.

```typescript
builder
  .startBatch(10n)  // Optimism
  .addVoucherRequest({
    tokens: [{ token: usdc, amount: 100n }],
    destinationChainId: 42161n,
    ref: 'myVoucher'
  })
  .addAction(new TransferAction({ ... }))
  .useVoucher('myVoucher')  // On destination chain
  .endBatch()
```

---

## 🎬 Actions

Actions represent operations to execute within a batch.

### TransferAction

Transfer tokens to an address.

```typescript
new TransferAction({
  token: usdc,
  to: recipientAddress,
  amount: 100n
})
```

### ApproveAction

Approve a spender for ERC-20 tokens.

```typescript
new ApproveAction({
  token: usdc,
  spender: contractAddress,
  amount: 100n
})
```

### FunctionCallAction

Call an arbitrary contract function.

```typescript
new FunctionCallAction({
  contract: nftContract,
  functionName: 'mint',
  args: [tokenId, quantity]
})
```

### CallAction

Low-level call with raw calldata.

```typescript
new CallAction({
  to: contractAddress,
  data: encodedCalldata,
  value: 0n
})
```

---

## 🎫 Voucher Requests

Voucher requests enable cross-chain token transfers. Create a request on the source chain, then consume it on the destination chain.

### Creating a Voucher Request

```typescript
builder
  .startBatch(10n)  // Source: Optimism
  .addVoucherRequest({
    tokens: [
      { token: usdc, amount: 100n },
      { token: weth, amount: 1000000000000000000n }  // 1 ETH
    ],
    destinationChainId: 42161n,  // Destination: Arbitrum
    ref: 'myVoucher'             // Reference name
  })
  .endBatch()
```

### Using a Voucher

```typescript
builder
  .startBatch(42161n)  // Destination: Arbitrum
  .useVoucher('myVoucher')  // Consume the voucher
  .addAction(...)           // Use the received tokens
  .endBatch()
```

---

## ⚙️ Configuration

### CrossChainConfig

```typescript
const config: CrossChainConfig = {
  // Chain RPC endpoints
  chains: {
    10: 'https://optimism-rpc.example.com',
    42161: 'https://arbitrum-rpc.example.com'
  },

  // Expiration time for voucher requests (seconds)
  expireTimeSeconds: 300,

  // Fee configuration
  feeConfig: {
    startFeePercent: 0.1,      // 0.1%
    maxFeePercent: 1.0,        // 1%
    feeIncreasePerSecond: 0.01,
    unspentVoucherFeePercent: 0.05
  },

  // XLP selection policy
  xlpSelectionConfig: {
    minXlps: 1,
    maxXlps: 5,
    depositReserveFactor: 1.1,  // 10% buffer
    includeBalance: true
  }
}

const sdk = new CrossChainSdk(config)
```

---

## 🔄 Execution Flow

### 1. Build the Operation

```typescript
const builder = sdk.createBuilder()
  .useAccount(smartAccount)
  .startBatch(sourceChain)
  .addVoucherRequest({ ... })
  .endBatch()
  .startBatch(destChain)
  .useVoucher('ref')
  .addAction(new TransferAction({ ... }))
  .endBatch()
```

### 2. Sign and Execute

```typescript
const executor = await builder.buildAndSign()

// Execute with callback for status updates
await executor.execute((status) => {
  console.log('Status:', status.phase, status.chainId)
})
```

### 3. Monitor Progress

The callback receives status updates as the operation progresses:

```typescript
interface ExecutionStatus {
  phase: 'pending' | 'submitted' | 'confirmed' | 'failed'
  chainId: bigint
  txHash?: string
  error?: Error
}
```

---

## 🧪 Testing

The SDK includes testnet utilities for local development:

```typescript
import { debugSetTokenAmount } from '@eth-infinitism/eil-sdk/testnet'

// Mint test tokens on a forked network
await debugSetTokenAmount(
  client,
  tokenAddress,
  recipientAddress,
  amount
)
```

---

## 🔗 Related Pages

- [Architecture](architecture.md) — Protocol lifecycle and data structures
- [Fees & Economics](fees-and-economics.md) — Fee configuration explained
- [Smart Accounts](../smart-accounts/README.md) — ERC-4337 account abstraction

