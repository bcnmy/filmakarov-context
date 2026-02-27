# Chapter 18 — EnumerableMap Library

**Repo:** [`erc7579/enumerablemap`](https://github.com/erc7579/enumerablemap)  
**Package name:** `@erc7579/enumerablemap4337`  
**Language:** Solidity  
**License:** Provided as-is, no warranty

---

## 1. Purpose

The `erc7579/enumerablemap` repository provides Solidity libraries for ERC-4337 compatible dynamic data structures in storage. Standard Solidity dynamic arrays and OpenZeppelin's `EnumerableSet`/`EnumerableMap` are not natively compatible with ERC-4337's associated storage access rules defined in [ERC-7562](https://eips.ethereum.org/EIPS/eip-7562). This library solves that problem by implementing custom storage slot computation that satisfies the ERC-7562 validation rules.

---

## 2. The Problem: Associated Storage and the Double-Keccak Issue

### ERC-7562 Associated Storage Rules

ERC-7562 defines the validation scope rules for Account Abstraction. During UserOperation validation, bundlers enforce strict storage access rules to prevent denial-of-service attacks. A storage slot is considered **"associated"** with address `A` if:

1. The slot value is `A`, or
2. The slot value was calculated as `keccak(A||x) + n`, where `x` is a `bytes32` value and `n` is in the range `0..128`.

Modules (validators, executors, hooks) running during validation can only access storage slots associated with the smart account address that is being validated.

### Why Standard Dynamic Arrays Fail

When a dynamic array is stored as a value in a mapping where the smart account address `A` is the key, Solidity's native storage layout produces a **double keccak**:

- The mapping slot is `keccak(A || baseSlot)` — this is the length slot and is compliant.
- But the array elements start at `keccak(keccak(A || baseSlot)) + n` — a second hash wrapping the first.

The resulting slot `keccak(keccak(A||x)) + n` does **not** match the required `keccak(A||x) + n` pattern from ERC-7562. Bundlers will reject any UserOperation whose validation touches these non-compliant storage slots, making standard dynamic arrays unusable inside ERC-7579 modules that run during validation.

The same problem applies to OpenZeppelin's `EnumerableSet` and `EnumerableMap`, which rely on Solidity's native dynamic array storage layout internally.

---

## 3. The Solution: Custom Slot Computation

The libraries in this repository replace Solidity's native dynamic array storage layout with a custom scheme. Instead of letting the compiler derive slots through its standard hashing rules, the library uses inline assembly to compute the starting slot for array elements directly as `keccak256(abi.encodePacked(account, baseSlot))`. This produces slots of the form `keccak(A||x) + n`, which is exactly the pattern ERC-7562 requires for associated storage.

All three libraries in the repo apply this technique to different data structure types.

---

## 4. What's Included

The repository contains three libraries:

### AssociatedArrayLib

A library for dynamic arrays that are associated with an address per ERC-7562. It replaces Solidity's native dynamic array by computing the starting storage slot as `keccak(A||x)`, where `A` is the associated account address and `x` is the base storage slot. Array elements are stored at sequential offsets from this starting slot.

This is the foundational building block — the other two libraries build on the same storage approach.

### EnumerableSet4337

A fork of OpenZeppelin's `EnumerableSet` that makes all storage access ERC-4337 compliant via associated storage. Beyond the slot computation change, it stores indexes in a mapping to make access to a given value efficient (O(1) for contains, add, remove).

### EnumerableMap4337

A library for managing an enumerable variant of Solidity's `mapping` type, with all storage access patterns made ERC-4337 compliant. This provides the same O(1) set/get/remove operations and O(n) enumeration as OpenZeppelin's `EnumerableMap`, but with storage slots that satisfy ERC-7562 validation rules.

---

## 5. Usage

### Installation

The library is available as a git dependency:

```json
{
  "dependencies": {
    "@erc7579/enumerablemap4337": "https://github.com/erc7579/enumerablemap"
  }
}
```

Or as a Foundry git submodule:

```bash
forge install erc7579/enumerablemap
```

### When to Use

Use these libraries whenever you are building an ERC-7579 module (validator, executor, hook, or fallback handler) that:

- Stores per-account dynamic collections (lists of keys, policies, permissions, etc.)
- Has storage that is accessed during ERC-4337 validation (i.e., during `validateUserOp` or related validation frames)
- Needs to enumerate stored values (iterate over all items)

If your module only uses simple `mapping(address => value)` types or fixed-size storage, standard Solidity storage is already ERC-7562 compliant and these libraries are not needed.

---

## 6. Where It's Used

The library is a dependency across several contracts in the Biconomy and ERC-7579 ecosystem:

- **Smart Sessions** (`erc7579/smartsessions`) — Uses `AssociatedArrayLib` and enumerable structures to manage per-account session data, policy lists, and permission tracking. This is the primary and most complex consumer.

- **ERC-7739 Validator** (`erc7579/erc7739Validator`) — Listed as a direct dependency in `package.json` for managing per-account validator state.

- **STX Contracts** (`bcnmy/stx-contracts`) — The unified Biconomy contract suite that includes Nexus, the STX Validator, and composability contracts. Uses the library for ERC-4337 compliant data management within validation-phase modules.

---

## 7. Design Choices

**Fork of OpenZeppelin rather than wrapper.** The `EnumerableSet4337` and `EnumerableMap4337` libraries are forks of OpenZeppelin's equivalents rather than wrappers around them. This was necessary because the storage layout changes are fundamental — you can't wrap an OZ `EnumerableSet` and redirect its storage slots without modifying the internal implementation.

**Account address as explicit parameter.** Unlike standard Solidity mappings where the compiler handles slot derivation implicitly, these libraries require the associated account address to be passed explicitly to storage operations. This is a deliberate design choice: the libraries need to know which address `A` to use in the `keccak(A||x)` computation, and there's no way to infer it automatically at the Solidity level.

**ERC-7562 range compliance.** The `+ n` offset for array elements is kept within the `0..128` range as specified by ERC-7562. This imposes a practical limit on how many sequential storage slots can be accessed from a single base slot during validation.

---

## 8. Relationship to ERC-7579 Module Architecture

In the ERC-7579 modular account architecture, modules are external contracts that are called by the smart account. When a module stores per-account data, it must use a storage pattern where the account address is the top-level mapping key. This naturally produces `keccak(A||x)` for simple mappings, which is compliant.

The problem arises specifically when modules need **dynamic collections per account** — for example, a session module that stores a variable-length list of active session keys for each account, or a list of policies attached to each permission. These require dynamic arrays or sets, and that's where these libraries become essential.

Without them, the only alternative would be to avoid dynamic arrays entirely during validation and use fixed-size storage patterns, which is impractical for systems like Smart Sessions that inherently manage variable-length collections of policies, signers, and permissions.

---

## PR Reference

| Date | Title | Link |
|------|-------|------|
| 2024-09-24 | upd code | https://github.com/erc7579/enumerablemap/pull/1 |

**Related PRs in Smart Sessions that work with the library:**

| Date | Title | Link |
|------|-------|------|
| 2024-10-16 | Fixing the Associated Array Lib storage layout | https://github.com/erc7579/smartsessions/pull/129 |
| 2024-09-02 | AssociatedArray Lib fixes | https://github.com/erc7579/smartsessions/pull/66 |
