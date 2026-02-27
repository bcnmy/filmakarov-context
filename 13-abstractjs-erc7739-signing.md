# Chapter 13 — AbstractJS SDK: ERC-7739 Typed Data Signing

**Knowledge Transfer Document**  
**Author:** Fil Makarov (@filmakarov)  
**Repo:** `bcnmy/abstractjs`  
**Role:** Author  
**PRs:** ~3  
**Related chapters:** Chapter 8 (ERC-7739 Integration — contracts), Chapter 1 (Nexus Smart Account), Chapter 10 (Smart Sessions SDK)

---

## 1. Overview

This chapter covers the SDK support for ERC-1271 contract signature verification from Nexus accounts, with ERC-7739 nested typed data signing as the default mechanism. The work spans three PRs that progressively built out a complete signing surface on the `toNexusAccount` object, culminating in PR #175 which established the current architecture where signing responsibility lives at the validator level with automatic ERC-7739 detection.

This is the SDK counterpart to the on-chain ERC-7739 integration described in Chapter 8. While Chapter 8 covers how the Nexus validators verify ERC-7739 signatures on-chain, this chapter covers how the SDK produces those signatures correctly.

---

## 2. The ERC-1271 Signing Problem for Smart Accounts

When a smart account needs to "sign" something — for example, approving a token permit, placing an off-chain order on a DEX, or authorizing a session — the signature must ultimately be verifiable on-chain via ERC-1271's `isValidSignature(bytes32 hash, bytes signature)` method.

For EOAs this is straightforward: the EOA signs the hash directly. For smart accounts, the challenge is that the account itself doesn't have a private key — it delegates signing to its active validator module, which holds or verifies the signer's key. This introduces two problems:

**Replay across accounts.** If a smart account simply forwards the raw hash to its validator for signing, the same signature could potentially be valid on a different smart account that uses the same validator and signer. The signature needs to be bound to the specific account.

**Domain separation.** EIP-712 typed data already includes domain separation, but the domain is typically the dApp's domain, not the smart account's. The smart account needs to add its own domain context to prevent cross-account replay.

ERC-7739 solves both problems by wrapping the original typed data in a nested EIP-712 envelope. The outer envelope's domain is the smart account itself, and the inner content is the original typed data the dApp requested. This produces a signature that is bound to both the dApp's domain and the specific smart account, preventing replay across accounts while remaining compatible with the standard ERC-1271 verification interface.

---

## 3. Signing Modes on the Nexus Account Object

The `toNexusAccount` object exposes multiple signing paths to accommodate different use cases and validator capabilities. Understanding which path to use is essential for correct ERC-1271 integration.

### 3a. Vanilla ERC-1271 Signing

Standard `signMessage` and `signTypedData` methods that produce signatures without ERC-7739 wrapping. These are used when interacting with validators that do not support ERC-7739, or when the verifying contract expects a plain ERC-1271 signature.

In this mode, the signer signs the hash directly (for personal messages) or the EIP-712 struct hash (for typed data), and the resulting signature is passed to the validator's `isValidSignatureWithSender` method without any nested envelope.

### 3b. ERC-7739 Signing

The `signMessageErc7739` and `signTypedDataErc7739` methods produce signatures wrapped in the ERC-7739 nested envelope. These methods:

1. Take the original message or typed data from the dApp
2. Wrap it in an outer EIP-712 typed data structure where the domain is the smart account (the Nexus contract address and its chain)
3. Have the signer sign the outer envelope
4. Return the signature in the format the on-chain ERC-7739 validator expects

These methods are exposed at the validator level and used by the Nexus account under the hood.

### 3c. Default Behavior

After PR #175, the Nexus account defaults to using ERC-7739 signing whenever the current validator supports it. This means developers using the standard `signMessage` and `signTypedData` on the Nexus account object will automatically get ERC-7739 wrapped signatures if the active validator has 7739 support — no explicit opt-in required.

This is the recommended path for most integrations. Developers only need to think about vanilla vs. 7739 signing if they are working with a validator that does not support 7739, or if the verifying contract specifically requires a non-7739 signature format.

### 3d. Validator-Level Signing

PR #175 moved the signing logic from the account level down to the validator entities. This was a deliberate architectural decision: different validators have different capabilities regarding ERC-7739. The K1 Default Validator and MEE K1 Validator support ERC-7739, while ownable validators and Smart Sessions validators do not.

By placing signing at the validator level, the SDK can dispatch to the correct signing path based on which validator is active on the account. The Nexus account's `signMessage` and `signTypedData` methods delegate to the current validator's signing implementation, which knows whether to apply 7739 wrapping or not.

This means the signing behavior is consistent regardless of which validator the account is using — the developer calls the same methods on the Nexus account, and the validator handles the details.

---

## 4. Validator and WalletClient Requirements

PR #175 introduced a breaking change in how validators are constructed:

- **`toDefaultModule` and `toMeeK1Module`** now require a `WalletClient` object instead of a `signer` object. This is because ERC-7739 signing requires the ability to sign EIP-712 typed data, which the viem `WalletClient` interface provides via `signTypedData`. A raw signer (private key or account) does not always expose this capability in the way needed for the nested envelope construction.

- **Ownable validators and Smart Sessions modules** still accept a `signer` object, since they do not support ERC-7739 and therefore do not need the `WalletClient` typed data signing capability.

- **`toValidator()`** (the generic validator constructor) accepts either a `WalletClient` or a `signer`, adapting its behavior based on what is provided.

This distinction reflects a real capability difference: producing an ERC-7739 signature requires signing a nested EIP-712 structure, which in turn requires the signer to support `eth_signTypedData_v4`. The `WalletClient` abstraction in viem guarantees this capability, while a bare signer may not.

---

## 5. 7739 Version Detection

PR #175 also introduced automatic detection of whether the active validator supports ERC-7739. When the Nexus account is initialized, the SDK checks the validator's capabilities and records whether 7739 signing should be used by default.

This detection is important because it allows the SDK to make the right signing choice without developer intervention. If a developer switches the active validator on their Nexus account (e.g., from the K1 Default Validator to a Smart Sessions validator), the signing behavior automatically adjusts — 7739 wrapping is applied when supported and skipped when not.

---

## 6. Simple Mode Typed Data Signing

PR #164 added EIP-712 typed data signing for MEE simple (non-fusion) mode. In this mode, the user interacts with the MEE system using standard smart account signing — the SuperTransaction is authorized via an EIP-712 signature rather than through a fusion mechanism (on-chain transaction, permit, or delegation).

This PR ensured that when a Nexus account signs typed data for an MEE simple mode quote, the signature is produced with the correct EIP-712 domain and type definitions matching what the on-chain validator expects. This is distinct from the ERC-7739 work (which is about ERC-1271 verification for dApp interactions) — here the typed data signing is used for MEE authorization.

---

## 7. Caveats and Known Limitations

- **WalletClient requirement is a breaking change.** Any code that was constructing `toDefaultModule` or `toMeeK1Module` with a raw signer needs to be updated to provide a `WalletClient`. This affects all integrations that use these validators. The migration path is to wrap the signer in a viem `WalletClient` via `createWalletClient`.

- **7739 support is validator-dependent.** Not all validators support ERC-7739. Developers building custom validators must explicitly implement the 7739 verification logic on-chain (see Chapter 8) and signal support in the SDK for auto-detection to work. If a custom validator supports 7739 but the SDK doesn't know about it, the default behavior will fall back to vanilla signing.

- **Nested envelope complexity.** The ERC-7739 nested envelope adds an extra layer of EIP-712 encoding. When debugging signature verification failures, developers need to check both the inner typed data (the dApp's original request) and the outer envelope (the smart account's domain binding). A mismatch at either level will cause `isValidSignature` to return failure.

- **Chain-specific domain.** The ERC-7739 outer envelope uses the Nexus account's address and chain ID as its EIP-712 domain. This means a 7739 signature is chain-specific — a signature produced for a Nexus account on Base cannot be used for the same account on Optimism. This is by design (preventing cross-chain replay), but developers building multichain dApps should be aware that signatures must be produced per-chain.

---

## 8. PR Reference

| Date | Title | Link |
|------|-------|------|
| 2025-12-17 | feat: add proper 7739 data signing capabilities to Nexus object | https://github.com/bcnmy/abstractjs/pull/175 |
| 2025-11-05 | feat: typed data sign for simple (smart-account) mode | https://github.com/bcnmy/abstractjs/pull/164 |
| 2025-06-06 | feat: nexus 1.0.2 deployment | https://github.com/bcnmy/abstractjs/pull/86 |
