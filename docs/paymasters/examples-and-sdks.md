
# Examples and SDKs

## Real-world Paymaster implementations and developer tooling.

This page highlights production-grade Paymasters and SDKs across the ecosystem. These projects illustrate real-world implementations, from verifying Paymasters to ERC-20 token gas payment infrastructure. 

> **⚖️ Disclaimer:** The following list is sorted alphabetically and does not reflect any ranking or order of importance. 

---

## 🔧 Alchemy

[Alchemy](https://www.alchemy.com/docs/gas-manager-services) provides:

- Simplified paymaster flow in their AA SDK
- Support for pay-per-use and freemium patterns
- Integrated bundling and sponsorship

---

## 🔧 Candide

[Candide](https://www.candide.dev/instagas) provides robust, feature-rich Paymaster infrastructure to simplify gas payments:

- Verifying Paymaster with custom, flexible gas policies
- Easy Setup of gas sponsorship with a no-code interface
- ERC-20 Paymaster that enables gas payments using stablecoins and other popular tokens 
- Developer-Focused with generous free tier on Mainnets and a full `UserOperation` simulation tooling

Candide delivers one of the most complete Paymaster solutions for developers building on account abstraction.

---

## 🔧 Circle (USDC Paymaster)

[Circle](https://www.circle.com/paymaster) announced a Paymaster that allows users to pay gas in **USDC**:

- Native stablecoin integration
- Focused on enterprise and wallet UX
- Currently live on mainnet

This makes stablecoin-based UX far more accessible.

---

## 🔧 Etherspot

[Etherspot](https://etherspot.io/arka-paymaster/) includes:

- Paymaster-as-a-service with support for verifying and quota-based modes
- SDK integrations for onboarding and token support
- Rollup-friendly architecture

Their system integrates seamlessly with their smart wallet framework.

---

## 🔧 Openfort

[Openfort](https://www.openfort.io/docs/development/gas-sponsorship) provides a production-grade Paymaster with full EntryPoint v0.9 support:

- **Parallelizable signing** — user and Paymaster sign independently for lower latency
- **Verifying Mode** with a flexible policy engine (contract, function, time, gas budget rules)
- **ERC-20 Mode** with 8 sub-modes including pre-funding, constant fees, and recipient routing
- Pay gas in stablecoins (USDC, USDT) or any ERC-20 token with off-chain exchange rates
- [Open-source contracts](https://github.com/openfort-xyz/openfort-contracts/tree/main/contracts/paymaster/PaymasterV3EPv9) for full transparency

Openfort's v0.9 Paymaster architecture removes the Paymaster from the critical signing path, enabling faster UX and independent scaling of auth and sponsorship services.

---

## 🔧 Pimlico

[Pimlico](https://docs.pimlico.io/references/paymaster) offers a robust Paymaster infrastructure:

- Verifying Paymaster with off-chain signed approvals
- ERC-20 Paymaster for gas-in-token UX
- Free and sponsored tiers
- Full `UserOperation` simulation tooling

Pimlico is among the most battle-tested paymaster providers in ERC-4337 today.

---

## 🔬 Visa Paymaster Experiment

[Visa](https://usa.visa.com/solutions/crypto/paying-blockchain-gas-fees-with-card.html) conducted an experiment to sponsor gas payments using a Visa card:

- Uses an off-chain verifier for authorization
- Settles fees via card-based payment rails
- Not production-ready, but a strong signal of ecosystem interest

While not dev-ready, it underscores major institutional exploration.

---

## 🔧 ZeroDev

[ZeroDev](https://docs.zerodev.app/) includes:

- Fully integrated Paymaster support in their AA SDK
- Gasless onboarding templates
- Pluggable backend logic for custom rules

Good fit for developers seeking end-to-end smart wallet + paymaster integration.

---

## ✅ Summary

From token-based gas models to off-chain verified sponsorship, Paymasters are becoming production-grade thanks to services like Pimlico, Etherspot, Openfort, and Circle. Choose your tooling based on how much control, abstraction, or UX fidelity you need.
