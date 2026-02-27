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
| 5 | 05-composability-contracts.md | Composability Contracts & Runtime Injection | Pending |
| 6 | 06-gasdaddy-token-paymaster.md | Gasdaddy / Token Paymaster | Pending |
| 7 | 07-mee-node-paymaster.md | MEE Node Paymaster | Pending |
| 8 | 08-erc7739-integration.md | ERC-7739 Typed Data Signing Integration | Pending |
| 9 | 09-scw-v2.md | SCW Contracts v2 (Legacy Smart Account) | Pending |
| 10 | 10-abstractjs-sdk.md | AbstractJS SDK Support | Pending |
| 11 | 11-composability-runtime-injection.md | Composability / Runtime Injection Details | Pending |
| 12 | 12-mee3-stx-fusion.md | MEE 3.0.0 / STX Validator / Fusion Modes | Pending |
| 13 | 13-erc7739-typed-data.md | ERC-7739 Typed Data Signing (Standalone) | Pending |
| 14 | 14-deployment-scripts.md | Deployment Scripts & Multi-chain Deploys | Pending |
| 15 | 15-prep-mode.md | PREP Mode | Pending |
| 16 | 16-erc7579-standard.md | ERC-7579 Standard Co-authorship | Pending |
| 17 | 17-erc7779-support.md | ERC-7779 Redelegation Support | Pending |
| 18 | 18-enumerable-map.md | EnumerableMap Library | Pending |
| 19 | 19-documentation.md | Documentation Contributions | Pending |
| 20 | 20-security-audit.md | Security & Audit Coordination | Pending |
| 21 | 21-test-infra-dx.md | Test Infrastructure & Developer Experience | Pending |

See also: [pr-reference-list.md](pr-reference-list.md) — cross-reference mapping every PR to its chapter.

## Covered Systems & Standards

**Smart Contract Systems:** Nexus Smart Account, Smart Sessions Module, MEE K1 Validator, STX Validator, Composability Contracts, Gasdaddy / Token Paymaster, MEE Node Paymaster, SCW v2, Deployment Scripts

**Standards:** ERC-4337, ERC-7579, ERC-7702, ERC-7739, ERC-7779, ERC-7780, EIP-712, EIP-2612

## Stats

- ~220 pull requests documented
- ~15 repositories across `bcnmy` and `erc7579` orgs
- 5 development phases spanning Feb 2024 – Feb 2026
