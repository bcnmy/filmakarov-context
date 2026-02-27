# Chapter 4 — STX Validator

**Knowledge Transfer Document**  
**Author:** Fil Makarov (@filmakarov)  
**Repos:** `bcnmy/stx-contracts`, `bcnmy/abstractjs`, `bcnmy/mee-node`  
**Role:** Primary author  
**Period:** November 2025 – February 2026

---

## 1. Purpose

The STX Validator is a next-generation ERC-7579 compatible validator module designed to validate MEE (Modular Execution Environment) SuperTransactions. It replaces the legacy K1MeeValidator (see Chapter 3) with a fundamentally different architecture.

The core problem with K1MeeValidator was that it was **tightly coupled to secp256k1 (EOA) signatures**. Every validation mode — Simple, Permit, Tx, Safe — had its own library with signature verification logic baked in. Adding a new signing scheme (like passkeys) meant duplicating or heavily modifying every mode.

StxValidator solves this by introducing a **signature-agnostic, two-layer modular architecture** that cleanly separates two concerns:

1. **"Is this UserOp part of a SuperTransaction?"** — handled by Stx Mode Verifiers
2. **"Is the signature cryptographically valid?"** — handled by Stateless Validators

This separation means new signing schemes (P256/passkeys, Safe multisig, or any future ERC-7780 compliant validator) can be added without touching any SuperTransaction mode logic, and new SuperTransaction modes can be added without touching any signature verification code.

The STX Validator powers MEE 3.0.0 and lives in the unified `bcnmy/stx-contracts` repository alongside Nexus and the Composability contracts.

---

## 2. Architecture & Structure

### Repository Location

The STX Validator lives in `bcnmy/stx-contracts` under `contracts/validators/stx-validator/`. The key files are:

```
contracts/validators/stx-validator/
├── StxValidator.sol              // Main validator — ERC-7579, ERC-7780, ERC-7739
├── ConfigManager.sol             // Per-account configuration system
├── ERC7739Validator.sol          // Nested typed data sign support
├── submodules/
│   ├── SimpleModeSubmodule.sol   // Simple mode (EIP-712 SuperTx signing)
│   ├── PermitSubmodule.sol       // Permit mode (ERC-2612)
│   ├── TxSubmodule.sol           // On-chain transaction mode
│   ├── SafeAccountSubmodule.sol  // Safe multisig mode (dual-role)
│   ├── NoStxModeVerifier.sol     // Fallback for vanilla ERC-4337/ERC-1271
│   ├── EOAStatelessValidator.sol // secp256k1 signature verification
│   └── p256/
│       ├── P256StatelessValidator.sol  // Passkey/WebAuthn verification
│       └── P256Verifier.sol            // Solidity P256 curve impl (Daimo FCL)
```

### Two-Layer Modular Design

The architecture implements a clean separation of concerns through two interfaces:

**Layer 1 — Stx Mode Verification (`IStxModeVerifier`):** Each submodule answers the question "Is this UserOp/data object part of a SuperTransaction, and can you extract the proof?" Different modes represent different ways a user can express their intent to authorize a SuperTransaction (direct signing, via a permit, via an on-chain tx, via a Safe multisig, or not at all for vanilla flows).

**Layer 2 — Signature Verification (`IStatelessValidator` / ERC-7780):** Each submodule answers the question "Given a hash and a signature, is this signature valid for this signer?" This layer is completely decoupled from SuperTransaction logic. It doesn't know or care about MEE — it just verifies cryptographic signatures statelessly.

### ConfigManager

The `ConfigManager.sol` handles per-account configuration of which `(stxModeVerifier, statelessValidator)` pairs are available. Key design elements:

**Preconfigured Signature Types:** Common combinations (e.g., EOA + Simple Mode, P256 + Simple Mode) are assigned optimized 4-byte prefixes. These are defined in `Constants.sol` and allow the validator to route to the correct submodule pair without any on-chain config lookup, saving gas.

**Custom Configs:** Accounts can enable arbitrary `(stxModeVerifier, statelessValidator)` pairs via a `configId`. This allows any future module implementing the respective interface to be plugged in without upgrading the StxValidator itself.

**Multiple Configs Per Account:** A single Nexus account can have multiple validation strategies enabled simultaneously — for example, both EOA and passkey signing with different modes.

**Ownership Data:** Per-account, per-validator storage for public keys and credentials (e.g., an EOA address for secp256k1, or a public key pair for P256).

---

## 3. Stx Mode Verifiers (Layer 1)

Each mode verifier implements `IStxModeVerifier` and represents a different way the user proves their UserOp is part of a SuperTransaction.

### Simple Mode (`SimpleModeSubmodule.sol`)

The user directly signs an EIP-712 typed data hash of the SuperTransaction struct. This is the most straightforward mode — the user's wallet presents the SuperTx for signing, and the signature is included in the UserOp.

Multichain support was added in `stx-contracts #9` (`fix: make simple mode 712 sigs multichain`) to ensure the EIP-712 domain separator doesn't prevent the same signature from being used across chains when the SuperTransaction spans multiple chains.

### Permit Mode (`PermitSubmodule.sol`)

Uses an ERC-2612 token permit where the SuperTransaction hash is embedded in the `deadline` field. The permit authorizes a token transfer from the user's account to the orchestrator's account (MEE node's Nexus), funding the orchestrator to execute the SuperTransaction on the user's behalf. The Stx hash embedded in the deadline field binds the permit to a specific SuperTransaction, preventing the orchestrator from using the permit outside of the agreed execution.

MEE 3.0.0 introduced different permit mode encoding (`mee-node #175`) to support the new architecture.

### Tx Mode (`TxSubmodule.sol`)

The user sends a dedicated on-chain Ethereum transaction (a "fusion" transaction) that transfers funds to the orchestrator's account, with the SuperTransaction hash appended to the transaction's calldata. The mode verifier extracts and validates this hash from the calldata, binding the funding transaction to the specific SuperTransaction the orchestrator will execute.

### Safe Account Mode (`SafeAccountSubmodule.sol`)

Supports Safe multisig wallets as the trigger for SuperTransactions. The user executes a Safe transaction that authorizes the SuperTx.

This submodule has a **dual role** — it implements both `IStxModeVerifier` (proving the UserOp is part of an Stx via a Safe tx) and `IStatelessValidator` (verifying the Safe multisig signature). This makes sense because Safe signature verification is inherently tied to the Safe execution context.

Added in `stx-contracts #13` (`feat: Add new fusion mode: safe smart account mode`), with corresponding mee-node support in `mee-node #156`.

### No-Stx Mode (`NoStxModeVerifier.sol`)

Fallback for scenarios where there is no SuperTransaction involved — vanilla ERC-4337 UserOps and standard ERC-1271 signature validation. This ensures the STX Validator can serve as a general-purpose validator module, not just for MEE flows. Backward compatibility with non-MEE smart account usage is preserved through this mode.

---

## 4. Stateless Validators (Layer 2)

Each stateless validator implements `IStatelessValidator` (ERC-7780) and performs pure cryptographic signature verification with no on-chain state dependency.

### EOA Stateless Validator (`EOAStatelessValidator.sol`)

Traditional secp256k1 ECDSA signature verification — the standard Ethereum EOA signing scheme. Includes signature malleability protections (fixes in `EcdsaHelperLib.sol`).

### P256 Stateless Validator (`P256StatelessValidator.sol`)

WebAuthn/Passkey support using the P256 (secp256r1) curve. This enables passwordless authentication flows where users sign with device biometrics (Touch ID, Face ID, hardware security keys).

The implementation uses a dual strategy: it first attempts to use the **RIP-7212 precompile** (a rollup improvement proposal that adds native secp256r1 verification at the EVM level for gas efficiency), and falls back to a **Solidity implementation** (`P256Verifier.sol`, based on Daimo's FCL library) on chains where the precompile is not yet available.

P256 mode support was integrated into mee-node in `mee-node #179` (`feat: p256 validator modes support`).

### Custom Validators (ERC-7780)

Any contract implementing the `IStatelessValidator` interface can be used as a signature validator. This is the extensibility point — future signing schemes (BLS, post-quantum, multi-party computation, etc.) can be added by deploying a new stateless validator contract and registering it via the ConfigManager, without any changes to StxValidator itself.

---

## 5. Validation Algorithm

### UserOp Validation Flow

When `validateUserOp` is called on the StxValidator, the following sequence occurs:

1. **Signature Type Detection:** The first bytes of `userOp.signature` contain a signature type prefix. For preconfigured types, this is a known 4-byte constant defined in `Constants.sol`. For custom configurations, it's a `configId`.

2. **Config Resolution:** The `ConfigManager` resolves the signature type prefix into a `(stxModeVerifier, statelessValidator)` pair. For preconfigured types, this is a direct mapping with no storage reads. For custom configs, it looks up the account's registered configuration.

3. **Stx Mode Verification:** The resolved `IStxModeVerifier` is called. It extracts and validates the proof that this UserOp is part of a SuperTransaction (or, in the case of `NoStxModeVerifier`, confirms it's a vanilla flow). The mode verifier returns the hash that needs to be signature-verified.

4. **Signature Verification:** The resolved `IStatelessValidator` is called with the hash from step 3 and the cryptographic signature. It performs pure signature verification (ECDSA recovery for EOA, P256 curve verification for passkeys, Safe multisig checks, etc.).

5. **Validation Result:** The combined result is returned to the EntryPoint.

### ERC-1271 / ERC-7739 Flow

For off-chain signature validation (e.g., signing messages for dApps):

1. The signature type prefix `0x177eee05` indicates a non-Stx ERC-1271 flow.
2. The `ERC7739Validator` handles nested typed data signing — wrapping the application's EIP-712 data inside the ERC-7739 envelope for secure contract signature verification.
3. The `IERC7739Multiplexer` interface routes the hash and signature to the appropriate stateless validator for cryptographic verification.
4. Safe caller detection (MulticallerWithSigner) is supported, along with a custom safe senders registry.

### ERC-7780 Flow

For arbitrary stateless validator usage, the flow is the same as above but the stateless validator is resolved from the account's custom config rather than a preconfigured type. Any contract implementing `IStatelessValidator` can participate.

---

## 6. Design Choices

### Why Two Layers Instead of Monolithic

The K1MeeValidator had four mode libraries, each containing both mode verification and signature verification logic interleaved. Adding P256 support would have required duplicating or forking every mode library. The two-layer design means N modes × M signature schemes = N + M contracts instead of N × M.

### Why Stateless Validation (ERC-7780)

Stateless validators receive everything they need as function arguments — no on-chain storage lookups for signing keys within the validator itself. This has several benefits: validators can be shared across accounts, they're cheaper to deploy and use, and they're easier to audit since they're pure functions. The ownership data (public keys) is stored per-account in the ConfigManager, not in the validators themselves.

### Why Safe Account Submodule Is Dual-Role

The Safe Account submodule implements both `IStxModeVerifier` and `IStatelessValidator` because Safe signature verification is inherently contextual — you need to verify both that the Safe transaction authorized the SuperTx (mode verification) and that the multisig signatures on the Safe transaction are valid (signature verification). Splitting these into two contracts would require passing Safe-specific context across a boundary that doesn't naturally accommodate it.

### Preconfigured vs Custom Configs

Preconfigured signature types (4-byte constants) exist purely for gas optimization — they avoid a storage read to resolve the mode verifier and stateless validator addresses. Common patterns like "EOA + Simple Mode" are used in the vast majority of transactions and benefit from this optimization. Custom configs provide the escape hatch for future extensibility without trading off gas on the hot path.

### EIP-712 for Multichain Signatures

Simple mode uses EIP-712 typed data signing of the SuperTx struct. The domain separator was initially chain-specific, which prevented a single signature from authorizing a SuperTransaction spanning multiple chains. This was fixed (`stx-contracts #9`) to make simple mode signatures multichain-compatible — a critical requirement for MEE's cross-chain execution model.

---

## 7. Usage Details

### Installing STX Validator on a Nexus Account

The STX Validator is an ERC-7579 validator module. It is installed on a Nexus account like any other module, with initialization data that specifies the initial configuration (which signature types to enable, ownership data for the signing keys).

The SDK provides helpers for generating the correct initdata format, including support for STX Validator initdata within Smart Sessions initialization (`abstractjs #189`).

### SDK Integration

Key SDK surface (in `bcnmy/abstractjs`):

- **MEE 3.0.0 + STX Validator support** (`abstractjs #184`) — core integration that replaced the K1 validator SDK surface
- **`signPermitQuote`** — refactored (`abstractjs #187`) to not rely on `_account`, making it cleaner for permit mode flows
- **Quote type detection** — the SDK properly detects and handles different quote types returned by the MEE node (`mee-node #142`)
- **STX Validator initdata format for Smart Sessions** (`abstractjs #189`) — allows Smart Sessions to be initialized with STX Validator-compatible configuration
- **MEE version negotiation** (`abstractjs #166`) — the SDK communicates the expected MEE version to the node

### MEE Node Integration (Supporting Context)

The mee-node required corresponding updates to work with the STX Validator:

- **SuperTx/meeUserOp hash alignment** (`mee-node #138–#143`) — ensuring the node generates hashes using the same algorithm as the contracts, including typehash fixes and EIP-712 formatting
- **P256 validator modes** (`mee-node #179`) — node-side support for passkey signing flows
- **Safe SA mode** (`mee-node #156`) — node-side support for Safe smart account transactions
- **MEE v3.0.0 integration** (`mee-node #175, #176`) — new permit mode encoding and general v3 integration
- **Quote type return** (`mee-node #142`) — proper quote type detection and propagation

### Migration from K1MeeValidator

The STX Validator is a drop-in ERC-7579 replacement for K1MeeValidator. All existing MEE modes are preserved (Simple, Permit, Tx, Safe Account, No-Stx fallback). The migration path is: uninstall K1MeeValidator module, install StxValidator module with equivalent configuration. The SDK handles this transparently for new accounts.

---

## 8. Useful Links

### Key PRs — stx-contracts

| Date | Title | Link |
|------|-------|------|
| 2026-02-20 | feat: stx validator: resolve conflicts with develop | [#19](https://github.com/bcnmy/stx-contracts/pull/19) |
| 2026-02-02 | Feat/stx validator | [#18](https://github.com/bcnmy/stx-contracts/pull/18) |
| 2025-12-01 | Fix [L3] restore pointers | [#16](https://github.com/bcnmy/stx-contracts/pull/16) |
| 2025-12-01 | fix [l1] [l2] events errors | [#15](https://github.com/bcnmy/stx-contracts/pull/15) |
| 2025-12-01 | Fix [h-1]: incorrect hashing | [#14](https://github.com/bcnmy/stx-contracts/pull/14) |
| 2025-11-25 | feat: Add new fusion mode: safe smart account mode | [#13](https://github.com/bcnmy/stx-contracts/pull/13) |
| 2025-11-20 | release: v2.2.1 | [#11](https://github.com/bcnmy/stx-contracts/pull/11) |
| 2025-11-17 | fix: make simple mode 712 sigs multichain | [#9](https://github.com/bcnmy/stx-contracts/pull/9) |

### Key PRs — abstractjs (SDK)

| Date | Title | Link |
|------|-------|------|
| 2026-02-26 | feat: add support of stx validator initdata format to SS init | [#189](https://github.com/bcnmy/abstractjs/pull/189) |
| 2026-02-24 | chore: fix tests to use proper quote type | [#188](https://github.com/bcnmy/abstractjs/pull/188) |
| 2026-02-23 | chore: refactor signPermitQuote not to use _account | [#187](https://github.com/bcnmy/abstractjs/pull/187) |
| 2026-02-10 | feat: mee 3.0.0, stx validator support | [#184](https://github.com/bcnmy/abstractjs/pull/184) |
| 2025-12-16 | feat: safe sa fusion mode | [#174](https://github.com/bcnmy/abstractjs/pull/174) |
| 2025-11-06 | feat: provide mee version to node | [#166](https://github.com/bcnmy/abstractjs/pull/166) |
| 2025-11-03 | chore: update mee 2.2.0 sc addresses | [#161](https://github.com/bcnmy/abstractjs/pull/161) |

### Key PRs — mee-node (Supporting Context)

| Date | Title | Link |
|------|-------|------|
| 2026-02-16 | feat: p256 validator modes support | [#179](https://github.com/bcnmy/mee-node/pull/179) |
| 2026-02-12 | chore: fix for stx validator | [#177](https://github.com/bcnmy/mee-node/pull/177) |
| 2026-02-11 | chore: some fixes for v3.0.0 integration | [#176](https://github.com/bcnmy/mee-node/pull/176) |
| 2026-02-09 | Add MEE 3.0.0 + different permit mode encoding | [#175](https://github.com/bcnmy/mee-node/pull/175) |
| 2025-12-23 | feat: Safe SA mode | [#156](https://github.com/bcnmy/mee-node/pull/156) |
| 2025-12-08 | fix: mee userop hash generation | [#141](https://github.com/bcnmy/mee-node/pull/141) |
| 2025-12-05 | fix: make stx hash to be generated with the same algo as meeUserOp hashes | [#139](https://github.com/bcnmy/mee-node/pull/139) |
| 2025-12-04 | fix: SuperTx and meeUserOp typehashes | [#138](https://github.com/bcnmy/mee-node/pull/138) |

### Related Chapters

- **Chapter 3 — MEE K1 Validator:** The predecessor that STX Validator replaces
- **Chapter 1 — Nexus Smart Account:** The ERC-7579 account that STX Validator is installed on
- **Chapter 12 — AbstractJS SDK: MEE 3.0.0 / STX Validator / Fusion Modes:** SDK-side integration

### Standards

- [ERC-7579: Minimal Modular Smart Accounts](https://eips.ethereum.org/EIPS/eip-7579)
- [ERC-7780: Stateless Signature Validators](https://github.com/ethereum/ERCs/pull/579)
- [ERC-7739: Nested Typed Data Sign](https://github.com/ethereum/ERCs/pull/483)
- [RIP-7212: Precompile for secp256r1 Curve Support](https://github.com/ethereum/RIPs/blob/master/RIPS/rip-7212.md)
- [MEE Specification (Fusion)](https://ethresear.ch/t/fusion-module-7702-alternative-with-no-protocol-changes/20949)
