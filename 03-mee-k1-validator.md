# Chapter 3 — MEE K1 Validator

**Repo:** `bcnmy/mee-contracts`  
**Primary Author:** Fil Makarov (@filmakarov)  
**Status:** Superseded by STX Validator (see Chapter 4)  
**PRs:** ~17 in `bcnmy/mee-contracts`

---

## 1. Purpose

The MEE K1 Validator is the core signature validation contract for the Modular Execution Environment (MEE). It verifies that a SuperTransaction — a meta-transaction structure that bundles multiple cross-chain UserOperations into a single user-signed object — was authorized by the account owner.

The name "K1" refers to secp256k1, the elliptic curve used by Ethereum EOAs for ECDSA signatures. The validator's job is to recover the signer from a provided signature, compare it to the registered account owner, and return a validation result that the EntryPoint can consume.

The K1 MEE Validator supports four distinct **signature modes**, each designed for a different user-facing signing experience. This multi-mode approach is the contract's defining design characteristic: a single validator handles fundamentally different authorization flows (direct EIP-712 signing, on-chain Ethereum transaction signing, ERC-2612 permit-based signing, and standard ERC-4337 fallback) by inspecting a mode byte in the signature payload and dispatching to the appropriate validation library.

### Why it exists

In a chain-abstracted environment, the user signs a single SuperTransaction that authorizes operations across multiple chains. The MEE Node then decomposes this into per-chain UserOperations, each of which must be individually validated on-chain. The K1 Validator bridges this gap: it can verify that a per-chain UserOperation was authorized as part of a broader SuperTransaction the user signed, using Merkle proofs to link individual operations back to the original signed root.

---

## 2. Architecture & Structure

### Contract Layout

```
src/
├── K1MeeValidator.sol          // Main validator contract
├── lib/
│   ├── HashLib.sol             // EIP-712 typehashes, domain separator, SuperTx hashing
│   ├── SimpleValidatorLib.sol  // Simple Mode validation logic
│   ├── TxValidatorLib.sol      // On-Chain Transaction Mode validation logic
│   ├── PermitValidatorLib.sol  // ERC-2612 Permit Mode validation logic
│   ├── NoMeeFlowLib.sol        // Non-MEE Fallback Mode validation logic
│   └── BytesLib.sol            // Byte manipulation utilities
```

### Core Dependencies

The K1MeeValidator inherits from or integrates with:

- **ERC-7579 IValidator interface** — implements `validateUserOp(PackedUserOperation, bytes32)` and `isValidSignatureWithSender(address, bytes32, bytes)` as required by the modular smart account standard.
- **ERC-7739 Validator Base** — for nested typed data signing support in ERC-1271 flows. The non-MEE fallback mode uses full ERC-7739 wrapped signature verification; MEE modes (permit, on-chain tx) use a simplified non-7739 ERC-1271 flow because the verifying context is different.
- **Solady ECDSA** — used instead of OpenZeppelin for gas-optimized `ecrecover` wrapper (the OZ-to-Solady migration happened during audit prep, PR #8).
- **Merkle proof verification** — each UserOperation in a SuperTransaction carries a Merkle proof demonstrating its inclusion in the SuperTransaction's Merkle root. The validator verifies this proof to confirm the individual operation was part of what the user actually signed.

### Owner Registry

The contract maintains a mapping of smart account addresses to their EOA owners:

```
mapping(address smartAccount => address owner) internal owners;
```

This is set during module installation (`onInstall`) and cleared during uninstallation (`onUninstall`). The owner is the EOA whose secp256k1 signature the validator expects to recover.

---

## 3. Core Features — The Four Modes

The K1 MEE Validator uses a **mode byte** at the start of the signature payload to determine which validation path to execute. Each mode has its own packed signature data layout with a static head part and mode-specific tail data (including timestamps, Merkle proofs, and mode-specific parameters). The validator reads the mode byte and dispatches to the appropriate library.

### 3.1 Simple Mode

**Purpose:** The cleanest MEE flow — the user signs an EIP-712 typed data hash of the entire SuperTransaction directly.

**When to use:** When the frontend/SDK has full control of the signing experience and can present EIP-712 typed data to the user's wallet.

**Validation flow:**

1. Extract the mode byte; dispatch to `SimpleValidatorLib`.
2. Extract the 65-byte ECDSA signature and the remaining Merkle proof bytes.
3. Compute the `meeUserOpHash` — an EIP-712-compliant hash of the individual MEE UserOperation. This is the leaf in the Merkle tree.
4. Verify the Merkle proof: the `meeUserOpHash` (leaf) must produce the SuperTransaction's Merkle root when combined with the proof.
5. Wrap the Merkle root into the full SuperTransaction EIP-712 typed data hash using `HashLib`, incorporating the EIP-712 domain separator.
6. Recover the signer from the ECDSA signature over this typed data hash using `ecrecover`.
7. Compare the recovered signer to `owners[userOp.sender]`.

**EIP-712 Typed Data Structure:**

The SuperTransaction is signed as an EIP-712 struct. The `HashLib` defines the typehash and domain separator. Originally Simple Mode signed a raw blind hash of the SuperTx; PR #45 introduced proper EIP-712 typed data signing, which means wallets can display human-readable signing prompts to users.

Key detail from PR #45: the implementation supports **arbitrary data structs as STX entries**. STX items are not limited to a fixed struct layout — they can contain varied data, and the hashing algorithm handles this by hashing each item individually and then composing them into the SuperTx hash.

**Domain separator (post audit fix L-03):**

Per PR #47, the EIP-712 domain separator was simplified to enable cross-chain SuperTransaction signatures. Because a single SuperTransaction spans multiple chains, the domain separator cannot include `chainId` or `verifyingContract` — these would bind the signature to a single chain. The `version` field was also removed since contract versions may differ across chains. Per EIP-712, unused domain fields can be omitted entirely from the struct type rather than zeroed out:

```solidity
// Domain typehash includes only `name`
_DOMAIN_TYPEHASH = keccak256("EIP712Domain(string name)");
```

This is a legitimate EIP-712 approach — the standard explicitly allows protocol designers to include only the fields that make sense for their signing domain.

### 3.2 On-Chain Transaction Mode (Fusion)

**Purpose:** Enables MEE authorization by having the user sign a standard Ethereum transaction that embeds the SuperTransaction hash in its calldata. This is the "Fusion" mode — fusing a regular ETH transaction with MEE authorization.

**When to use:** When the user needs to perform an on-chain transaction anyway (e.g., a token transfer or approval) and wants to simultaneously authorize a SuperTransaction without a separate signing step. Also useful for wallets that don't support EIP-712 typed data signing.

**Validation flow:**

1. Extract the mode byte; dispatch to `TxValidatorLib`.
2. Parse the RLP-encoded signed Ethereum transaction from the signature payload.
3. Extract `r`, `s`, `v` from the transaction's signature fields.
4. Reconstruct the unsigned transaction hash from the RLP-encoded data (stripping the signature fields).
5. Recover the signer via `ecrecover(unsignedTxHash, v, r, s)`.
6. Extract the SuperTransaction hash from the transaction's calldata — the STX hash is appended at the end of the `data` field. The validator reads the last 32 bytes of the transaction's calldata to obtain the STX root.
7. Compute the `meeUserOpHash` leaf and verify the Merkle proof against the extracted STX hash.
8. Compare the recovered signer to `owners[userOp.sender]`.

**RLP parsing details:**

The `TxValidatorLib` includes RLP decoding logic to extract fields from the signed transaction. A notable audit finding (PR #15) identified that `RLP_ENCODED_R_S_BYTE_SIZE` could be incorrect in edge cases where `r` or `s` values have leading zeros, changing their RLP-encoded length. The fix added proper handling for variable-length RLP encoding of these signature components.

**EIP-155 support:**

The transaction validation supports EIP-155 replay protection. The `v` value is adjusted according to the EIP-155 formula (`v = chainId * 2 + 35 + recoveryBit`) to derive the correct recovery parameter for `ecrecover`. A constant `EIP_155_MIN_V_VALUE` is used to detect and handle EIP-155 vs legacy transaction formats.

### 3.3 ERC-2612 Permit Mode

**Purpose:** Combines an ERC-20 token approval (via ERC-2612 `permit()`) with SuperTransaction authorization. The user signs a single message that both authorizes the Nexus account (orchestrator) to spend tokens (for fees) and authorizes the SuperTransaction execution.

**When to use:** When the user needs to grant token spending approval as part of the MEE flow — typically for paying gas in ERC-20 tokens. Instead of requiring a separate `approve` transaction, the permit signature is combined with the MEE authorization.

**Validation flow:**

1. Extract the mode byte; dispatch to `PermitValidatorLib`.
2. Decode the ERC-2612 permit parameters: `token`, `spender`, `value`, `deadline`, `nonce`.
3. Reconstruct the EIP-712 permit typed data hash according to the ERC-2612 standard (using the token contract's domain separator).
4. Recover the signer from the permit signature.
5. Extract the SuperTransaction hash from the `deadline` field. The `deadline` is a `uint256`, which is the same size as a `bytes32` — the STX hash is stored directly in this field. This dual-purpose encoding is the key trick: the permit signature simultaneously authorizes a token spend and commits to a specific SuperTransaction.
6. Verify the Merkle proof linking the `meeUserOpHash` to the STX root extracted from the deadline.
7. Compare the recovered signer to `owners[userOp.sender]`.

**Permit frontrunning (PR #16):**

An audit finding identified that permits can be frontrun — an attacker could observe the permit in the mempool and execute it before the intended transaction. The fix ensures that permit execution failure (due to frontrunning) doesn't cause the entire MEE flow to revert. Instead, if the permit has already been executed (nonce consumed), the validation still succeeds as long as the allowance is sufficient.

### 3.4 Non-MEE Fallback Mode

**Purpose:** Standard ERC-4337 UserOperation validation with no MEE-specific logic. Enables the smart account to be used as a regular smart account when not participating in a SuperTransaction.

**When to use:** When the user sends a standard single-chain UserOperation that doesn't involve the MEE Node or SuperTransaction architecture.

**Validation flow:**

1. Extract the mode byte; dispatch to `NoMeeFlowLib`.
2. The `userOpHash` provided by the EntryPoint is used directly (no Merkle proof needed — this is a standalone operation).
3. Recover the signer via standard ECDSA verification.
4. Compare to `owners[userOp.sender]`.

**ERC-1271 / ERC-7739 handling:**

In non-MEE fallback mode, the full ERC-7739 nested typed data signing flow is used for `isValidSignatureWithSender`. This is the same pattern as the standard Nexus K1 Validator. For MEE modes (permit, on-chain tx), a simplified non-7739 ERC-1271 flow is used instead, because the signing context differs from what ERC-7739 expects.

---

## 4. Implementation Details

### Merkle Proof Architecture

Every MEE mode (except non-MEE fallback) uses Merkle proofs to link an individual UserOperation back to the SuperTransaction the user signed. The structure is:

```
SuperTransaction
├── meeUserOpHash[0]  (chain A operation)
├── meeUserOpHash[1]  (chain B operation)
├── meeUserOpHash[2]  (chain C operation)
└── ...
```

The user signs the **Merkle root** (or a hash derived from it). Each per-chain UserOperation carries a Merkle proof that, when applied to its own `meeUserOpHash`, reproduces this root. The on-chain validator:

1. Computes the `meeUserOpHash` for the UserOperation being validated.
2. Iteratively hashes the leaf with each 32-byte proof element.
3. Compares the resulting root to the one extracted from the user's signature.

This is a standard binary Merkle tree — no sorted pairs or custom variants. The Murky library was initially used for testing but was later removed (PR #8, commit "chore: remove murky") in favor of direct proof construction in tests.

### HashLib — EIP-712 Hashing

`HashLib.sol` is the central hashing library. Key responsibilities:

- **Domain separator construction:** Uses only the `name` field (post L-03 fix) to enable cross-chain signatures. No `chainId`, `verifyingContract`, or `version`.
- **SuperTransaction typehash:** Defines the EIP-712 struct type for the SuperTransaction.
- **Item hashing:** Each STX item (MEE UserOperation) is individually hashed, then the array of hashes is packed and keccak'd to form the items component of the SuperTx struct hash.
- **`eip712Domain()` view function:** Returns the domain fields for off-chain tooling (e.g., ethers.js, viem) to construct matching signatures.

---

## 5. Design Choices

### Why Four Modes?

Each mode addresses a distinct UX/integration scenario:

- **Simple Mode** is the ideal case — clean EIP-712 signing — but requires wallet support.
- **On-Chain Transaction Mode** captures value from transactions the user is already making, and works with wallets that can only sign raw transactions.
- **Permit Mode** eliminates the need for a separate approval transaction, reducing friction in token-payment flows.
- **Non-MEE Fallback** ensures the validator doesn't lock users into MEE-only usage — the same smart account can operate normally.

The mode byte approach (single byte prefix) was chosen for minimal gas overhead while maintaining clear separation between fundamentally different validation paths.

### Why K1 (secp256k1) Only?

The K1 Validator is limited to secp256k1 because that's what all Ethereum EOAs use. This was a deliberate simplification for the first version. The limitation became a driver for the STX Validator (Chapter 4), which adds P256 (secp256r1) support for passkey/WebAuthn flows.

### Why EIP-712 Was Added to Simple Mode Later

Initially, Simple Mode signed a raw (blind) hash of the SuperTransaction. PR #45 introduced proper EIP-712 typed data signing. The motivation: wallets can display EIP-712 typed data to users in a human-readable format, whereas blind hash signing just shows an opaque hex string. This is both a UX improvement (users can see what they're signing) and a security improvement (reduces blind-signing risks).

### Library Architecture Over Inheritance

The per-mode library pattern was chosen over inheritance (e.g., separate validator contracts per mode) to keep everything in a single deployed contract. This simplifies deployment, module installation, and avoids the smart account needing to install/manage multiple validators. The tradeoff is a larger contract size, which was later optimized.

---

## 6. Usage Details

### Module Installation

The K1 MEE Validator is installed as an ERC-7579 validator module on a Nexus smart account. During `onInstall`, the caller provides the EOA owner address:

```solidity
function onInstall(bytes calldata data) external {
    address owner = address(bytes20(data));
    owners[msg.sender] = owner;
}
```

The `msg.sender` here is the smart account itself (since module installation is called via the account's `installModule` function).

### SuperTransaction Flow (Simple Mode Example)

1. User constructs a SuperTransaction containing multiple cross-chain operations via the SDK (abstractjs).
2. SDK builds the Merkle tree of `meeUserOpHash` leaves and computes the root.
3. SDK constructs the EIP-712 typed data hash of the SuperTransaction (using `HashLib`'s typehash and domain separator).
4. User signs the typed data hash via their wallet.
5. SDK sends the SuperTransaction to the MEE Node.
6. MEE Node decomposes the SuperTransaction into per-chain UserOperations.
7. For each UserOperation, the MEE Node constructs the packed signature data containing the mode byte (simple mode), the user's ECDSA signature, timestamps, and the Merkle proof for this operation.
8. The MEE Node (or a bundler) submits each UserOperation to its respective chain's EntryPoint.
9. EntryPoint calls `validateUserOp` on the smart account, which delegates to the K1 MEE Validator.
10. The validator verifies the Merkle proof and recovers the signer, returning validation success/failure.

### ERC-1271 Signature Verification

The validator also implements `isValidSignatureWithSender` for on-chain signature verification (e.g., verifying signed messages for dApp interactions). The behavior depends on whether the signature carries an MEE mode byte or is a standard ERC-1271 request:

- **Non-MEE (fallback):** Full ERC-7739 nested typed data verification.
- **MEE modes (permit, on-chain tx):** Simplified ERC-1271 verification without ERC-7739 wrapping, since the signing context is MEE-specific.

---

## 7. Caveats & Known Limitations

### Cross-Chain Domain Separator Trade-off

To enable cross-chain SuperTransaction signatures, the EIP-712 domain separator omits `chainId` and `verifyingContract` (PR #47). This is valid per EIP-712, but it means the domain separator provides weaker replay protection compared to a standard single-chain EIP-712 domain. The SuperTransaction itself includes chain-specific data in its items, so replay across different SuperTransactions is still prevented — but the domain separator alone does not bind to a specific chain or contract.

### Permit Frontrunning

While the permit frontrunning issue (PR #16) was mitigated, the fundamental MEV risk of permits being extracted from the mempool remains a general Ethereum concern, not specific to this contract. The mitigation ensures the MEE flow doesn't break if frontrunning occurs, but doesn't prevent it.

### RLP Byte Size Edge Case

The RLP-encoded `r` and `s` values in the On-Chain Transaction Mode can have variable lengths when leading zeros are present (PR #15). While fixed, this is the kind of encoding detail that can cause issues if the SDK-side encoding doesn't match the contract's expectations. The node and SDK must produce RLP-encoded transactions consistent with what `TxValidatorLib` expects to parse.

### secp256k1 Only

No support for P256 (passkeys/WebAuthn), ed25519, or other curves. Users must sign with a standard Ethereum EOA.

---

## 8. Scope for Improvement

The K1 MEE Validator has been superseded by the STX Validator (Chapter 4), which addresses the key limitations — most notably adding P256 support and aligning with the unified `stx-contracts` architecture and MEE 3.0.0.

---

## 9. Audit History

The K1 MEE Validator went through a dedicated audit cycle coordinated by Filio. Key audit-related PRs:

| Finding | Severity | PR | Description |
|---------|----------|-----|-------------|
| Issues #3 and #9 | Mixed | [#13](https://github.com/bcnmy/mee-contracts/pull/13) | Combined fix for two findings |
| Issue #8 — Underflow in calculateRefund | Low | [#14](https://github.com/bcnmy/mee-contracts/pull/14) | Potential underflow when actual gas exceeds pre-funded amount |
| RLP_ENCODED_R_S_BYTE_SIZE incorrect | Low | [#15](https://github.com/bcnmy/mee-contracts/pull/15) | Variable-length RLP encoding of signature components |
| Permit can be frontrun | Medium | [#16](https://github.com/bcnmy/mee-contracts/pull/16) | ERC-2612 permit MEV vulnerability |
| Low-risk #7 and informationals | Low/Info | [#17](https://github.com/bcnmy/mee-contracts/pull/17) | Batch of low-risk and informational findings |
| [M-01] Struct encoding | Medium | [#46](https://github.com/bcnmy/mee-contracts/pull/46) | Non-standard EIP-712 struct encoding in HashLib |
| [L-03] Domain separator cross-chain | Low | [#47](https://github.com/bcnmy/mee-contracts/pull/47) | Domain separator prevented cross-chain SuperTx signatures |

The audit was performed by Pashov Audit Group (referenced in PR #46 and #47 linking to `PashovAuditGroup/Biconomy_October25_MERGED`).

---

## 10. Useful Links

### K1 MEE Validator PRs (chronological)

| Date | Title | Link |
|------|-------|------|
| 2024-12-13 | Minor fixes | https://github.com/bcnmy/mee-contracts/pull/2 |
| 2025-01-16 | Move some interfaces and libs to dependencies | https://github.com/bcnmy/mee-contracts/pull/4 |
| 2025-01-17 | MEE EP => prepare for audit | https://github.com/bcnmy/mee-contracts/pull/5 |
| 2025-01-31 | K1 Audit Prep | https://github.com/bcnmy/mee-contracts/pull/8 |
| 2025-02-11 | Fix issues #3 and #9 | https://github.com/bcnmy/mee-contracts/pull/13 |
| 2025-02-11 | Fix Issue #8 Potential Underflow in calculateRefund | https://github.com/bcnmy/mee-contracts/pull/14 |
| 2025-02-12 | Fix RLP_ENCODED_R_S_BYTE_SIZE can be incorrect | https://github.com/bcnmy/mee-contracts/pull/15 |
| 2025-02-12 | Fix Permit can be frontrun :: issue #10 | https://github.com/bcnmy/mee-contracts/pull/16 |
| 2025-02-12 | Fix Low-risk #7 and informationals | https://github.com/bcnmy/mee-contracts/pull/17 |
| 2025-05-14 | New Fusion Mode: MetaMask Delegation Toolkit mode (deprecated) | https://github.com/bcnmy/mee-contracts/pull/39 |
| 2025-09-25 | Minor optimizations and refactors + bump to 1.0.2 | https://github.com/bcnmy/mee-contracts/pull/42 |
| 2025-10-08 | feat: EIP-712 MEE signatures in simple mode | https://github.com/bcnmy/mee-contracts/pull/45 |
| 2025-10-21 | fix: [M-01] struct encoding | https://github.com/bcnmy/mee-contracts/pull/46 |
| 2025-11-12 | Fix [L-03] Domain separator cross-chain | https://github.com/bcnmy/mee-contracts/pull/47 |

### Related PRs in Other Repos

- **Release/v0.0.2** — https://github.com/bcnmy/mee-contracts/pull/30
- **Release/v0.0.4** — https://github.com/bcnmy/mee-contracts/pull/38
- **MetaMask Delegation Toolkit fusion mode (SDK, deprecated)** — https://github.com/bcnmy/abstractjs/pull/101
- **Refactor fusion modes (SDK)** — https://github.com/bcnmy/abstractjs/pull/111

### Audit Reports

- Pashov Audit Group — `PashovAuditGroup/Biconomy_October25_MERGED` (referenced from PRs #46, #47)
