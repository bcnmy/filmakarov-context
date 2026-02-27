# Chapter 17 — ERC-7779 Support (Redelegation)

**Knowledge Transfer Document**
**Author:** Fil Makarov (@filmakarov)
**Repos:** `bcnmy/nexus`, `erc7579/erc7579-implementation`
**Role:** Implementer
**PRs:** 3

---

## What Is ERC-7779?

ERC-7779 ("Interoperable Delegated Accounts") standardizes how EIP-7702 delegated EOAs manage storage and handle redelegation — the process of switching from one smart account implementation to another.

The core problem: when an EOA delegates to a smart account via EIP-7702, that account uses storage slots on the EOA's address. If the user later wants to switch to a different smart account implementation (e.g., from a competitor's account to Nexus, or vice versa), leftover storage from the old implementation could collide with the new one, causing undefined behavior, locked accounts, or asset loss.

ERC-7779 solves this with two interfaces:

- **`IInteroperableDelegatedAccount`** — exposes the account's storage base slots so wallets can verify compatibility before redelegation.
- **`IRedelegableDelegatedAccount`** — provides an `onRedelegation()` hook that cleans up storage before switching implementations.

## Filio's Implementation

### PR #231 — ERC-7779 support v0.1 (Nexus)

Initial integration into Nexus. Added the ERC-7779 interfaces to Nexus so it can:
1. Report its storage base slots (Nexus uses ERC-7201 namespaced storage)
2. Handle `onRedelegation()` to clean up account state when a user wants to migrate away from Nexus

### PR #41 — 7779 bases management + use in MSA (erc7579-implementation)

Added storage base management to the reference ERC-7579 Modular Smart Account implementation. This ensures the standard reference code properly tracks and reports storage bases, making it available to any ERC-7579 account, not just Nexus.

### PR #50 — Fix import for dependency usage (erc7579-implementation)

Cleanup fix so the ERC-7779 code works correctly when consumed as a dependency by other projects.

## Why This Matters

ERC-7779 is a **no-vendor-lock-in guarantee**. Biconomy prominently references it alongside EIP-7702 in their marketing — users can delegate their EOA to Nexus and later switch to another implementation without getting stuck. This is a competitive differentiator: Nexus was among the first smart accounts to implement ERC-7779 support.

The standard also interacts with PREP mode (Chapter 15): when an EOA uses PREP to bootstrap itself as a Nexus account, ERC-7779 ensures that same EOA can cleanly migrate away if needed.

## Caveats

- ERC-7779 is best-effort — `onRedelegation()` doesn't guarantee complete storage cleanup, especially for mappings and dynamic structures. It cleans up what it can.
- The standard requires proper authentication before executing `onRedelegation()` to prevent malicious cleanup.
- Storage base verification relies on both the old and new implementations honestly reporting their slots. There's no on-chain enforcement of accuracy.
- ERC-7779 was still in draft status during Filio's implementation. The spec may evolve.

## Related Chapters

- **Chapter 1 (Nexus)** — the account that implements ERC-7779
- **Chapter 15 (PREP Mode)** — EIP-7702 initialization that ERC-7779 complements
- **Chapter 16 (ERC-7579)** — the modular account standard that ERC-7779 builds on
