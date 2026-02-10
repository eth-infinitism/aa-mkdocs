# Paymaster Signature

## Feature overview

Starting with ERC-4337 version 0.9,
it is possible to include a paymaster signature in the `paymasterAndData` field of the `PackedUserOperation`.

This paymaster signature is not included when calculating the `UserOperation` hash, same as the `signature` field,
which allows for the Account and the Paymaster to sign the rest of the `UserOperation` in parallel.

## Generating a Paymaster Signature

1. Create a `paymasterAndData` bytes array as usual.
2. Append the `PAYMASTER_SIG_MAGIC` (`0x22e325a297439656`) signature marker to the end of the `paymasterAndData` array.
3. Build the rest of the `UserOperation` and send it for signing to the Account and the Paymaster services in parallel.
4. Set the `signature` field of the `UserOperation` to the result of the Account signature as usual.
5. Calculate the size of the Paymaster Signature and store it in a `uint16 paymasterSignatureSize` variable.
6. Replace the `paymasterAndData` field with the value `abi.encodePacked(paymasterAndData, paymasterSignatureSize, PAYMASTER_SIG_MAGIC)`.

## Summary of `paymasterAndData` format

The `paymasterAndData` field in its most complex form is a byte array with the following format:

```
    paymasterAddress(20) || verificationGasLimit(16)  || postOpGasLimit(16) || paymasterData ||
    paymasterSignature   || paymasterSignatureSize(2) || PAYMASTER_SIG_MAGIC (0x22e325a297439656)`
```
