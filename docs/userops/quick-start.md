# Quick Start Guide

Get up and running with EntryPoint v0.9 and 7702 UserOperations in under 5 minutes! This guide will walk you through with installing the necessary dependencies and creating bundler client.

## Installation

### Install Required Packages

```bash
npm i @etherspot/free-bundler
# or
yarn add @etherspot/free-bundler
# or
bun add @etherspot/free-bundler
```

## Getting Started

### 1. Get list of all supported networks (including testnets)
```typescript
import {getSupportedChainIds} from '@etherspot/free-bundler';

const supportedChainIds = getSupportedChainIds();
```

### 2. Get common bundler and rpc url for a given chain id
```typescript
import { getFreeBundlerUrl } from '@etherspot/free-bundler';

const chainId = 1;
const bundlerAndRpcUrl = getFreeBundlerUrl(chainId);
```

### 3. Create bundler client
Refer [here](https://viem.sh/account-abstraction/actions/bundler/estimateUserOperationGas) for more actions using bundler client which involves eth_estimateUserOperationGas, eth_sendUserOperation etc.
```typescript
import { createFreeBundler } from '@etherspot/free-bundler';
import { publicActions, walletActions } from 'viem';
import { mainnet } from "viem/chains";

const chain = mainnet;
const bundlerClient = createFreeBundler({chain});

// optionally extend the client to use public and wallet actions
const commonClient = createFreeBundler({chain})
                      .extend(publicActions)
                      .extend(walletActions);
```

1. **[Constructing UserOps](../userops/basic.md)** - Create and Send UserOp

---
