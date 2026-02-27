# Chapter 1 — Nexus Smart Account

**Knowledge Transfer · Filio Makarov · February 2026**  
**Repos:** `bcnmy/nexus` (archived) · `bcnmy/stx-contracts` (current unified repo)  
**Companion chapters:** Ch. 3 & 4 (Validators) · Ch. 5 (Composability) · Ch. 8 (ERC-7739) · Ch. 15 (PREP Mode)

---

## 1. Purpose

Nexus is Biconomy's flagship ERC-7579 modular smart account. It replaced the legacy SCW v2 (`bcnmy/scw-contracts`) as Biconomy's production smart account and serves as the foundation for all higher-order products: Smart Sessions, MEE/SuperTransactions, composable cross-chain executions, and EIP-7702 onboarding.

The core value proposition over SCW v2 is **standardized modularity**. Rather than Biconomy-specific module interfaces, Nexus is built on the ERC-7579 standard (co-authored by Filio), meaning any third-party module ecosystem can plug in without modifications. This positions Biconomy accounts as interoperable within the broader ERC-7579 module ecosystem (Rhinestone, etc.).

Nexus targets ERC-4337 EntryPoint v0.7, with v2.0.0 work begun for EP v0.8.

---

## 2. Architecture & Structure

### Account model

Nexus uses a **singleton implementation + per-user proxy** pattern inherited from Gnosis Safe / ERC-1967:

- `Nexus.sol` — the singleton implementation contract. Deployed once per chain.
- `NexusProxy.sol` — a minimal ERC-1967 proxy deployed per user. All user storage lives here. Delegate-calls into the singleton.
- `NexusBootstrap.sol` / `NexusBootstrapLib.sol` — initialization helper contracts. Handle first-time module installation during account creation.

Users never interact with `Nexus.sol` directly. All calls go through the proxy.

### Core contracts

| Contract | Role |
|---|---|
| `Nexus.sol` | Main implementation: validation, execution, module management |
| `NexusProxy.sol` | Per-user ERC-1967 proxy. Receives `receive()` for native ETH. |
| `NexusAccountFactory.sol` | Deploys `NexusProxy` instances via CREATE2 for deterministic addresses |
| `K1ValidatorFactory.sol` | Convenience factory — deploys account + installs K1 validator atomically |
| `NexusBootstrap.sol` | Stateless helper for complex initialization (multiple modules at once) |
| `NexusBootstrapLib.sol` | Library with bootstrap data encoding helpers |
| `RegistryAdapter.sol` | Optional ERC-7484 module registry integration (namespaced storage, ERC-7201) |
| `BiconomyMetaFactory.sol` | Meta-factory routing deployments to different factory implementations |

### Module system (ERC-7579)

Nexus supports four module types defined by ERC-7579:

| Type | Type ID | Role |
|---|---|---|
| Validator | `1` | Validates UserOp signatures. Implements `validateUserOp` / `isValidSignatureWithSender`. |
| Executor | `2` | Can execute calls on behalf of the account (trusted). |
| Fallback Handler | `3` | Handles calls to unknown function selectors on the account. |
| Hook | `4` | Pre/post execution hooks. Called before and after every UserOp execution. |

A fifth internal concept — **Multi-type module** — lets a single contract register as multiple module types simultaneously (e.g., a contract that is both a Validator and an Executor).

### Storage layout (ERC-7201)

Nexus uses **namespaced storage** (ERC-7201) throughout to prevent storage collisions between the singleton and proxies, and between different modules. Each component declares its own storage struct at a deterministic slot derived from a namespace string. The `RegistryAdapter` was specifically fixed in v1.2.0 to use namespaced storage after an audit finding (PR #264).

### Hooking system

Every UserOp execution passes through the hook chain:

1. Pre-validation hooks — called before signature validation. Introduced in v1.2.0 specifically to support **resource locking** in chain-abstracted environments (MEE). A pre-validation hook can reject a UserOp before validation even runs.
2. Post-execution hooks — called after the execution phase completes.

Hooks can be installed per-module (module-specific hooks) or globally. The hooking system had multiple audit-identified edge cases in v1.2.0 — hook bypass via bootstrap, double-hooking, and hook bypass via Module Enable Mode — all remediated in the Pashov/Zenith audit cycle (PRs #267–#275).

### EntryPoint relationship

Nexus implements `IAccount` from ERC-4337. The EntryPoint calls `validateUserOp` on the account, which delegates to the installed validator. Execution is handled by `execute` / `executeBatch` / `executeFromExecutor`. Nexus currently targets **EP v0.7**. A v2.0.0 branch for **EP v0.8** (with EIP-7702 native support) was initiated in Feb 2025 (#243, #244) but is not yet released.

---

## 3. Core Features

### Default Validator

Introduced in v1.2.0. Previously, every Nexus account required an explicit validator to be specified on every initialization flow. The default validator concept allows an account to have a "fallback" validator — the K1 Validator (secp256k1 ECDSA) in v1.x, the **STX Validator** going forward — which handles validation when no explicit validator is specified in the UserOp `nonce`.

This is critical for EIP-7702 flows and PREP mode, where the initialization must be as lightweight as possible.

### Module Enable Mode

A mechanism allowing a UserOp to install a new module **within the same UserOp** that first uses it — no separate setup transaction required. The module is installed and used atomically.

**Important context:** Module Enable Mode was designed before MEE existed, solving a specific "install before first use" UX problem in a single-chain world. With MEE's SuperTransaction model, this problem is handled differently (module installation can be a preceding step in a SuperTransaction). Module Enable Mode has seen **very little usage in practice** and is a candidate for removal in future versions.

### Pre-validation Hooks (Resource Locking)

The primary motivation for introducing pre-validation hooks in v1.2.0 was **resource locking** for MEE's chain-abstracted execution environment. When a user submits a SuperTransaction that executes across multiple chains, a resource-locking hook can prevent the same funds from being double-spent by rejecting UserOps that conflict with an in-flight cross-chain operation. The hook runs before validation so the lock is enforced regardless of which validator is used.

### Native Composability Support

Nexus v1.2.0 introduced `ComposableExecutionBase` — a built-in execution layer that allows runtime parameter injection without requiring a separate ERC-7579 Executor module. Accounts that inherit `ComposableExecutionBase` can process composability instructions natively. See **Chapter 5** for full composability details.

### PREP Mode (Provably Rootless EIP-7702 Proxy)

PREP is a novel initialization mechanism for EIP-7702 accounts. See **Chapter 15** for full details. At the Nexus level: `handlePREP()` is a function on the account that processes a signed initialization proof, bootstrapping the account as a Nexus without relying on a factory or trusted deployer. Introduced in v1.2.0 (#246), with a length check fix from the Pashov audit (#265).

### ERC-7739 (Nested Typed Data Signing)

Nexus supports ERC-7739 for secure ERC-1271 typed data verification. See **Chapter 8** for full details. The feature was progressively hardened through v1.0.x and fully stabilized in the Cantina ERC-7739 audit cycle.

### ERC-7779 (Redelegation)

Support for the redelegation standard, allowing accounts to safely transfer delegation from one implementation to another. Introduced in v1.0.x (#231) and maintained in the `erc7579-implementation` reference repo.

### MultiType Module Installation

A single `installModule` call can register a contract as multiple module types simultaneously. This avoids the need for multiple transactions when installing a module that serves multiple roles (e.g., a contract that is both an executor and a hook).

---

## 4. Implementation Details

### Validation flow

```
EntryPoint.handleOps()
  → Nexus.validateUserOp()
      → [pre-validation hooks]
      → decode nonce to extract validator address
      → validator.validateUserOp()
      → return validationData
  → [EntryPoint pays prefund]
  → Nexus.execute() / executeBatch()
      → [pre-execution hooks]
      → execute calls
      → [post-execution hooks]
```

The validator address is encoded in the upper bits of the UserOp `nonce` (ERC-4337 nonce key mechanism). If no validator is encoded, the default validator is used.

### Module storage

Installed modules are tracked in enumerable linked lists (sentinels), one per module type. This allows iterating over installed modules — critical for correct uninstallation. An early audit finding (Smart Sessions, also relevant here) showed that associated array libraries used for module storage could have incorrect storage layout; this was fixed in the v1.2.0 cycle.

### Uninstallation constraints

Nexus enforces that the **last initialized validator cannot be uninstalled** (M-02 in Pashov audit, fixed in #268). Uninstalling all validators would leave the account permanently inaccessible. The check traverses the validator linked list to confirm at least one validator remains after removal.

### Bootstrap flows

`NexusBootstrap.sol` provides stateless helper functions for complex initialization sequences — installing multiple modules atomically during account creation. Bootstrap data is ABI-encoded and passed as `initData` to the factory. v1.2.0 revamped the bootstrapper to correctly support pre-validation hooks during initialization (Pashov L-12, #270), and removed registry-specific methods from the bootstrap contract (#261) to simplify the interface.

### Registry integration

Early versions of Nexus included `RegistryAdapter.sol` for optional ERC-7484 module registry queries — allowing accounts to verify that a module being installed is known to a trusted registry. In practice this feature was **not adopted**, added complexity, and introduced several audit findings (storage collisions, factory issues). Later versions significantly reduced registry integration; the registry factory was removed entirely (L-05, #266), and registry methods were removed from the bootstrap (#261). The registry adapter itself remains but as a slimmed-down, namespaced-storage version.

### Initcode front-running prevention

PR #233 addressed a vulnerability where an attacker could observe a pending `initCode` in the mempool and front-run the deployment with a different initialization, hijacking the deterministic CREATE2 address. The fix ensures the account validates its own initialization data as part of the first UserOp.

---

## 5. Design Choices (Brief)

- **ERC-7579 over custom module standard** — interoperability with third-party module ecosystems; Filio co-authored the standard.
- **Singleton + proxy** — standard pattern for gas-efficient account deployment; storage isolation per user.
- **ERC-7201 namespaced storage** — prevents storage collisions in the proxy/singleton architecture.
- **Registry optional, then progressively removed** — the registry concept added audit surface and operational complexity without driving adoption. Future versions move away from it entirely.
- **Pre-validation hooks over post-only** — required specifically for resource locking in MEE; post-execution hooks alone cannot prevent a UserOp from being executed if it conflicts with a cross-chain lock.
- **Pragma `>=` instead of `=`** — changed in #288 to avoid deployment friction when downstream consumers have different compiler version requirements.

---

## 6. Usage Details

### Standard account creation

```
BiconomyMetaFactory → NexusAccountFactory.createAccount(initData, salt)
  → deploys NexusProxy
  → calls proxy.initializeAccount(initData)
    → NexusBootstrap decodes initData
    → installs validator(s), executor(s), hooks
```

The `initData` is constructed using `NexusBootstrapLib` helpers. The simplest case is a single K1 (or STX) validator with an owner EOA address.

### EIP-7702 account creation

Two paths:

1. **Factory path** — EOA signs an EIP-7702 authorization designating Nexus implementation, then calls factory to initialize.
2. **PREP path** — EOA signs a PREP proof off-chain; account self-initializes on first use. No factory transaction required. See Chapter 15.

### Installing modules

Modules are installed via `account.installModule(moduleTypeId, moduleAddress, initData)`. This can be done:

- During initial bootstrap (via factory initData)
- In a subsequent UserOp calling `installModule` directly
- Via Module Enable Mode (within the same UserOp that first uses the module — see section 3, and the caveat below)

### Module Enable Mode flow

The UserOp signature includes an embedded enable-mode payload containing the module address, type, init data, and a validity signature. During `validateUserOp`, if the nonce signals enable mode, the account installs the module before delegating validation to it.

**Caveat:** As noted above, this feature is a candidate for removal. Do not build new flows that depend on it.

### Interacting with hooks

Hooks implement `preCheck(address msgSender, uint256 msgValue, bytes calldata msgData)` and `postCheck(bytes calldata hookData)`. The pre-check receives full call context; the post-check receives the return value from pre-check (for stateful coordination between pre and post).

---

## 7. Caveats & Known Limitations

**Hook bypass vectors (all patched in v1.2.0):**
- Bootstrap could bypass hook installation checks — fixed by revamping bootstrapper flow (#270).
- Module Enable Mode could bypass hooks during installation — fixed (#267).
- `setRegistry()` call did not pass through hooks — fixed to always hook (#259).
- Double-hooking was possible in certain installation paths — added deduplication guard (#253).

**Last validator constraint:** Attempting to uninstall the last validator will revert. Systems that programmatically manage validators must account for this — install the replacement before uninstalling the current one.

**Reinit via relay (v1.2.1):** An issue was found where a relayer could trigger reinitialization of an already-initialized account under specific conditions. Fixed in v1.2.1 PR #293.

**Module Enable Mode reliability:** The feature works but has seen near-zero production usage. Maintain skepticism about edge cases in the enable-mode validation path. It may be removed in a future major version.

**Registry methods removed from bootstrap:** If any off-chain tooling generates bootstrap data using registry-specific bootstrap functions, those functions no longer exist as of v1.2.0. Regenerate initData using the current `NexusBootstrapLib` API.

**CREATE2 address determinism depends on initData:** The factory uses `keccak256(abi.encode(initData, salt))` as the CREATE2 salt. Any change to the initData (e.g., changing the validator address, adding a module) will change the counterfactual address. Front-ends that pre-compute addresses must use exactly the same initData at deployment time.

---

## 8. Scope for Improvement

**EP v0.8 + EIP-7702 native (v2.0.0 — in progress):** The EP v0.8 branch (#243, #244) introduces native EIP-7702 support with cleaner init flows and removes technical debt from the EP v0.7 compatibility layer. This is the primary active development direction.

**Module Enable Mode removal:** The feature adds complexity to the validation path, has near-zero adoption, and is superseded by MEE's SuperTransaction model. A clean removal would simplify both the codebase and the audit surface.

**STX Validator as default (replacing K1):** The K1 Validator is the current default. The STX Validator (Chapter 4) is the intended long-term default, supporting additional signature schemes including P256/passkeys. Migration path needs to be defined.

**Further registry reduction:** The registry adapter is still present but largely inert. Full removal would reduce contract size and eliminate the remaining adapter-related storage footprint.

---

## 9. Version History (Brief)

| Version | Date | Key changes |
|---|---|---|
| v1.0.0 | mid-2024 | Initial release. Module Enable Mode, MultiType installation. Cantina/Cyfrin audit cycle. |
| v1.0.1 | Nov 2024 | ERC-7739 hardening (7739 as external dep, sig malleability fix). |
| v1.0.2 | Mar 2025 | `onERCxxxReceived` hotfix. K1 validator version revert. |
| v1.2.0 | Mar–Apr 2025 | Major release. PREP mode, default validator, pre-validation hooks, native composability, bootstrapper revamp, Pashov/Zenith audit remediation (~15 fixes), NexusProxy `receive()`. |
| v1.2.1 | Oct 2025 | Reinit-via-relay fix. Maintenance. |
| v1.3.0/v1.3.1 | Nov 2025 (MEE Suite v2.2.1) | Native token runtime injection in composability. EIP-712 signing for smart-account mode (replaces blind STX hash). ERC-7702 accounts now initializable via relayers. Gas optimizations. |
| v2.0.0 | In progress | EP v0.8, EIP-7702 native flows, cleaner init. |

---

## 10. Useful Links

| Resource | URL |
|---|---|
| Current repo (stx-contracts) | https://github.com/bcnmy/stx-contracts |
| Legacy nexus repo | https://github.com/bcnmy/nexus |
| ERC-7579 standard | https://eips.ethereum.org/EIPS/eip-7579 |
| ERC-4337 standard | https://eips.ethereum.org/EIPS/eip-4337 |
| Abstract docs (deployed addresses) | https://github.com/bcnmy/abstract-docs |
| Nexus audit reports (in repo) | https://github.com/bcnmy/nexus/tree/main/audits |
| AA benchmarks | https://github.com/bcnmy/aa-benchmarks |
| PR reference (this KT series) | knowledge-transfer-pr-reference.md — Chapter 1 |
