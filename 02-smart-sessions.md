# Chapter 2 — Smart Sessions Module

**Repo:** `erc7579/smartsessions`  
**Authors:** Filipp Makarov (Biconomy) · zeroknots.eth (Rhinestone)  
**Related chapters:** Chapter 1 (Nexus — host account), Chapter 10 (AbstractJS SDK — Smart Sessions Support), Chapter 8 (ERC-7739 — nested signing in 1271 flows)

---

## 1. Purpose

Smart Sessions is a validator module for ERC-7579 modular smart accounts that enables **session key management** — the ability to delegate granular, scoped, time-bound permissions to third-party signers (session keys) without exposing the account owner's primary key.

The core use case: a user wants an application (a trading bot, an AI agent, a DeFi automation protocol) to execute transactions on their behalf, but only under tightly defined conditions:

- Only to a specific contract
- Only calling a specific function
- Only with specific parameter values (e.g. amount ≤ 1 ETH)
- Only for a limited number of executions
- Only until a specific deadline
- Only after a specific timestamp

All of these constraints are enforced **on-chain** by smart contracts called **policies**. The session key never gets unrestricted access — even if it is compromised, the damage is bounded by the policy configuration.

### Who needs this

- **DeFi protocols** — let users pre-approve automated rebalancing or liquidation protection
- **Gaming/GameFi** — let game clients transact on behalf of a player within a game session without wallet pop-ups
- **AI agents** — delegate precise spending limits to an agent wallet
- **Subscriptions** — allow recurring charges within defined parameters
- **Cross-chain orchestration** — MEE SuperTransactions are authorized and validated via Smart Sessions

---

## 2. Architecture & Structure

### Contract hierarchy

```
SmartSession (main entry point)
├── SmartSessionBase (core logic: session storage, validation routing)
└── SmartSessionERC7739 (mixin: ERC-1271 + nested EIP-712 via ERC-7739)
```

**ISmartSession** defines the public interface (enable/disable sessions, validate userOps, validate ERC-1271 signatures).

### Module type

SmartSession installs on a Nexus (or any ERC-7579 account) as a **Validator** (type 1). When the account owner designates SmartSession as the validator for a specific UserOp nonce prefix, all UserOps signed with a session key route through SmartSession's `validateUserOp`.

### Session data model

A **Session** is the fundamental unit:

```
Session {
    ISessionValidator sessionValidator       // who can sign for this session
    bytes             sessionValidatorInitData
    bytes32           salt                   // for unique permissionId derivation
    PolicyData[]      userOpPolicies          // policies applied to the whole UserOp
    ERC7739Data       erc7739Policies         // 1271 allowed contents
    ActionData[]      actions                 // per-action policies
}
```

Each Session computes to a `PermissionId` — a `bytes32` hash that uniquely identifies the session for a given account. Permissions are stored in a mapping keyed by `(smartAccount, permissionId)`.

### Session validators

The **SessionValidator** is a sub-module responsible for verifying the cryptographic signature on a UserOp. Built-in options:

- **K1 (ECDSA) session validator** — standard EOA key
- **MultiKey session validator** — multiple keys required (moved to main repo in PRs #48, #49)

The `ISessionValidator` interface is intentionally minimal, enabling third parties to plug in any signature scheme (passkeys, multisig, etc.).

### Policies

Policies are separate contracts implementing one of three interfaces:

| Policy type | Scope | Interface |
|---|---|---|
| **UserOp Policy** | Applied to the entire UserOp | `IUserOpPolicy` |
| **Action Policy** | Applied to a specific contract call | `IActionPolicy` |
| **ERC-1271 Policy** | Applied to off-chain signing requests | `IERC1271Policy` |

Policies are registered as **separate ERC-7579 module types** (a design choice introduced in PR #82) so they can have their own installation/uninstallation lifecycle on the account.

### Storage

Sessions and policy state are stored in **namespaced storage** under the account address, ensuring complete isolation between accounts using the same SmartSession contract singleton. The `AssociatedArray` library (PRs #66, #129) manages ordered sets of enabled sessions and installed modules.

---

## 3. Core Features

### 3.1 Use Mode — executing with a session key

When a session key wants to act on behalf of an account, it constructs a UserOp and signs it. The UserOp's `signature` field encodes:

1. `SmartSessionMode` (USE)
2. `PermissionId` identifying which session to use
3. The session validator's signature over the UserOp hash

SmartSession decodes this, checks the session is enabled for the calling account, runs all configured UserOp policies, then runs action policies for each call in the batch. If everything passes, validation succeeds.

### 3.2 Enable Mode — enabling a session inline

A session does not have to be pre-registered in a prior transaction. **Enable Mode** allows a session to be enabled within the same UserOp that uses it. The session data is embedded in the UserOp signature alongside an EIP-712 typed data signature from the account owner authorising that session.

This is the key UX feature: apps can request a new session on first use, the user signs once (off-chain, no gas), and the session is both enabled and immediately used within a single UserOp. The enable mode signature diagram was added as documentation in PR #100.

Enable mode typehash was audited and fixed for correctness in PR #273 (Nexus) and PR #80 (remove PermissionId from enable mode sig, as it's derivable on-chain).

### 3.3 Multichain sessions

A single owner signature can authorise the same session across multiple chains. The multichain enabling flow (PRs #28, #32) uses an EIP-712 envelope wrapping a multichain digest so the user signs once and the session is valid on any configured chain.

#### How the multichain session object is constructed

The challenge: the owner needs to commit to session configurations on N different chains with a single signature, but passing the full session data for all N chains in every UserOp would be prohibitively large.

The solution uses a **hash substitution trick** in `HashLib.sol` (originally `MultichainHashLib.sol`):

1. **Off-chain (SDK side):** The SDK prepares one full `Session` struct per chain. It computes the EIP-712 `hashStruct` of each session. The final signed object is a `MultichainSession` struct containing an array of `(chainId, sessionDigest)` pairs — where `sessionDigest` is the hash of the full session for that chain.

2. **The signed digest** is therefore a commitment to all sessions on all chains in compact form — just an array of 32-byte hashes and chain IDs. The owner signs this once off-chain via EIP-712.

3. **On-chain (enabling time):** When a UserOp arrives on chain X to enable a session, the enable mode payload contains:
   - The **full session data** for chain X only
   - The **pre-hashed digests** for all other chains (passed in as `bytes32` values)
   - The owner's signature over the multichain digest

4. **Verification on chain X:**
   - The contract locally computes `sessionHash[X]` from the full session data provided in the payload
   - It takes the pre-hashed values for all other chains at face value (they are commitments, not re-verified)
   - It reconstructs the `MultichainSession` struct: `(chainId[0], sessionDigest[0]), (chainId[1], sessionDigest[1]), ...` where `sessionDigest[X]` is the locally computed hash and all others are the passed-in values
   - It computes the final EIP-712 hash of this struct and verifies the owner's signature against it

5. **Security property:** An attacker cannot swap in a different session for chain X because `sessionDigest[X]` is computed locally from the provided session data — it cannot be spoofed. An attacker cannot swap sessions for other chains either because those hashes are committed to in the signature. Changing any `(chainId, sessionDigest)` pair invalidates the owner's signature.

The key insight is that each chain only needs the full session data for **itself** to verify the signature. All other chains are represented as opaque 32-byte hashes — analogous to a Merkle proof where you reveal one leaf in full and provide sibling hashes for the rest.

The SDK integration for constructing this object lives in Chapter 10 (abstractjs PR #110), where `buildMultichainSessionEnableObject` assembles the full struct and the owner signs it once.

### 3.4 Policies in detail

**Built-in policies as of the tracked period:**

| Policy | Type | Description |
|---|---|---|
| SudoPolicy | UserOp / Action | Approves everything — unrestricted. For trusted sessions. |
| UniActionPolicy | Action | Usage count limit per action |
| SpendingLimitsPolicy | Action | ERC-20 transfer value cap |
| TimeFramePolicy | UserOp | Valid-until / valid-after time bounds |
| ValueLimitPolicy | UserOp | Native ETH value cap per UserOp |
| ERC20Policy | Action | Token-specific controls (originally included approvals, later refactored) |
| ContractWhitelistPolicy | Action | Only allow calls to whitelisted contract addresses (PR #159) |

**Policy configuration minimums:** At least one UserOp policy **or** at least one action policy is required per session (PR #85). This prevents accidentally creating a fully unconstrained session with no policies.

**Action policy fallback (PR #91):** If an action has no specific policies configured but a fallback action policy exists, the fallback is applied. This simplifies configuration for accounts wanting a blanket rule for unexpected call targets.

**Minimum policies enforcement (PR #90):** The session can define a minimum set of policies that must be configured — useful for protocols wanting to guarantee certain safety constraints are always present.

### 3.5 ERC-1271 / ERC-7739 support

SmartSession validates not just UserOps but also off-chain typed data signing requests (ERC-1271 `isValidSignature`). The `SmartSessionERC7739` mixin implements the nested EIP-712 envelope approach defined by ERC-7739, preventing signature replay across different sessions or accounts.

ERC-1271 policies are a separate category — when a dApp asks the smart account to sign typed data off-chain, the configured ERC-1271 policies run to decide whether that content is permitted. The `allowedERC7739Content` field on the session controls which typed data structures are signable.

Fixes and refinements across multiple PRs: #71, #73, #77, #130, #133.

### 3.6 FLZ compression of USE mode signature (PR #143)

The USE mode signature (mode byte + permissionId + session validator sig) was previously FLZ-compressed for calldata savings. This was later **removed** (PR #143 explicitly does not compress) because the 28 bytes of calldata overhead reduction was not worth the complexity and the gas cost of decompression.

---

## 4. Implementation Details

### PermissionId derivation

`PermissionId` is derived deterministically from `(sessionValidator, sessionValidatorInitData, salt, policies)` — the full session configuration. This means:

- The same session configuration always yields the same `PermissionId`
- Adding a `salt` allows creating multiple sessions with identical policies but distinct IDs
- `PermissionId` was intentionally removed from the enable mode signature (PR #80) because the validator can recompute it on-chain from the session data

### AssociatedArray library

Ordered enumerable sets stored in ERC-7201 namespaced storage slots, keyed by `(account, PermissionId)`. Used to track enabled sessions per account, installed policies per session, etc. Storage layout for this library had a bug (PR #129 fix) where it was storing data in the wrong slots — a critical correctness fix.

### Policy initialization guard

Policies are initialized when a session is enabled. If a policy already has state from a prior session, its `initializeWithMultiplexer` must handle the existing state correctly. PR #87 fixed a bug where policies were not properly reinitializing when state already existed, and PR #117 fixed the reverse — storage not being properly cleaned before reinit.

### Double-add protection

PR #86 fixed a case where the same `ActionId` could be added twice to a session, leading to duplicate policy enforcement and corrupted state.

### Status field removal from UniActionPolicy (PR #121)

The `status` field was found to be redundant — the policy's active/inactive state is already tracked in the parent session's storage. Removing it reduced storage slots and simplified the contract.

### Signature encoding for ERC-1271 (PR #130)

A bug where the ERC-1271 flow was not enforcing minimum required policies. The fix ensures the same policy enforcement rigor applies to off-chain signing requests as to UserOps.

### Approvals in ERC20 policy

Originally the ERC20 policy supported tracking and limiting token approvals (`approve()` calls). This was split out:
- PR #138 (Stacked): Removed approval tracking from the base ERC20 policy to simplify it
- PR #146: Re-introduced approval controls but scoped per spender — a cleaner model where limits are tracked at the `(token, spender)` level rather than globally

---

## 5. Design Choices & Rationale

### Why separate policy contracts rather than inline policy logic

Early session key systems (including Biconomy's SCW v2 `SessionValidationModule`) embedded validation logic for each use case directly into a monolithic module. The Smart Sessions design externalises this into pluggable policy contracts. This means:

- Policy logic can be audited independently
- New use cases add new policy contracts without touching the core
- Multiple policies can be composed per session
- Policies can be reused across different sessions and accounts

The tradeoff is more contract calls during validation, but this is acceptable given that session key validation is a relatively infrequent operation compared to reading policies.

### Policies as separate ERC-7579 module types (PR #82)

By giving policies their own module type designation, they participate in the standard install/uninstall lifecycle of the ERC-7579 account. This means:
- Registry approval checks can apply to policy contracts
- The account's hook system can intercept policy installation
- Policies can be enumerated and managed via standard interfaces

### SessionValidator as a sub-module (not embedded)

The cryptographic verification of session signatures is delegated to `ISessionValidator` implementations rather than being hardcoded. This makes SmartSession signature-scheme-agnostic. A Passkey-based session needs only a passkey SessionValidator — the policy evaluation layer is identical.

### PermissionId removed from enable mode sig (PR #80)

Initially the enable mode signature included the PermissionId explicitly. This was redundant since PermissionId is fully deterministic from the session data that is also in the signature. Removing it saves bytes and removes a potential mismatch attack vector where a malicious bundler could supply a mismatched PermissionId.

### FLZ compression removed from USE mode (PR #143)

The USE mode signature is calldata in the UserOp. FLZ compression was explored to reduce calldata cost on L2s. It was ultimately removed because: the decompression gas cost partially offset savings, it added implementation complexity, and the 28-byte saving was modest relative to the overall UserOp size.

### AssociatedArray over a bare EnumerableSet

Standard OpenZeppelin `EnumerableSet` doesn't support keying by an external `account` address — it stores a single global set. The `AssociatedArray` pattern allows `set[account]` semantics while remaining gas-efficient. This library lives in `erc7579/enumerablemap` (Chapter 18).

---

## 6. Usage Details

### Installing the module

```solidity
// Install SmartSession as a validator on a Nexus account
nexus.installModule(
    MODULE_TYPE_VALIDATOR,
    SMART_SESSION_ADDRESS,
    "" // no initdata required; sessions are added separately
);
```

It is valid to install SmartSession with empty initdata. Sessions are configured after installation.

### Creating a session (pre-enable)

```typescript
// Using AbstractJS SDK (Chapter 10)
const response = await usersNexusClient.grantPermission({
    sessionRequestedInfo: [{
        sessionPublicKey: sessionOwner.address,
        actionPoliciesInfo: [{
            contractAddress: "0xTargetContract",
            functionSelector: "0xdeadbeef",
            validUntil: BigInt(Math.floor(Date.now() / 1000) + 7 * 24 * 3600),
            usageLimit: BigInt(100)
        }]
    }]
});
// Returns permissionId, to be stored and used when calling usePermission
```

### Using a session key

```typescript
const usePermissionsModule = toSmartSessionsValidator({
    account: nexusAccount,
    signer: sessionOwner,    // the session key
    moduleData: { permissionId }
});
const sessionClient = nexusClient.extend(smartSessionUseActions(usePermissionsModule));
await sessionClient.usePermission({
    calls: [{ to: targetContract, data: calldata }]
});
```

### Enable Mode flow (inline session creation)

The SDK's `getEnableSessionDetails` (from `@rhinestone/module-sdk`) prepares the enable mode data. The user signs the session struct off-chain. The bundler submits a UserOp that simultaneously enables and uses the session. No prior on-chain transaction needed.

### Multichain session enable

```typescript
// SDK: Chapter 10, abstractjs PR #110
const multichainsessionEnableObject = await buildMultichainSessionEnableObject({
    chains: [base, arbitrum, optimism],
    session: sessionConfig
});
// Owner signs once; the enable object is valid on all specified chains
```

### Checking if a permission is enabled

```typescript
// abstractjs PR #151
const isEnabled = await nexusClient.isPermissionEnabled({ permissionId });
```

---

## 7. Caveats & Known Limitations

### Policy re-initialization edge cases

When a policy already has state from a previous session (same policy contract, same account), re-enabling it requires careful state management. The `initializeWithMultiplexer` function must handle existing state gracefully. Early versions had bugs here (PR #87 and #117) — if extending or modifying policies, test re-installation carefully.

### Module already installed guard

If SmartSession has enabled sessions when `uninstallModule` is called, it reverts (`SmartSessionModuleAlreadyInstalled` — confusingly named, should read as "cannot uninstall with active sessions"). Sessions must be individually disabled before the module can be uninstalled. This is intentional to prevent accidental loss of active session state, but it means cleanup must be explicit.

### AssociatedArray storage layout bug (PR #129)

There was a non-trivial storage layout bug in the AssociatedArray library affecting production versions deployed before that fix. If working with a deployment predating PR #129, storage enumeration could return incorrect results. Check deployed version against this PR.

### Minimum policy requirement

A session with zero UserOp policies and zero action policies cannot be created (PR #85 enforcement). This is a safety guard but it means you cannot create a "no constraints" session without explicitly choosing SudoPolicy.

### ERC-1271 policy minimum enforcement (pre-PR #130)

Early versions did not enforce minimum policies on the ERC-1271 flow — a session could sign arbitrary typed data even if the session configuration implied it should not. Fixed in PR #130. Versions before that fix may be vulnerable to over-permissive off-chain signing.

### No cross-session privilege escalation (by default)

Sessions cannot grant new sessions by default. There is a prototype feature (PR #150 in `smartsessions`) for allowing sessions to add other sessions via a fallback action routine, but it is explicitly opt-in and marked as dangerous due to privilege escalation potential. Do not enable this for production sessions.

### FLZ decompression gas overhead

The decompression step (if anyone were to re-introduce it) runs in the validation phase, which is gas-metered. Excessive decompression cost could cause validation to fail or exceed the verificationGasLimit.

### Multichain session replay considerations

Multichain sessions sign a digest that is valid on all specified chains. If an attacker gains access to a session key, they can use it on all chains where the session is enabled. When designing session configurations, consider whether truly multichain access is necessary or whether per-chain sessions are safer.

---

## 8. Key Audit Findings & Their Fixes

| Finding | PR | Summary |
|---|---|---|
| Policy init when state exists | #87 | Policies not re-initialized if prior state existed |
| Storage not cleaned before reinit | #117 | Stale policy state after session removal |
| AssociatedArray storage layout | #129 | Incorrect slot computation in library |
| ERC-1271 min policies not enforced | #130 | Off-chain signing bypassed policy checks |
| ERC-7739 verifyingAddress | #73 | Wrong address used in 7739 envelope |
| Fix orders in Typehashes | #72 | Typehash field order mismatch vs encoding |
| Enable mode typehash | (also Nexus #273) | Typehash did not match struct layout |
| Issue 15 (Spearbit) | #116 | See Cantina report |
| Issues 19, 20-21, 23 (Spearbit) | #118, #119, #120 | See Cantina report |
| Aggregated Cantina/Spearbit | #83 | Batch remediation of initial audit |

---

## 9. Scope for Improvement

### Policy composability UI

Currently composing multiple policies requires knowing their contract addresses and encoding initdata by hand. A registry-backed policy marketplace with SDK abstractions would significantly lower the developer experience bar.

### Policy upgradability

Policies are immutable contracts. If a policy has a bug, the only recourse is to disable the session and re-enable with a different policy address. A proxy pattern for policies could allow patching, but introduces centralisation risk.

### Gas optimisation in validation

Policy validation is sequential — each policy is called individually. Batching policy checks into a single call or using multicall patterns could reduce validation gas, which matters for chains with high gas sensitivity.

### Universal policy (in progress)

A more expressive "universal" policy combining time bounds, value limits, usage counts, and call whitelisting into a single contract would reduce the number of installed policies per session, saving both storage and validation gas. There is ongoing work in the abstractjs SDK (PR #149 — "ss with universal policy" test).

### Session lifecycle events

Events for session enable and disable exist, but a more comprehensive indexing story (dedicated subgraph or event schema) would help dApps track session state across chains.

### Delegation chaining

The current architecture does not support a session key delegating to sub-keys. This is an intentional simplification but is increasingly relevant for AI agent architectures where an agent needs to spawn sub-agents with further-restricted permissions.

---

## 10. Version & Deployment Notes

Smart Sessions is versioned independently from Nexus. Deployment addresses are maintained in `bcnmy/abstract-docs` and `bcnmy/documentation`. The module is deployed at deterministic addresses via CREATE2.

The module's deployment scripts are covered in Chapter 14 (Deployment Scripts), specifically `stx-contracts #17` which added dedicated Smart Sessions deployment scripts separate from the main STX suite.

---

## 11. Useful Links

| Resource | URL |
|---|---|
| Smart Sessions repo | https://github.com/erc7579/smartsessions |
| Biconomy Smart Sessions docs | https://docs.biconomy.io/modules/validators/smartSessions |
| Biconomy Smart Sessions tutorial | https://docs.biconomy.io/tutorials/smart-sessions |
| Smart Sessions policies reference | https://docs.biconomy.io/modules/validators/smartSessions/policies/ |
| Rhinestone module-sdk (usage) | https://docs.rhinestone.wtf/module-sdk/using-modules/smart-sessions |
| Enable mode diagram | https://github.com/erc7579/smartsessions/pull/100 |
| Cantina/Spearbit audit remediation | https://github.com/erc7579/smartsessions/pull/83 |
| AssociatedArray library | https://github.com/erc7579/enumerablemap |
| AbstractJS SDK – Sessions (Ch. 10) | See Chapter 10 |
