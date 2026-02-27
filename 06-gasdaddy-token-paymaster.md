# Chapter 6 — Gasdaddy / Token Paymaster

**Repo:** `bcnmy/gasdaddy`  
**Role:** Primary developer, released v1.0.0-p  
**Standards:** ERC-4337 Paymaster  

---

## Purpose

Gasdaddy is Biconomy's Token Paymaster — an ERC-4337 paymaster contract that enables smart account users to pay gas fees in ERC-20 tokens instead of native ETH. This removes a major UX barrier: users no longer need to hold ETH on every chain just to transact.

The paymaster sits between the EntryPoint and the smart account in the ERC-4337 flow. It fronts the gas cost in ETH to the EntryPoint, then charges the user's smart account the equivalent amount in their chosen ERC-20 token.

---

## Architecture & Structure

### Pricing Modes

Gasdaddy supports two distinct pricing modes for determining how much token to charge for a given gas cost:

**1. External Mode**  
Price data is provided from off-chain. The bundler or relayer supplies a signed price quote alongside the UserOp. This allows for flexible, real-time pricing without on-chain oracle dependency.

**2. Independent Mode**  
Uses on-chain oracles (Chainlink price feeds / TWAP) to determine the token-to-ETH exchange rate autonomously. No off-chain infrastructure required — the paymaster reads the price directly from the oracle during execution.

### Core Flow

1. User submits a UserOp specifying the token they want to pay with
2. During validation, the paymaster verifies the user has sufficient token balance and the pricing data is valid
3. The EntryPoint executes the UserOp, with the paymaster covering gas in ETH
4. In `postOp`, the paymaster charges the smart account the actual gas cost (converted to tokens) plus any applicable markup

---

## Core Features

### PostOp Charging

The paymaster charges the user in the `postOp` phase rather than during validation. This is important because the actual gas consumed is only known after execution completes. Charging in `postOp` means users pay for what they actually use rather than a worst-case estimate.

*PRs: #43 (Charge user in postOp), #35 (Fix how token PM charges SA)*

### Markup and Time-Per-Token Pricing

The paymaster supports per-token configuration of pricing markup and time-based parameters. This allows operators to set different fee margins for different tokens and adjust pricing freshness requirements on a per-token basis.

*PR: #42 (Markup and Time per token)*

### Bulk Refund

A method for batch-processing refunds, enabling the paymaster operator to efficiently return overcharged amounts across multiple users in a single transaction.

*PR: #31 (Add bulk refund method)*

### Withdrawal Delay and Minimum Deposit

Security mechanisms introduced from the Chainlight audit:

- **Withdrawal delay:** Prevents the paymaster operator from instantly draining funds. There is a time-lock between initiating and completing a withdrawal, giving users time to react.
- **Minimum deposit:** Ensures the paymaster always maintains a minimum ETH deposit in the EntryPoint, preventing a situation where the paymaster accepts UserOps but cannot cover them.

*PR: #27 (Fix Chainlight 005)*

### Unused Gas Penalty

A penalty mechanism for UserOps that significantly overestimate their gas usage. This disincentivizes gas griefing attacks where a malicious user submits UserOps with inflated gas limits to lock up paymaster funds without actually consuming the gas.

*PR: #26 (Fix Chainlight 006)*

---

## Security — Chainlight Audit

Gasdaddy underwent a security audit by Chainlight. Key findings and remediations:

| Finding | Fix | PR |
|---------|-----|----|
| Chainlight 005 — Withdrawal security | Introduced withdrawal delay + minimum deposit | [#27](https://github.com/bcnmy/gasdaddy/pull/27) |
| Chainlight 006 — Gas griefing vector | Added unused gas penalty | [#26](https://github.com/bcnmy/gasdaddy/pull/26) |

Additional code quality improvements were made in PRs #41 (Fix Todos and Reviews) and #44 (review PR).

---

## Caveats

- Token support depends on available oracle infrastructure (Independent mode) or off-chain pricing infrastructure (External mode)
- The paymaster operator must maintain sufficient ETH deposits in the EntryPoint to cover gas for incoming UserOps
- Markup and pricing parameters require active management by the operator to remain competitive and solvent

---

## Useful Links

| Resource | Link |
|----------|------|
| Repository | `bcnmy/gasdaddy` |
| Fix Chainlight 005 — Withdrawal delay + min deposit | [PR #27](https://github.com/bcnmy/gasdaddy/pull/27) |
| Fix Chainlight 006 — Unused gas penalty | [PR #26](https://github.com/bcnmy/gasdaddy/pull/26) |
| Bulk refund method | [PR #31](https://github.com/bcnmy/gasdaddy/pull/31) |
| Fix how token PM charges SA | [PR #35](https://github.com/bcnmy/gasdaddy/pull/35) |
| Markup and time per token | [PR #42](https://github.com/bcnmy/gasdaddy/pull/42) |
| Charge user in postOp | [PR #43](https://github.com/bcnmy/gasdaddy/pull/43) |
| Code review and cleanups | [PR #41](https://github.com/bcnmy/gasdaddy/pull/41), [PR #44](https://github.com/bcnmy/gasdaddy/pull/44) |
