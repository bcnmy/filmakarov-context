# Chapter 20 — Security & Audit Coordination

**Knowledge Transfer Document**
**Author:** Fil Makarov (@filmakarov)
**Cross-cutting across all repos**
**Role:** Primary audit coordinator and remediation author
**PRs:** ~30+ audit-related

---

## Overview

Filio coordinated and remediated security findings across every major Biconomy smart contract audit over two years. This chapter consolidates the audit history, the firms involved, what was found, and patterns to be aware of going forward.

## Audit History by Product

### Nexus Smart Account

**Cantina Code** — Initial Nexus audit and ERC-7739 addon audit.
- Spearbit/Cantina finding #9 fix (PR #144)
- ERC-7739 remediations and report (PRs #216, #218)

**Cyfrin / CodeHawks Competition** (Sep 2024)
- Community audit competition. Finding 5.4.1 returndata fix (PR #197).

**Zenith** — Nexus v1.2.0 audit.
- Issues 4, 5, 6 fixed (PR #257)
- Report added to repo (PR #263)

**Pashov Review** (Mar 2025) — Nexus v1.2.0, ~15 findings.
This was the densest single audit remediation cycle. Findings and fixes:

| Severity | Finding | Fix PR |
|----------|---------|--------|
| M-01 | Hook bypass for module installation via module enable mode | #267 |
| M-02 | Last inited validator check on uninstalling validator | #268 |
| L-01 | Use namespaced storage in RegistryAdapter | #264 |
| L-02 | Length check in `handlePREP()` | #265 |
| L-05 | Remove registry factory (+L-06, L-07) | #266 |
| L-09 | Modules may fail to be uninstalled | #274 |
| L-10 | Hook check bypass / remove double hooking check | #275 |
| L-12 | Revamp bootstrapper flows to support prevalidation hooks | #270 |
| L-13 | Bypassing hooking with bootstrap | #269 |
| L-14 | Enable mode typehash | #273 |

Additional Nexus security work:
- Initcode front-running prevention (PR #233)
- Fix hooking the `fallback()` (PR #183)
- TSTORE validator for `_checkRegistry` PoC (PR #184) — security research exploring transient storage for registry checks
- Reinit via relay fix [L-01] (PR #293, v1.2.1)

### MEE / K1 Validator

Audit covering the K1 Validator and related contracts. Findings and fixes:

| Finding | Fix PR |
|---------|--------|
| Issues #3 and #9 | #13 |
| Issue #8 — potential underflow in calculateRefund | #14 |
| RLP_ENCODED_R_S_BYTE_SIZE can be incorrect | #15 |
| Permit can be frontrun (issue #10) | #16 |
| Low-risk #7 and informationals | #17 |
| [M-01] Struct encoding | #46 |
| [L-03] Domain separator prevents cross-chain SuperTx signatures | #47 |

### STX Contracts

Audit of the unified stx-contracts repo:

| Severity | Finding | Fix PR |
|----------|---------|--------|
| H-1 | Incorrect hashing | #14 |
| L-1, L-2 | Events/errors | #15 |
| L-3 | Restore pointers | #16 |

### Composability Contracts

Two rounds — an initial security review in mee-contracts, then a dedicated audit after extraction to its own repo:

| Severity | Finding | Fix PR | Repo |
|----------|---------|--------|------|
| C-01 | Missing delegatecall check | #1 | composability |
| L-03 | Set EP in the constructor | #2 | composability |
| L-06 | Storage layout for call and delegatecall | #3 | composability |
| L-02 | Length check for balance paramData | #9 | composability |
| Info/Low 1,2,4,5,7 | Security review fixes | #24 | mee-contracts |

### Gasdaddy / Token Paymaster (Chainlight)

| Finding | Fix PR |
|---------|--------|
| Chainlight 005 — withdrawal delay and min deposit | #27 |
| Chainlight 006 — unused gas penalty | #26 |

### Smart Sessions (Cantina/Spearbit)

Major batch remediation (PR #83) plus individual finding fixes:

| Finding | Fix PR |
|---------|--------|
| Cantina/Spearbit batch | #83 |
| Finding 15 | #116 |
| Issue 19 | #118 |
| Issues 20-21 | #119 |
| Issue 23 | #120 |

### SCW v2 (Legacy)

- Aggregated audit remediations (PR #196)
- SMA-379 ABI SVM audit remediations pt 1 (PR #197)

## Recurring Security Patterns

Across all audits, certain categories of findings recurred. These are the themes a new engineer should watch for:

**Hook/module bypass paths** — Multiple findings (M-01, L-10, L-13 in Nexus) involved ways to install or interact with modules that bypassed the hook system. Any new code path that touches module installation or execution must be checked against the hook lifecycle.

**Storage layout issues** — Findings in Nexus (L-01), Composability (L-06), and Smart Sessions (#129) all involved storage slot collisions or incorrect namespacing. Biconomy uses ERC-7201 namespaced storage throughout — any new storage must follow this pattern.

**Hashing and encoding correctness** — The STX H-1 (incorrect hashing), MEE M-01 (struct encoding), and multiple typehash fixes show that EIP-712 typed data hashing is error-prone. Always verify typehashes against the actual struct definitions and test cross-chain compatibility.

**Initialization and re-initialization** — Initcode front-running (#233), reinit via relay (#293), and PREP length checks (#265) all relate to account initialization security. The initialization path is a high-risk surface.

**Uninstallation edge cases** — M-02 (last validator check) and L-09 (modules failing to uninstall) show that module removal paths need as much scrutiny as installation paths.

## Audit Firms Used

| Firm | Products Audited |
|------|-----------------|
| Cantina Code | Nexus, ERC-7739 addon |
| Spearbit | Smart Sessions (joint with Cantina) |
| Cyfrin/CodeHawks | Nexus (competition) |
| Pashov | Nexus v1.2.0 |
| Zenith | Nexus v1.2.0 |
| Chainlight | Gasdaddy |
| Internal/unnamed | MEE contracts, STX contracts, Composability |

## Where Audit Reports Live

- **All historical audits:** `stx-contracts/audits/` — this is the consolidated location for all audit reports across products
- **Documentation:** `abstract-docs` and `documentation` repos link to audit reports
- **Nexus README** also links to audits (PR #280)

## Related Chapters

- Every product chapter (1–9) contains the specific audit findings for that product
- **Chapter 19 (Documentation)** — audit report links in docs
- **Chapter 15 (PREP)** — L-02 length check finding
- **Chapter 8 (ERC-7739)** — dedicated Cantina audit for the 7739 addon
