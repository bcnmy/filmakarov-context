# Chapter 12 — AbstractJS SDK: MEE 3.0.0 / STX Validator / Fusion Modes

**Knowledge Transfer Document**  
**Author:** Fil Makarov (@filmakarov)  
**Repo:** `bcnmy/abstractjs`  
**Role:** Primary author  
**PRs:** ~11  
**Related chapters:** Chapter 3 (MEE K1 Validator), Chapter 4 (STX Validator — contracts & node), Chapter 10 (Smart Sessions SDK)

---

## 1. Overview

This chapter covers the TypeScript SDK surface within `bcnmy/abstractjs` that integrates with the MEE (Modular Execution Environment) architecture — specifically the fusion mode implementations, the transition from the MEE K1 Validator to the STX Validator, and the MEE 3.0.0 protocol version.

The SDK is responsible for constructing SuperTransaction quotes, encoding them according to the active fusion mode, signing them with the appropriate scheme, and submitting them to MEE nodes. It also handles version negotiation with nodes, contract address resolution, and quote type detection.

---

## 2. Architecture Context

The MEE architecture enables cross-chain, multi-operation bundles called SuperTransactions. A user defines a set of operations across one or more chains, the MEE node quotes the execution cost, and the user authorizes the quote. The MEE node then executes the operations, with the on-chain validator verifying the user's authorization.

**Fusion modes** are the different mechanisms by which a user can authorize a SuperTransaction. Each mode defines how the user's signature is produced and what on-chain verification logic is invoked. The SDK must encode the quote and signature differently depending on which fusion mode is active.

**Evolution path:** The original MEE K1 Validator (Chapter 3) supported four modes and was deployed as a standalone contract. The STX Validator (Chapter 4) is its successor, built as part of the unified `stx-contracts` architecture. MEE 3.0.0 introduced a new protocol version with different permit mode encoding and updated type hashes. The SDK needed to support this evolution while maintaining backward compatibility during the transition period.

The SDK interfaces with MEE nodes via their quote API — it sends the desired operations, receives a quote (cost, execution plan), and then signs and submits the quote back. The node validates the signature against the on-chain validator before executing.

---

## 3. Fusion Modes

### 3a. Simple Mode

The most straightforward fusion mode. The user signs an EIP-712 typed data hash of the SuperTransaction directly. The on-chain validator recovers the signer from the EIP-712 signature and checks it against the account's authorized key.

From the SDK perspective, this involves constructing the EIP-712 domain and type definitions matching the on-chain type hashes, then using the user's signer to produce the signature. The SDK must ensure the domain separator and struct hashing match exactly what the validator expects — mismatches here cause signature validation failures on-chain.

### 3b. Permit Mode

Permit mode combines an ERC-2612 token approval with SuperTransaction authorization. The user signs a single message that both approves a token spend (to cover execution costs) and authorizes the SuperTransaction. This is gas-efficient because it avoids a separate approval transaction.

PR #187 refactored `signPermitQuote` to no longer depend on the internal `_account` object, making the signing flow more composable and testable. The MEE 3.0.0 transition (PR #184) introduced a different permit mode encoding, requiring the SDK to handle both the old and new formats based on the MEE protocol version.

### 3c. MetaMask Delegation Toolkit Mode

PR #101 added a fusion mode for MetaMask's Delegation Toolkit, which provides an alternative authorization mechanism based on EIP-7702 delegations. In this mode, instead of signing the SuperTransaction directly, the user creates a delegation via the MetaMask toolkit, and the MEE node uses this delegation as the authorization proof.

This mode required the SDK to interface with MetaMask's delegation APIs and encode the delegation data in the format the on-chain validator expects.

### 3d. Safe Smart Account Mode

PR #174 added fusion mode support for Safe smart accounts. Safe uses a different signature scheme and account architecture than Nexus, so the SDK needed a dedicated code path for constructing and signing quotes when the underlying account is a Safe.

PR #180 added the Safe typekit as a peer dependency, providing the necessary types and utilities for Safe account interaction without bundling the full Safe SDK.

### 3e. Fusion Mode Refactoring

PR #111 consolidated all fusion mode logic within the SDK. Prior to this refactor, each mode's encoding and signing was spread across multiple locations. The refactor centralized mode selection, encoding, and signing into a unified structure, making it easier to add new modes and reducing code duplication.

This refactor also established a clearer pattern for how mode-specific behavior is isolated — each mode implements a consistent interface for quote encoding and signature production, with a mode selector dispatching to the correct implementation based on the quote type.

---

## 4. MEE 3.0.0 and STX Validator SDK Integration

PR #184 is the core integration PR for MEE 3.0.0 and STX Validator support. This was a substantial change that updated the SDK to work with the new protocol version, including:

- New type hashes for SuperTransaction and meeUserOp structs, matching the updated on-chain contracts
- Different permit mode encoding for MEE 3.0.0
- Updated quote construction and signing flows to align with the STX Validator's verification logic

**STX Validator initdata for Smart Sessions** (PR #189) — when initializing Smart Sessions on an account that uses the STX Validator, the session configuration must include the validator's init data format. This PR ensured the Smart Sessions initialization flow in the SDK correctly encodes the STX Validator-specific init data.

**Quote type handling** (PR #188) — fixed tests to use the correct quote type, reflecting the MEE 3.0.0 distinction between different quote formats (e.g., sponsored vs. user-paid, permit vs. simple).

**MEE version provision to node** (PR #166) — the SDK communicates the MEE protocol version it supports to the node, allowing the node to select the correct execution and validation path. This is important during transition periods where nodes may support multiple protocol versions simultaneously.

**SC address updates** (PR #161) — updated the hardcoded smart contract addresses in the SDK to reflect the MEE 2.2.0 deployment, ensuring the SDK points to the correct on-chain validator and related contracts.

---

## 5. Key Fixes and Evolution

The fusion mode and MEE SDK surface evolved over roughly 9 months (June 2025 – February 2026):

**June 2025 — MetaMask Delegation Toolkit mode (#101):** First non-standard fusion mode added, establishing the pattern for mode extensibility.

**July 2025 — Fusion mode refactor (#111):** Consolidated all mode logic into a unified structure, cleaning up the codebase after the initial mode implementations.

**September 2025 — Payment token info refactor (#135):** Internal cleanup of how payment token information is handled within quote flows.

**November 2025 — MEE 2.2.0 address updates (#161) and version provision (#166):** Infrastructure updates for the MEE 2.2.0 deployment and version negotiation with nodes.

**December 2025 — Safe SA fusion mode (#174) and Safe typekit (#180):** Safe smart account support added, extending MEE/fusion capabilities beyond Nexus accounts.

**February 2026 — MEE 3.0.0 + STX Validator (#184, #187, #188, #189):** The major protocol version upgrade, including new encoding, refactored permit signing, STX Validator init data for Smart Sessions, and test alignment.

---

## 6. E2E Tests as Source of Truth

The abstractjs repository includes comprehensive end-to-end tests for the STX Validator and fusion mode flows. These tests exercise the full path from quote construction through signing to on-chain validation, covering all supported fusion modes and quote types.

For anyone working on this surface, the e2e tests serve as the most reliable source of truth for how to correctly use the STX Validator SDK integration. They demonstrate the expected parameter formats, signing flows, and mode-specific behaviors in runnable form, and are kept in sync with the latest contract and node changes.

---

## 7. Caveats and Known Limitations

- **Protocol version coupling.** The SDK's type hashes, encoding formats, and signing flows are tightly coupled to specific MEE protocol versions. The MEE 3.0.0 transition (PR #184) is a clear example — permit mode encoding changed between versions. During transition periods, the SDK must detect or be configured with the correct version to produce valid signatures.

- **Safe SA mode maturity.** The Safe fusion mode (#174) is newer than the Nexus-focused modes. Developers integrating with Safe accounts should pay close attention to the e2e tests and be aware that this code path has seen less production exposure.

- **Address management.** The SDK maintains hardcoded contract addresses that must be updated with each deployment (PR #161). When deploying to new chains or upgrading contracts, the SDK address mappings must be updated — stale addresses will cause transaction failures or incorrect validator selection.

---

## 8. PR Reference

| Date | Title | Link |
|------|-------|------|
| 2026-02-26 | feat: add support of stx validator initdata format to SS init | https://github.com/bcnmy/abstractjs/pull/189 |
| 2026-02-24 | chore: fix tests to use proper quote type | https://github.com/bcnmy/abstractjs/pull/188 |
| 2026-02-23 | chore: refactor signPermitQuote not to use _account | https://github.com/bcnmy/abstractjs/pull/187 |
| 2026-02-10 | feat: mee 3.0.0, stx validator support | https://github.com/bcnmy/abstractjs/pull/184 |
| 2026-01-22 | chore: add safe typekit as peer dependency | https://github.com/bcnmy/abstractjs/pull/180 |
| 2025-12-16 | feat: safe sa fusion mode | https://github.com/bcnmy/abstractjs/pull/174 |
| 2025-11-06 | feat: provide mee version to node | https://github.com/bcnmy/abstractjs/pull/166 |
| 2025-11-03 | chore: update mee 2.2.0 sc addresses | https://github.com/bcnmy/abstractjs/pull/161 |
| 2025-09-01 | chore: refactor is payment token info | https://github.com/bcnmy/abstractjs/pull/135 |
| 2025-07-01 | chore: refactor fusion modes | https://github.com/bcnmy/abstractjs/pull/111 |
| 2025-06-18 | feat: metamask delegation toolkit fusion mode | https://github.com/bcnmy/abstractjs/pull/101 |
