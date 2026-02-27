# Filmakarov Knowledge Transfer

Technical knowledge transfer repository documenting ~220 pull requests authored by [@filmakarov](https://github.com/filmakarov) across the `bcnmy` and `erc7579` GitHub organisations over 24 months (February 2024 – February 2026).

## Purpose

This repository captures architecture decisions, implementation details, design rationale, and audit history for all major smart contract systems built during this period. It is structured so that team members (or AI systems) can look up any component and understand how it works, why it was built that way, and where to find the relevant code.

## How to Use

Start with [000-start-here.md](000-start-here.md) — it contains the master index of all work, organized by product area and chronological phase. From there, follow links to individual chapter files for deep dives.

## Chapters

| # | File | Topic | Status |
|---|------|-------|--------|
| 0 | [000-start-here.md](000-start-here.md) | Master index & overview | Done |
| 1 | [01-nexus-smart-account.md](01-nexus-smart-account.md) | Nexus Smart Account (ERC-7579 modular account) | Done |
| 2 | [02-smart-sessions.md](02-smart-sessions.md) | Smart Sessions Module (session key management) | Done |
| 3 | [03-mee-k1-validator.md](03-mee-k1-validator.md) | MEE K1 Validator (SuperTransaction validation) | Done |
| 4 | [04-stx-validator.md](04-stx-validator.md) | STX Validator (next-gen MEE validator, MEE 3.0.0) | Done |
| 5 | [05-composability-contracts.md](05-composability-contracts.md) | Composability Contracts | Done |
| 6 | [06-gasdaddy-token-paymaster.md](06-gasdaddy-token-paymaster.md) | Gasdaddy / Token Paymaster | Done |
| 7 | [07-mee-node-paymaster.md](07-mee-node-paymaster.md) | MEE Node Paymaster | Done |
| 8 | [08-erc7739-integration.md](08-erc7739-integration.md) | ERC-7739 Integration (Nested Typed Data Signing) | Done |
| 9 | [09-scw-contracts-v2.md](09-scw-contracts-v2.md) | SCW Contracts v2 (Legacy Biconomy Smart Account) | Done |
| 10 | [10-abstractjs-smart-sessions.md](10-abstractjs-smart-sessions.md) | AbstractJS SDK: Smart Sessions Support | Done |
| 11 | [11-abstractjs-composability.md](11-abstractjs-composability.md) | AbstractJS SDK: Composability / Runtime Injection | Done |
| 12 | [12-abstractjs-mee-stx-fusion.md](12-abstractjs-mee-stx-fusion.md) | AbstractJS SDK: MEE 3.0.0 / STX Validator / Fusion Modes | Done |
| 13 | [13-abstractjs-erc7739-signing.md](13-abstractjs-erc7739-signing.md) | AbstractJS SDK: ERC-7739 Typed Data Signing | Done |
| 14 | [14-deployment-scripts.md](14-deployment-scripts.md) | Deployment Scripts | Done |
| 15 | [15-prep-mode.md](15-prep-mode.md) | PREP Mode (Provably Rootless EIP-7702 Proxy) | Done |
| 16 | [16-erc7579-standard-coauthorship.md](16-erc7579-standard-coauthorship.md) | ERC-7579 Standard Co-authorship | Done |
| 17 | [17-erc7779-redelegation.md](17-erc7779-redelegation.md) | ERC-7779 Support (Redelegation) | Done |
| 18 | [18-enumerablemap-library.md](18-enumerablemap-library.md) | EnumerableMap Library | Done |
| 19 | [19-documentation.md](19-documentation.md) | Documentation | Done |
| 20 | [20-security-audit-coordination.md](20-security-audit-coordination.md) | Security & Audit Coordination | Done |
| 21 | [21-test-infrastructure.md](21-test-infrastructure.md) | Test Infrastructure & Developer Experience | Done |

See also: [pr-reference-list.md](pr-reference-list.md) — cross-reference mapping every PR to its chapter.

## Covered Systems & Standards

**Smart Contract Systems:** Nexus Smart Account, Smart Sessions Module, MEE K1 Validator, STX Validator, Composability Contracts, Gasdaddy / Token Paymaster, MEE Node Paymaster, SCW v2, Deployment Scripts

**Standards:** ERC-4337, ERC-7579, ERC-7702, ERC-7739, ERC-7779, ERC-7780, EIP-712, EIP-2612

## Stats

- ~220 pull requests documented
- ~15 repositories across `bcnmy` and `erc7579` orgs
- 5 development phases spanning Feb 2024 – Feb 2026
