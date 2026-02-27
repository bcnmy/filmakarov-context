# Chapter 11 — AbstractJS SDK: Composability / Runtime Injection

**Knowledge Transfer Document**  
**Author:** Fil Makarov (@filmakarov)  
**Repo:** `bcnmy/abstractjs`  
**Role:** Primary author of composability SDK surface  
**PRs:** ~5  
**Related chapters:** Chapter 5 (Composability Contracts), Chapter 12 (MEE 3.0.0 / Fusion Modes)

---

## 1. Overview

This chapter covers the TypeScript SDK surface within `bcnmy/abstractjs` that enables developers to use the on-chain Composability contracts (Chapter 5) from the frontend. The core idea behind composability is runtime parameter injection — allowing the output of one on-chain call to feed as input to the next call within the same execution bundle, without requiring custom intermediary contracts.

The SDK provides encoding helpers that let developers declare these injection points in TypeScript. The SDK then produces the correctly encoded calldata that the on-chain Composability Module (for existing ERC-7579 accounts) or Composable Execution Base (for Nexus-native support) can interpret and execute.

---

## 2. Architecture Context

Runtime injection solves a fundamental problem in composable on-chain execution: when building a batch of calls, you often don't know the exact parameters for a later call until an earlier call has executed. For example, a swap's output amount is unknown until the swap executes, but you may need that amount as input to a subsequent deposit call.

The composability system works in two layers:

**On-chain (Chapter 5):** The Composability Module / Composable Execution Base executes a sequence of calls. Between calls, it can read return data from previous calls and inject specified values into the calldata (or target address, or ETH value) of subsequent calls. The injection points and source locations are defined in metadata that accompanies the execution batch.

**SDK (this chapter):** The developer defines the execution sequence in TypeScript, marking which parameters should be injected at runtime rather than hardcoded. The SDK encodes these declarations into the metadata format the on-chain contracts expect. The developer works with familiar ABI types and function signatures — the SDK handles the low-level offset calculations and packed encoding.

The key abstraction is that a parameter in a call's arguments can be either a **static value** (known at SDK time) or a **runtime value** (to be filled from a previous call's return data at execution time). The SDK encoding distinguishes between these two cases and produces the correct injection instructions.

---

## 3. Core SDK Primitives

### 3a. `runtimeEncodeAbiParameters` Helper

PR #62 introduced the foundational utility for encoding runtime-injected parameters. This function works similarly to viem's `encodeAbiParameters` but allows certain parameter slots to be marked as runtime-injected rather than statically provided.

When a parameter is marked for runtime injection, the SDK encodes a placeholder along with metadata specifying where the actual value should come from (which previous call's return data, at what offset, and of what type). The on-chain module reads this metadata during execution and overwrites the placeholder with the real value.

This helper is the building block that all higher-level composability SDK features build upon.

### 3b. Efficiency Mode for Input Params

PR #63 added an optimized encoding path for common injection patterns. In many real-world cases, the injection pattern is straightforward — e.g., "take the first return value of the previous call and use it as the second argument of the next call." Efficiency mode reduces the encoding overhead for these standard patterns by using a more compact metadata representation.

This matters for gas costs — the injection metadata is part of the calldata sent on-chain, so more compact encoding directly reduces transaction costs. Efficiency mode is opt-in; the SDK falls back to the full encoding for complex or non-standard injection patterns.

### 3c. Target and Value Runtime Injection

PR #141 extended runtime injection beyond calldata parameters to also support injecting the **target address** and **ETH value** of a call. Prior to this, only function arguments within calldata could be runtime-injected — the target contract address and any native value sent with the call had to be statically known.

This extension enables more advanced patterns, such as dynamically routing a call to a contract address returned by a previous call, or sending an ETH amount determined by a prior computation.

---

## 4. Cross-Chain Composability: Across Intent Wrapper

PR #123 added an SDK wrapper for composable cross-chain executions using the Across bridging protocol. This builds on the contract-side Across Intent support (Chapter 5, composability PR #8) and provides a TypeScript interface for constructing bridging intents that participate in composable execution flows.

PR #127 complemented this with spoke pool addresses and fee configuration arguments, providing the necessary on-chain routing data for Across intents across supported chains.

For the contract-level details of how Across intents integrate with the Composability contracts, see Chapter 5.

---

## 5. Caveats and Known Limitations

- **Encoding alignment with contracts.** The SDK's encoding of injection metadata must exactly match what the on-chain Composability Module expects. Any changes to the on-chain metadata format (e.g., due to a contract upgrade or the move from `mee-contracts` → `composability` → `stx-contracts`) require corresponding SDK updates. The encoding is not self-describing — a mismatch will silently produce incorrect execution rather than a clean revert.

- **Efficiency mode applicability.** Efficiency mode only applies to a subset of injection patterns. Developers using complex multi-source injections or injecting into nested struct fields should verify whether their pattern is supported by efficiency mode, or fall back to the standard encoding path.

- **Return data dependencies are sequential.** Runtime injection can only reference return data from calls that have already executed within the same batch. Circular or forward-referencing dependencies are not possible. The SDK does not currently validate dependency ordering — an incorrectly ordered batch will fail at execution time on-chain.

- **Target and value injection interactions.** When combining target/value injection with calldata injection in the same call, the metadata encoding becomes more complex. Developers should test these combinations carefully, as the on-chain pointer restoration logic (see stx-contracts PR #16, `Fix [L3] restore pointers`) had bugs in early versions that have since been fixed.

- **Across intent wrapper scope.** The Across wrapper provides a convenience layer but does not abstract away all bridging complexity. Developers still need to handle spoke pool address selection, fee estimation, and relay timing considerations. The wrapper handles the composability encoding aspect — how to embed an Across intent within a composable execution batch.

---

## 6. PR Reference

| Date | Title | Link |
|------|-------|------|
| 2025-09-10 | feat: `target` and `value` runtime injection | https://github.com/bcnmy/abstractjs/pull/141 |
| 2025-08-14 | chore: add fees arg and spoke pool addresses | https://github.com/bcnmy/abstractjs/pull/127 |
| 2025-07-24 | feat: across intent wrapper | https://github.com/bcnmy/abstractjs/pull/123 |
| 2025-04-17 | feat: add efficiency mode for input params | https://github.com/bcnmy/abstractjs/pull/63 |
| 2025-04-17 | feat: add runtimeEncodeAbiParameters helper | https://github.com/bcnmy/abstractjs/pull/62 |
