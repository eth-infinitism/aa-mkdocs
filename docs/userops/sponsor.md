#Sponsored Userop

This guide will help understanding how to broadcast a [7702](https://eip7702.io/) sponsored userop using smart wallet implemented [here](https://github.com/eth-infinitism/account-abstraction/blob/develop/contracts/accounts/Simple7702Account.sol) using free bundler client and a paymaster client.

### 1. Create Smart Account Client
=== "account.ts"
    ```ts
    import { commonClient } from './client'
    import { toSimple7702SmartAccount } from 'viem/account-abstraction'
    import { privateKeyToAccount } from 'viem/accounts'
    const owner = privateKeyToAccount('0x...') // add private key here

    export const smartAccount = await toSimple7702SmartAccount({
        implementation: "0xa46cc63eBF4Bd77888AA327837d20b23A63a56B5", // simple7702Account for ep9
        client: commonClient,
        owner,
    });
    ```
=== "client.ts"
    ```ts
    import { createFreeBundler } from '@etherspot/free-bundler'
    import { publicActions, walletActions } from 'viem'
    import { mainnet } from 'viem/chains'
    const chain = mainnet

    export const commonClient = createFreeBundler({chain})
                                    .extend(publicActions)
                                    .extend(walletActions)
    ```

### 2. Create Paymaster client
Refer [here](https://viem.sh/account-abstraction/clients/paymaster) for more options and detailed docs for paymaster client
=== "paymaster.ts"
    ```ts
    import { createPaymasterClient } from "viem/account-abstraction"

    export const paymasterClient = createPaymasterClient({
        transport: http("") // add paymaster url here
    });

    // NOTE: this can change according to the paymaster implementation.
    // use the corresponding paymaster's docs to understand what has to used for context.
    export const paymasterContext = { policyId: "" } // add policy id here 
    ```

### 3. Send Userop
=== "example.ts"
    ```ts
    import { commonClient } from './client'
    import { toSimple7702SmartAccount } from 'viem/account-abstraction'
    import { privateKeyToAccount } from 'viem/accounts'
    import { SignAuthorizationReturnType } from 'viem'
    import { paymasterClient, paymasterContext } from './paymaster'

    const owner = privateKeyToAccount('0x...') // add private key here

    const smartAccount = await toSimple7702SmartAccount({
        implementation: "0xa46cc63eBF4Bd77888AA327837d20b23A63a56B5", // simple7702Account for ep9
        client: commonClient,
        owner,
    })

    // overriding for ep9 address
    smartAccount.entryPoint.address = "0x433709009B8330FDa32311DF1C2AFA402eD8D009"
    console.log("wallet:: ", smartAccount.address)

    // check sender's code to decide if eip7702Auth tuple is necessary for userOp.
    const senderCode = await commonClient.getCode({
        address: smartAccount.address
    })

    let authorization: SignAuthorizationReturnType | undefined
    const { address: delegateAddress } = smartAccount.authorization

    if(senderCode !== `0xef0100${delegateAddress.toLowerCase().substring(2)}`) {
        authorization = await commonClient.signAuthorization(smartAccount.authorization)
    }

    const userOpHash = await commonClient.sendUserOperation({
        account: smartAccount,
        authorization,
        calls: [
            {to: "0x09FD4F6088f2025427AB1e89257A44747081Ed59", value: parseUnits('0.0000001', 18)}
        ],
        paymaster: paymasterClient,
        paymasterContext: paymasterContext ? JSON.parse(paymasterContext) : undefined,
    });

    console.log('userOpHash:: ', userOpHash)
    ```
=== "client.ts"
    ```ts
    import { createFreeBundler } from '@etherspot/free-bundler'
    import { publicActions, walletActions } from 'viem'
    import { mainnet } from 'viem/chains'
    const chain = mainnet

    export const commonClient = createFreeBundler({chain})
                                    .extend(publicActions)
                                    .extend(walletActions)
    ```
=== "paymaster.ts"
    ```ts
    import { createPaymasterClient } from "viem/account-abstraction"

    export const paymasterClient = createPaymasterClient({
        transport: http("") // add paymaster url here
    });

    // NOTE: this can change according to the paymaster implementation.
    // use the corresponding paymaster's docs to understand what has to used for context.
    export const paymasterContext = { policyId: "" } // add policy id here 
    ```
[![Open in StackBlitz](https://developer.stackblitz.com/img/open_in_stackblitz.svg)](https://stackblitz.com/github/etherspot/free-bundler?file=examples%2Findex.ts)

After opening StackBlitz, run:
```bash
npx tsx examples/index.ts \
--private-key 0x... \
--paymaster-url 'https://....' \
--paymaster-context ''
```
<small><em>For more run options, see the <a href="https://github.com/etherspot/free-bundler/blob/master/examples/USAGE.md">GitHub examples usage</a>.</em></small>