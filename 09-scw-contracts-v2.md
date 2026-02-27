# Chapter 9 — SCW Contracts v2 (Legacy Biconomy Smart Account)

**Knowledge Transfer Series — Fil Makarov (@filmakarov)**
**Status:** Legacy / Superseded by Nexus (ERC-7579)
**Repo:** `bcnmy/scw-contracts`

---

## Purpose

SCW Contracts v2 was Biconomy's production smart contract wallet — an ERC-4337 compliant smart account that served as the company's flagship account abstraction product before the transition to Nexus. It was the system that real users interacted with in production.

The architecture drew from Gnosis Safe and Argent wallet designs, and was also compliant with ERC-6900. It provided a modular validation layer, session key management, and passkey support — many of which were conceptual predecessors to features later rebuilt in the ERC-7579 Nexus architecture.

This document exists as a **historical reference** for anyone who needs to understand, maintain, or reason about the legacy codebase, including migration paths for users still on SCW v2.

---

## Architecture & Structure

SCW v2 follows a **UUPS upgradeable singleton + proxy** pattern. All user accounts are lightweight proxies that `delegatecall` into a shared implementation contract.

### Contract Hierarchy

```
┌─────────────────────────────────────────────────┐
│                  User Account                    │
│              (Proxy.sol instance)                 │
│         delegatecalls to SmartAccount ↓          │
├─────────────────────────────────────────────────┤
│              SmartAccount.sol                     │
│        (UUPS upgradeable singleton)              │
│   Co-authored by Chirag Titiya & Fil Makarov     │
├──────────┬──────────┬───────────────────────────┤
│ Module   │ Fallback │ Executor.sol               │
│ Manager  │ Manager  │ (calls/delegatecalls       │
│ .sol     │ .sol     │  to dApp contracts)        │
└──────────┴──────────┴───────────────────────────┘
        ▲
        │ implements
┌───────┴─────────────────────────────────────────┐
│           BaseSmartAccount.sol                    │
│    (Abstract — EIP-4337 IAccount interface)       │
└─────────────────────────────────────────────────┘
```

### Core Contracts

**SmartAccount.sol** — The primary implementation contract. All user accounts are Proxy instances that delegatecall into this singleton. It is UUPS-upgradeable, meaning the implementation address can be updated through a self-call. Co-authored by Chirag Titiya and Fil Makarov (listed in NatSpec).

**BaseSmartAccount.sol** — Abstract contract implementing the EIP-4337 `IAccount` interface. Defines the `validateUserOp` signature and the interaction surface with the ERC-4337 EntryPoint.

**SmartAccountFactory.sol** — Factory for deploying Smart Account proxies via CREATE2, providing deterministic counterfactual addresses.

**Proxy.sol** — Lightweight UUPS proxy. Each user account is an instance of this contract, storing only the implementation address and delegating all logic to SmartAccount.sol.

**ModuleManager.sol** — Gnosis Safe-style module management. Modules are external contracts that can execute transactions on behalf of the smart account. The manager maintains a linked list of enabled modules.

**FallbackManager.sol** — Manages a delegate-call fallback handler, allowing the smart account to respond to calls for function signatures not defined on SmartAccount.sol itself.

**Executor.sol** — Helper contract providing internal methods for `call` and `delegatecall` execution to external dApp contracts.

---

## Core Modules

Filio authored and maintained five validation/session modules for SCW v2:

### EcdsaOwnershipRegistryModule

The core ownership module. Maps each Smart Account address to its EOA owner address via a registry pattern. Validates UserOps by recovering the signer from the ECDSA signature and checking it against the registered owner. Also implements EIP-1271 `isValidSignature` for off-chain signature verification. Only EOAs can authorize transactions through this module.

### PasskeyRegistryModule

WebAuthn/Passkey validation module using the P-256 (secp256r1) curve. Enables users to authenticate smart account operations using device biometrics (fingerprint, face) or hardware security keys via the WebAuthn standard. This was an early production implementation of passkey-based smart account authentication.

### SessionValidationModule

Session key management — the direct predecessor to the ERC-7579 Smart Sessions module (Chapter 2). Allowed granting temporary, scoped permissions to session keys so that dApps could execute transactions on behalf of the user within defined constraints without requiring the owner's signature each time.

### MultichainECDSAValidator

Cross-chain ECDSA validation, enabling signature schemes that could authorize operations across multiple chains.

### BatchedSessionRouterModule

Routing layer for batched session validations. Allowed multiple session-authorized operations to be validated and executed in a single batched transaction.

---

## Design Choices

**UUPS over Transparent Proxy:** The UUPS pattern was chosen so that upgrade logic lives in the implementation contract rather than the proxy, keeping proxies minimal and gas-efficient.

**Gnosis Safe-style Module Management:** The linked-list module pattern (ModuleManager) was borrowed from Gnosis Safe's battle-tested design. This gave a familiar, audited pattern for enabling/disabling modules.

**Registry-based Ownership:** Rather than storing the owner directly on each proxy, the EcdsaOwnershipRegistryModule uses a mapping from smart account address to owner EOA. This keeps the proxy storage clean and moves ownership into the module layer.

**Singleton + Proxy Architecture:** A single SmartAccount implementation serves all users. This minimizes deployment costs (each user only deploys a lightweight proxy) and allows all users to be upgraded simultaneously by updating the singleton.

---

## Audit Remediations (Feb 2024)

The two PRs within the knowledge transfer period represent the tail end of the SCW v2 security lifecycle:

- **PR #196 — Aggregated Audit Remediations:** Combined fixes from multiple security audits, addressing findings across the core contracts.
- **PR #197 — SMA-379 ABI SVM Audit Remediations pt 1:** Specific audit findings for the Session Validation Module (SVM), addressing ABI encoding and validation edge cases.

These were among the final maintenance touches before engineering focus shifted fully to Nexus.

---

## Caveats & Known Limitations

**Superseded architecture.** SCW v2 predates ERC-7579 and does not implement the modular account standard that became the industry direction. Module interfaces are Biconomy-specific rather than standardized.

**Module interoperability.** Unlike ERC-7579 modules (which work across any compliant account), SCW v2 modules are tightly coupled to the Biconomy account implementation.

**Upgrade path.** Users on SCW v2 need a migration to Nexus. The `abstract-docs` repo contains v2-to-Nexus upgrade documentation with initdata generation snippets (see abstract-docs PR #33).

**Prior work not fully captured.** Filio's contributions to this repo predate the February 2024 PR tracking period. The EcdsaOwnershipRegistryModule, PasskeyRegistryModule, and other modules were authored earlier — only the final audit remediations fall within the tracked window.

---

## Transition to Nexus

SCW v2 → Nexus → stx-contracts represents the complete evolution of Biconomy's smart account architecture during Filio's tenure.

The move to Nexus was motivated by the emergence of ERC-7579 as an industry standard for modular smart accounts, enabling cross-vendor module compatibility. Many architectural patterns from SCW v2 — session keys, ownership modules, passkey validation, proxy-based deployment — were redesigned and rebuilt in the ERC-7579 paradigm:

| SCW v2 | Nexus (ERC-7579) |
|--------|-------------------|
| SessionValidationModule | Smart Sessions Module (Ch. 2) |
| EcdsaOwnershipRegistryModule | K1 Validator |
| PasskeyRegistryModule | P256 Validator modes |
| Gnosis Safe-style ModuleManager | ERC-7579 module types (Validator, Executor, Hook, Fallback) |
| Custom module interfaces | Standardized ERC-7579 interfaces |
| ERC-6900 compliance | ERC-7579 compliance |

---

## Useful Links

| Resource | Link |
|----------|------|
| Repository | `bcnmy/scw-contracts` |
| PR #196 — Aggregated Audit Remediations | https://github.com/bcnmy/scw-contracts/pull/196 |
| PR #197 — SMA-379 ABI SVM Audit Remediations | https://github.com/bcnmy/scw-contracts/pull/197 |
| v2-to-Nexus Upgrade Docs | `bcnmy/abstract-docs` PR #33 |
