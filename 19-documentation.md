# Chapter 19 — Documentation

**Knowledge Transfer Document**
**Author:** Fil Makarov (@filmakarov)
**Repos:** `bcnmy/abstract-docs`, `bcnmy/documentation` (legacy), `bcnmy/nexus`
**Role:** Primary maintainer of deployment addresses and technical docs
**PRs:** ~25

---

## Overview

Filio maintained two generations of Biconomy's developer documentation and was responsible for keeping deployment addresses, chain support, audit references, and technical guides up to date across every release.

## Repos

### `bcnmy/abstract-docs` (current)

The active documentation site for Biconomy's Abstract stack. Filio's contributions fall into three categories:

**Chain & address additions** — Every time contracts were deployed to a new chain, Filio updated the docs with addresses. Chains added include: Apechain (#28), Lisk (#43), Unichain Katana (#42), Soneium (#12), Monad testnet (#7), Sonic/Blast (#9 in legacy docs), and others. Release-specific address updates: genesis release (#13), Nexus 1.0.2 (#9), Nexus 1.2.0 (#14), bootstrap address change (#17).

**Technical guides:**
- Runtime injection function descriptions and examples (#22) — documents how to use the Composability contracts from the SDK
- v2-to-Nexus upgrade docs with initdata generation snippets (#33, #40) — migration guide from SCW v2 to Nexus
- Smart Sessions docs fixes (#37)
- Node Paymaster Factory docs (#32)
- MEE versioning switch (#45)

**Infrastructure:** WalletConnect addition (#44), steps script fix (#39), header fix (#15), ordering changes (#16).

### `bcnmy/documentation` (legacy)

The older Biconomy documentation site. Filio's contributions here were primarily address and deployment tracking: paymaster addresses (#120, #137), Nexus addresses and audit links (#112, #113, #115), Sonic/Blast/SPM fixes (#134).

### `bcnmy/nexus` (README & audits)

- Clean README and add audit links (#280)
- Zenith audit report addition (#263)
- ERC-7739 audit report addition (#218)

## What a New Engineer Needs to Know

- **Deployment addresses live in `abstract-docs`** — when deploying to a new chain, update the docs repo with the new addresses. Follow the pattern established in existing chain addition PRs.
- **Every release needs a docs update** — new contract versions require updated addresses, and if the release changes behavior, the technical guides need updating too.
- **Audit reports are tracked in two places**: the `nexus/audits/` directory (PDF reports) and the docs site (links and summaries). Both need updating when a new audit completes.
- **The docs transitioned from `documentation` to `abstract-docs`** — the legacy repo is still live but the active one is `abstract-docs`. New content goes there.
- **MEE versioning** was adopted in the docs (#45) — contract suite versions are now tracked under MEE version numbers rather than individual contract versions.

## Related Chapters

- **Chapter 14 (Deployment Scripts)** — the scripts that produce the addresses documented here
- **Chapter 20 (Security & Audit Coordination)** — audit reports referenced in docs
- All product chapters (1–8) — each has corresponding documentation that Filio maintained
