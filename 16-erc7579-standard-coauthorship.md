# Chapter 16 — ERC-7579 Standard Co-authorship

**Knowledge Transfer Document**
**Author:** Fil Makarov (@filmakarov)
**Repo:** `ethereum/ERCs`
**Role:** Co-author

---

## What Is ERC-7579?

[ERC-7579: Minimal Modular Smart Accounts](https://eips.ethereum.org/EIPS/eip-7579) defines the minimally required interfaces and behavior for modular smart accounts and modules to ensure interoperability across the ecosystem.

Co-authors: zeroknots, filmakarov, kopy-kat, leekt, yaonam, rockmin216.

## Why It Matters to Biconomy

ERC-7579 is the foundational standard underlying Biconomy's entire smart account stack. Nexus is an ERC-7579 implementation, and all modules (validators, executors, hooks, fallback handlers) conform to the standard's module type system. Smart Sessions, the Composability Module, and the STX Validator all rely on ERC-7579's interfaces.

Filio's co-authorship means he shaped the standard itself, not just Biconomy's implementation of it. Design decisions in Nexus — such as Module Enable Mode, module type identifiers, the bootstrapper pattern, and the hook lifecycle — trace directly to ERC-7579's specification.

## Filio's Contributions to the Standard

- **Module type system** — the `uint256` module type identifiers (1 = Validator, 2 = Executor, 3 = Fallback, 4 = Hook) and the `isModuleType` check pattern.
- **Module Enable Mode** — the mechanism allowing modules to be installed via a UserOp signature without a separate on-chain transaction. Filio implemented this in Nexus (PR #107) and the pattern was formalized in the standard.
- **`installModule` / `uninstallModule` lifecycle** — the standard interface for module management that all ERC-7579 accounts must implement.
- **Execution modes** — the `ModeCode` encoding for call type (single, batch, delegatecall) and exec type.
- **Collaboration with Rhinestone** — ERC-7579 was a joint effort between Biconomy and Rhinestone (zeroknots). The Smart Sessions module in `erc7579/smartsessions` is a direct product of this collaboration.

## Practical Implications

Anyone modifying Nexus's module management, adding new module types, or building new modules should read ERC-7579 directly. The canonical spec lives at:

- **EIP text:** https://eips.ethereum.org/EIPS/eip-7579
- **Reference implementation:** `erc7579/erc7579-implementation`

Changes to how Nexus handles module installation, validation, or execution should be checked against the standard to maintain compliance and cross-ecosystem interoperability (e.g., with Rhinestone's module registry).

## Related Chapters

- **Chapter 1 (Nexus)** — the primary ERC-7579 implementation
- **Chapter 2 (Smart Sessions)** — ERC-7579 validator module
- **Chapter 5 (Composability)** — ERC-7579 executor module
- **Chapter 17 (ERC-7779)** — builds on ERC-7579 for redelegation
