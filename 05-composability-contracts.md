# Chapter 5 — Composability Contracts

**Repos:** `bcnmy/mee-contracts` → `bcnmy/composability` → `bcnmy/stx-contracts`  
**Author:** Fil Makarov (@filmakarov)  
**Related chapters:** Chapter 1 (Nexus — native composability support), Chapter 11 (AbstractJS SDK — Composability / Runtime Injection)

---

## 1. Purpose

The Composability Contracts enable runtime parameter injection — allowing outputs from one on-chain call to feed as inputs to another within the same execution, without deploying custom smart contracts. This lets developers build multi-step DeFi workflows (swap → approve → deposit, bridge → supply, etc.) entirely from frontend code using the AbstractJS SDK.

---

## 2. Architecture & Structure

### Repo Migration

The composability code originated inside `bcnmy/mee-contracts` (PRs #18–#29), was extracted into a standalone `bcnmy/composability` repo for its own release cycle and audit scope, and was ultimately unified into `bcnmy/stx-contracts` (PR #3) alongside Nexus and the STX Validator.

### Three Components

The system is built around three components that share a common library:

**ComposableExecutionLib** — The core library. Contains all logic for processing input parameters (injection) and output parameters (capture). Both integration surfaces below delegate to this library, so the injection/capture logic is written once.

**ComposableExecutionModule** — An ERC-7579 executor module that can be installed on any ERC-7579 smart account. This is the path for accounts that don't have native composability built in. The account calls (or delegatecalls) into this module, which handles the composable execution flow. Can be installed as an executor module and/or a fallback handler module depending on the account's architecture.

**ComposableExecutionBase** — A base contract that smart accounts can inherit to enable composable execution natively. Nexus v1.2.0+ inherits from this (PR #258 in nexus), so Nexus accounts get composability without installing any additional module.

### How It Fits in the Stack

```
┌─────────────────────────────────────────────┐
│  AbstractJS SDK (Chapter 11)                │
│  runtimeERC20BalanceOf(), buildComposable() │
│  Encodes placeholders + injection metadata  │
└──────────────────┬──────────────────────────┘
                   │ calldata with placeholders
                   ▼
┌─────────────────────────────────────────────┐
│  Nexus Smart Account                        │
│  executeComposable() ← native (Base)        │
│  — or —                                     │
│  ComposableExecutionModule (ERC-7579)       │
└──────────────────┬──────────────────────────┘
                   │ for each step:
                   ▼
┌─────────────────────────────────────────────┐
│  ComposableExecutionLib                     │
│  1. processOutputParams() — capture returns │
│  2. processInputParams() — inject values    │
│  3. Execute the actual call                 │
└─────────────────────────────────────────────┘
```

The SDK encodes which parameters are runtime-injected (placeholders), which storage slots to capture return data into, and any constraints. The on-chain contracts resolve these at execution time.

---

## 3. Core Features

### Return Data Capture

After each call in a composable sequence, the system can capture return values into indexed storage slots. If a function returns multiple values, any subset of them can be stored. These slots are then available for injection into subsequent calls.

### Runtime Parameter Injection

Placeholder bytes in a call's `calldata` are replaced with values read from on-chain state at execution time. The system supports several runtime value sources, exposed via SDK helpers:

| SDK Function | On-chain Source | Introduced |
|---|---|---|
| `runtimeERC20BalanceOf` | `balanceOf(address)` on ERC-20 token | MEE v1.0.0 |
| `runtimeNativeBalanceOf` | Native token balance of an address | MEE v2.2.0 |
| `runtimeERC20AllowanceOf` | `allowance(owner, spender)` on ERC-20 | MEE v1.0.0 |
| `runtimeParamViaCustomStaticCall` | Arbitrary `staticcall` returning ≤32 bytes | MEE v2.2.0 |

### Static Types Handling

Any static Solidity types (`uint256`, `address`, `bytes32`, etc.) can be injected into `abi.encode`-d function calldata. The library understands the ABI encoding layout and replaces the correct 32-byte word at the right offset. This was formalized in mee-contracts PR #21.

### Target and Value Injection

Beyond calldata parameters, the system can inject runtime values into the **target address** and **ETH value** of a call (composability PR #5, abstractjs PR #141). This enables patterns like sending native tokens to a dynamically-determined recipient.

### Constraints

Constraints are on-chain checks that must pass before a composable step executes. If a constraint fails, the step does not execute. Constraints serve double duty:

- **Validation** — ensuring injected values meet criteria (e.g., balance is not zero, amount ≥ minimum)
- **Execution ordering** — in the MEE orchestration context, the orchestrator retries steps until constraints are satisfied, which naturally sequences cross-chain flows (e.g., wait for bridge completion before proceeding)

Constraint types include equality checks, range checks (`greaterThanOrEqualTo`), and non-zero checks.

### Across Intent Composable Executions

Composability PR #8 and abstractjs PR #123 added integration with the Across Protocol bridge, enabling cross-chain composable flows. A user can bridge tokens via Across and then compose subsequent steps on the destination chain that use the bridged amount (resolved at runtime via `runtimeERC20BalanceOf`). The SDK provides an `acrossIntentWrapper` that encodes the bridge instruction alongside the composable execution on the destination chain.

### Native Value Forwarding

The composability module can forward ETH through the execution flow (mee-contracts PR #23). This was a security-relevant addition — the initial implementation didn't properly handle `msg.value`, and the fix ensured native value is correctly passed through to the target call.

---

## 4. Implementation Details

### The Injection Algorithm

The composable execution flow for a sequence of calls works as follows:

1. **Decode the composable batch.** The calldata contains an array of composable execution entries, each specifying: target, value, calldata, output capture instructions, and input injection instructions.

2. **For each entry in order:**
   - **Process inputs** (`processInputParams`): Scan the entry's calldata for placeholder markers. For each placeholder, read the runtime value (from a storage slot previously written by an earlier step, or from an on-chain state read like `balanceOf`). Replace the placeholder bytes in calldata with the actual value. Validate any constraints attached to the parameter.
   - **Execute the call** to the target with the resolved calldata and value.
   - **Process outputs** (`processOutputParams`): If the entry specifies output capture, decode the return data and write the requested values into indexed storage slots for use by later entries.

### Storage Layout (Call vs Delegatecall)

This is the most architecturally significant implementation detail and the source of the critical audit finding (C-01, composability PR #1).

The `ComposableExecutionModule` can be used in two ways:
- **Via `delegatecall`** — the module's code runs in the context of the smart account, so storage writes go to the smart account's storage.
- **Via `call`** — the module's code runs in its own context, so storage writes go to the module contract's storage.

The storage slot derivation in `Storage.sol` accounts for both the **account address** and the **caller address**, ensuring that:
- Different accounts don't collide in the module's storage (when used via `call`)
- The same account using both flows doesn't have cross-contamination

```
// Storage slot = keccak256(abi.encode(account, caller, baseSlot))
// When delegatecall: caller == account (since msg.sender == the account itself)
// When call: caller == the account, but storage lives in the module
```

It is recommended that a smart account consistently uses either `call` or `delegatecall` for the composability module, not both.

The critical audit finding (C-01) was a missing `delegatecall` check — the module didn't properly distinguish between the two contexts, which could lead to storage corruption. PR #1 in the composability repo fixed this.

### EntryPoint Reference

The composability module stores a reference to the ERC-4337 EntryPoint address, set in the constructor (composability PR #2, fixing audit finding L-03). This is needed because the module needs to know when it's being called in the context of an ERC-4337 UserOp execution vs. a direct call.

### Storage Layout Fix for Call and Delegatecall

Audit finding L-06 (composability PR #3) refined the storage layout further, ensuring the slot derivation correctly separates call and delegatecall storage spaces to prevent any overlap.

---

## 5. Design Choices

**Two integration surfaces (Module vs Base):** The module approach works with any ERC-7579 account without requiring changes to the account implementation. The base contract approach is more gas-efficient (no cross-contract call overhead) and was added for Nexus v1.2.0 where native composability was a design goal. Both share the same `ComposableExecutionLib`.

**Library-first architecture:** All composable execution logic lives in `ComposableExecutionLib`, with the module and base contract being thin wrappers. This keeps logic DRY and means fixes/improvements propagate to both surfaces.

**Extraction then re-unification:** The composability code was initially developed inside `mee-contracts` for rapid iteration. It was extracted to `bcnmy/composability` to give it an independent audit scope and release cycle (v1.0.0, v1.1.0). It was later unified into `stx-contracts` when the overall architecture consolidated.

**Placeholder-based injection:** Rather than using a callback pattern (where the composability system would call back into the SDK), the system uses a simpler approach — the SDK pre-encodes placeholders in calldata, and the on-chain code replaces them. This avoids complex on-chain state machines and keeps the gas cost proportional to the number of injections.

---

## 6. Usage Details

### SDK Integration (Chapter 11)

Developers interact with composability through the AbstractJS SDK's `buildComposable()` method:

```typescript
import { runtimeERC20BalanceOf } from "@biconomy/abstractjs";

// Step 1: Swap USDC for WETH (exact amount known)
const swapInstruction = await oNexus.buildComposable({
  type: "default",
  data: {
    chainId: optimism.id,
    to: swapRouter,
    abi: swapRouterAbi,
    functionName: "swapExactTokensForTokens",
    args: [usdcAmount, 0, path, nexusAddress, deadline]
  }
});

// Step 2: Deposit all received WETH into Aave
// Amount unknown until step 1 executes
const depositInstruction = await oNexus.buildComposable({
  type: "default",
  data: {
    chainId: optimism.id,
    to: aavePool,
    abi: aavePoolAbi,
    functionName: "supply",
    args: [
      wethAddress,
      runtimeERC20BalanceOf({
        tokenAddress: wethAddress,
        targetAddress: nexusAddress,
        constraints: [balanceNotZeroConstraint]
      }),
      nexusAddress,
      0
    ]
  }
});
```

The SDK encodes `runtimeERC20BalanceOf` into a placeholder with metadata that tells the on-chain contracts to call `balanceOf(nexusAddress)` on the WETH token and inject the result into the `amount` argument position.

Key SDK helpers (from Chapter 11):
- `runtimeEncodeAbiParameters` (abstractjs PR #62) — Core utility for encoding runtime-injected parameters
- Efficiency mode for input params (abstractjs PR #63) — Optimized encoding for common patterns
- Target and value runtime injection (abstractjs PR #141)
- Across intent wrapper (abstractjs PR #123)

### Module Installation vs Native

**On Nexus v1.2.0+:** No module installation needed. Call `executeComposable()` directly on the Nexus account. The account inherits `ComposableExecutionBase`.

**On other ERC-7579 accounts:** Install `ComposableExecutionModule` as an executor module. The account then calls into the module for composable execution.

---

## 7. Caveats & Known Limitations

### Audit Findings (All Remediated)

| Severity | Finding | Fix |
|----------|---------|-----|
| **Critical** | C-01: Missing delegatecall check — storage corruption possible when module used via delegatecall | composability PR #1 |
| **Low** | L-02: Missing length check for balance `paramData` | composability PR #9 |
| **Low** | L-03: EntryPoint not set in constructor | composability PR #2 |
| **Low** | L-06: Storage layout for call vs delegatecall not fully separated | composability PR #3 |
| **Info/Low** | Items 1,2,4,5,7 from composability security review | mee-contracts PR #24 |

### Constraints on Injection

- `runtimeParamViaCustomStaticCall` is limited to return values ≤ 32 bytes (a single `uint256`, `address`, or `bytes32`). Dynamic types like `bytes` or `string` cannot be injected via this function.
- Only static types can be injected into ABI-encoded calldata. Dynamic types (arrays, strings, bytes) in function arguments require different handling.

### Call vs Delegatecall Consistency

As noted in the storage layout section, mixing `call` and `delegatecall` for the composability module on the same account is not recommended. Storage slots are derived differently for each context, so state written via `call` is not readable via `delegatecall` and vice versa.

### Gas Overhead

The composability layer adds gas overhead for:
- Storage reads/writes for captured return values
- On-chain state reads for runtime values (e.g., `balanceOf` calls)
- Constraint validation checks
- Calldata manipulation (placeholder replacement)

This overhead is generally acceptable for DeFi workflows where the gas savings from avoiding custom contract deployment far outweigh the per-execution cost.

---

## 8. Version History

| Version | Date | Key Changes |
|---------|------|-------------|
| Initial development | Feb–Mar 2025 | Core library, module, and base contract developed in `mee-contracts` (PRs #18–#29) |
| **v1.0.0** | Mar 2025 | First release in `bcnmy/composability` (PR #4). Includes all audit fixes (C-01, L-03, L-06). Target and value injection (#5) |
| **v1.1.0** | Oct 2025 | Across Intent composable executions (#8), L-02 length check fix (#9) |
| Unified into stx-contracts | Nov 2025 | Imported into `bcnmy/stx-contracts` (PR #3) |

---

## 9. Scope for Improvement

- **Dynamic type injection** — Currently limited to static types. Supporting dynamic types (arrays, strings) in ABI-encoded calldata would enable more complex injection patterns.
- **Multi-word return value injection** — `runtimeParamViaCustomStaticCall` is capped at 32 bytes. Supporting wider return values would unlock injection from functions that return structs or multiple values.
- **Gas optimization** — The storage read/write pattern for captured return values could potentially be optimized using transient storage (`TSTORE`/`TLOAD`) since the captured values only need to persist within a single transaction.
- **Richer constraint types** — Adding more built-in constraint types beyond equality and range checks (e.g., bitwise masks, custom predicate calls) could enable more expressive execution conditions.

---

## 10. Useful Links

### Repositories

- **Current home:** [bcnmy/stx-contracts](https://github.com/bcnmy/stx-contracts) (contracts/composability/)
- **Standalone repo (historical):** [bcnmy/composability](https://github.com/bcnmy/composability)
- **Explanatory repo:** [bcnmy/composability-stack-explained](https://github.com/bcnmy/composability-stack-explained)

### Documentation

- [Runtime Parameter Injection (Biconomy Docs)](https://docs.biconomy.io/new/getting-started/understanding-runtime-injection)
- [Understanding Composable Orchestration](https://docs.biconomy.io/new/learn-about-biconomy/understanding-composable-orchestration)
- [Composability Stack blog post](https://blog.biconomy.io/introduction-to-the-composability-stack/)

### PR Reference

**bcnmy/mee-contracts (initial development):**

| Date | Title | Link |
|------|-------|------|
| 2025-02-19 | [Composability] Refactor: Lib + native support | [#18](https://github.com/bcnmy/mee-contracts/pull/18) |
| 2025-02-26 | Composability optimizations | [#20](https://github.com/bcnmy/mee-contracts/pull/20) |
| 2025-02-26 | [Composability] Single chain + static types handling | [#21](https://github.com/bcnmy/mee-contracts/pull/21) |
| 2025-03-10 | [Composability] Forward native value from Composability module (Issue #6) | [#23](https://github.com/bcnmy/mee-contracts/pull/23) |
| 2025-03-10 | [Composability] security review: fix info and low levels [1,2,4,5,7] | [#24](https://github.com/bcnmy/mee-contracts/pull/24) |
| 2025-03-12 | [Composability] Add tests | [#25](https://github.com/bcnmy/mee-contracts/pull/25) |
| 2025-03-13 | [Composability] Introduce Composability Module usage flows w/o fallback | [#26](https://github.com/bcnmy/mee-contracts/pull/26) |
| 2025-03-17 | [Composability] Refactor imports and Natspec | [#27](https://github.com/bcnmy/mee-contracts/pull/27) |
| 2025-03-17 | [Composability] Fix: do not expect msg value | [#28](https://github.com/bcnmy/mee-contracts/pull/28) |
| 2025-03-18 | Remove composability | [#29](https://github.com/bcnmy/mee-contracts/pull/29) |

**bcnmy/composability (standalone):**

| Date | Title | Link |
|------|-------|------|
| 2025-03-27 | Fix [C-01] Missing delegatecall check | [#1](https://github.com/bcnmy/composability/pull/1) |
| 2025-03-27 | Fix [L-03] Set EP in the constructor | [#2](https://github.com/bcnmy/composability/pull/2) |
| 2025-03-27 | Fix [L-06] Storage layout for call and delegatecall | [#3](https://github.com/bcnmy/composability/pull/3) |
| 2025-03-29 | Release/v1.0.0 | [#4](https://github.com/bcnmy/composability/pull/4) |
| 2025-04-01 | Inject Target and Value | [#5](https://github.com/bcnmy/composability/pull/5) |
| 2025-07-24 | Feat: Across Intent composable executions | [#8](https://github.com/bcnmy/composability/pull/8) |
| 2025-10-21 | fix [L-02] length check for balance paramData | [#9](https://github.com/bcnmy/composability/pull/9) |
| 2025-10-31 | Release 1.1.0 | [#10](https://github.com/bcnmy/composability/pull/10) |

**bcnmy/stx-contracts:**

| Date | Title | Link |
|------|-------|------|
| 2025-11-05 | feat: import composability contracts | [#3](https://github.com/bcnmy/stx-contracts/pull/3) |

**Nexus native composability:**

| Date | Title | Link |
|------|-------|------|
| 2025-03-17 | [Nexus 1.2.0] Native Composability Support | [nexus #258](https://github.com/bcnmy/nexus/pull/258) |
