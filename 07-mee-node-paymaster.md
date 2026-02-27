# Chapter 7 — MEE Node Paymaster

**Product:** Node Paymaster & Node Paymaster Factory  
**Repo:** `bcnmy/stx-contracts` (previously `bcnmy/mee-contracts`)  
**Author:** Fil Makarov (@filmakarov)  
**Status:** Production  

---

## Purpose

The Node Paymaster is paymaster infrastructure purpose-built for Biconomy's Modular Execution Environment (MEE). It allows MEE Nodes to pay gas fees for UserOperations that are part of SuperTransactions (Stx) they execute on behalf of users.

In the MEE architecture, nodes are the executors of cross-chain SuperTransactions. When a node picks up a SuperTransaction and submits its constituent UserOps on-chain, it needs a way to cover gas costs, recover those costs from the appropriate party (user or dApp), and earn a premium for its services. The Node Paymaster provides all of this in a single ERC-4337-compliant paymaster contract, validated through the standard EntryPoint.

---

## Architecture & Structure

### Contract Hierarchy

The Node Paymaster is composed of three main contracts:

- **`BaseNodePaymaster`** — Abstract base contract that inherits from the ERC-4337 `IPaymaster` interface and Ownable. Implements the core paymaster logic: `validatePaymasterUserOp` and `postOp` hooks, refund calculations, premium models, and access control. Holds the refund receiver mode and premium configuration state.

- **`NodePaymaster`** — Concrete implementation that extends `BaseNodePaymaster`. Adds worker EOA whitelisting functionality and is the contract actually deployed via the factory. On deployment, ownership is transferred to the node operator's master EOA.

- **`NodePaymasterFactory`** — Factory contract that deploys `NodePaymaster` instances using CREATE2 for deterministic, counterfactual addresses. Supports atomic deploy-and-fund flows so a node operator can deploy their paymaster and stake it at the EntryPoint in a single transaction.

### Deployment Pattern

Each MEE Node deploys its **own** `NodePaymaster` instance. This is a deliberate per-node isolation design — each node has its own deposit at the EntryPoint, its own access control list, and its own premium configuration. The factory uses CREATE2 with the node's master EOA address (and whitelisted worker EOAs) as part of the salt, making addresses deterministic and predictable before deployment.

```
NodePaymasterFactory
    │
    ├── CREATE2 ──> NodePaymaster (Node A)  ──> EntryPoint deposit
    ├── CREATE2 ──> NodePaymaster (Node B)  ──> EntryPoint deposit
    └── CREATE2 ──> NodePaymaster (Node C)  ──> EntryPoint deposit
```

### Relationship to the MEE Stack

```
User signs SuperTransaction
        │
        ▼
   MEE Node picks up Stx
        │
        ▼
   Node submits UserOps on-chain
        │
        ├── userOp.paymasterAndData = NodePaymaster address + mode config
        │
        ▼
   EntryPoint calls NodePaymaster.validatePaymasterUserOp()
        │
        ▼
   EntryPoint executes UserOp
        │
        ▼
   EntryPoint calls NodePaymaster.postOp()
        │
        └── Refund calculation + premium deduction
```

---

## Core Features

### Refund Receiver Modes

The paymaster supports three distinct modes for determining who bears the gas cost and who receives the refund for unused gas:

- **USER mode** — The user is the gas sponsor. The user pre-pays an estimated maximum gas cost (plus premium) as part of the SuperTransaction flow. After execution, unused gas is refunded to the user.

- **DAPP mode** — A dApp sponsor pays for the gas. The dApp pre-pays the estimated cost, and the refund for unused gas goes back to the dApp address.

- **KEEP mode (Sponsorship)** — The node itself sponsors the gas. No refund is issued to anyone; the node absorbs the full cost. This enables fully gasless flows for end users, where the node (or the party funding the node) covers everything.

### Premium Models

The node earns revenue through a configurable premium on top of the actual gas cost:

- **Percentage premium** — The premium is a percentage of the `actualGasCost`. For example, a 10% premium means if the actual gas cost was 0.001 ETH, the total charge is 0.0011 ETH.

- **Fixed premium** — The premium is a fixed amount on top of the `actualGasCost`, regardless of how much gas was used.

These two models give node operators flexibility in their pricing strategy. The premium parameters are set at the paymaster level by the node owner.

### Access Control via tx.origin

The Node Paymaster restricts which EOAs can trigger UserOp sponsorship through `tx.origin` checks. Only the node owner's **master EOA** and explicitly **whitelisted worker EOAs** are authorized.

This is an intentional design decision that makes the paymaster **incompatible with public ERC-4337 mempools** — but that's fine, because MEE operates a private node infrastructure where proven nodes submit transactions directly, not through a public mempool. The MEE network has its own trust model with slashing mechanisms for malicious behavior.

Worker EOA whitelisting allows node operators to scale horizontally — they can run multiple bundler instances, each with its own hot wallet, while all sharing the same paymaster deposit.

Worker EOAs are managed through:
- Batch whitelisting at deployment time (passed to the factory)
- `whitelistWorkerEOAs(address[])` — batch add after deployment
- `removeWorkerEOA(address)` — revoke a worker
- Both functions are `onlyOwner`

### Factory: Deploy-and-Fund

The `NodePaymasterFactory` provides a `deployAndFundNodePaymaster` function that:

1. Deploys a new `NodePaymaster` via CREATE2
2. Forwards attached ETH as the initial EntryPoint deposit
3. Returns the deployed address

A companion `getNodePaymasterAddress` function returns the counterfactual address before deployment, enabling the node to know its paymaster address ahead of time (useful for configuring the node software, pre-computing `paymasterAndData` values, etc.).

---

## Implementation Details & Design Choices

### Why Per-Node Isolation

Each node gets its own paymaster instance rather than sharing a singleton. This was chosen because:
- Each node manages its own deposit at the EntryPoint independently
- Nodes have different pricing strategies (premium models/amounts)
- Access control is per-node (each node has its own set of worker EOAs)
- Failure isolation: one node's deposit running out doesn't affect others
- Clean accounting for each node operator

### The `isNotEOA` Check

The access control originally used an `isContract` check, but this was changed to `isNotEOA` (PR #35). The reason is subtle: `isContract` checks `code.length > 0`, which returns false for EOAs but also for contracts during construction. The `isNotEOA` framing more accurately captures the intent — the paymaster needs to know whether `tx.origin` is an authorized EOA, and the negative check (`isNotEOA`) is semantically clearer and handles edge cases better.

### Removal of Implied Cost Mode

An early version of the paymaster had an "implied cost mode" where the gas cost was inferred rather than explicitly passed. This was removed (PR #34) in favor of the explicit refund receiver modes (USER, DAPP, KEEP), which are clearer and less error-prone.

### Refund Sanity Checks

A sanity check was added before refund calculations (PR #36) to prevent edge cases where the calculated refund could underflow or produce incorrect values. This is a defensive measure against unexpected gas accounting scenarios.

### Abandoned Approach: Master EOA Signature in paymasterAndData

An approach was explored (PR #40, closed) where the node's master EOA would sign the UserOp hash, and this signature would be appended to `userOp.paymasterAndData` for validation. This was abandoned because in the MEE flow, the node selects which worker EOA will submit the transaction **after** the user has already signed. Since the user's signature covers `paymasterAndData`, this data must be fixed before user signing. The tx.origin-based approach avoids this chicken-and-egg problem entirely.

---

## Usage Details

### Deploying a Node Paymaster

A node operator interacts with the `NodePaymasterFactory`:

1. Call `getNodePaymasterAddress(masterEOA, workerEOAs, salt)` to compute the counterfactual address
2. Call `deployAndFundNodePaymaster(masterEOA, workerEOAs, salt)` with ETH attached for the initial EntryPoint deposit
3. The returned address is the node's paymaster, ready to use

### Configuring in UserOps

When the MEE node constructs a UserOp for submission, it sets `userOp.paymasterAndData` to include:
- The node's `NodePaymaster` address
- The refund mode (USER / DAPP / KEEP)
- Premium configuration
- Any mode-specific data (e.g., refund receiver address for DAPP mode)

### Managing Worker EOAs

```solidity
// Add multiple workers at once
nodePaymaster.whitelistWorkerEOAs([worker1, worker2, worker3]);

// Remove a specific worker
nodePaymaster.removeWorkerEOA(worker1);
```

### Topping Up the Deposit

The node operator can top up the EntryPoint deposit at any time by sending ETH to the paymaster's `deposit()` function or directly through the EntryPoint's `depositTo`.

---

## Caveats

- **Not compatible with public ERC-4337 mempools.** The tx.origin access control means only the node's own EOAs can trigger sponsorship. This is by design for MEE's private infrastructure.

- **Deposit management is manual.** Node operators must monitor and top up their EntryPoint deposits. If the deposit runs out, UserOps using this paymaster will fail validation.

- **Premium configuration is per-paymaster, not per-UserOp.** If a node wants different pricing for different scenarios, it cannot change premium mid-transaction. The premium model applies uniformly to all UserOps processed by that paymaster instance.

- **Worker EOA whitelisting is on-chain.** Adding or removing workers requires a transaction from the owner. For nodes that frequently rotate hot wallets, this is an operational consideration.

---

## Scope for Improvement

- **Dynamic premium per-UserOp:** Allowing premium configuration to be passed in `paymasterAndData` rather than being a global paymaster setting would give nodes more pricing flexibility.

- **Automatic deposit monitoring:** Integration with off-chain monitoring to alert or auto-top-up when deposits are running low.

- **Multi-EntryPoint support:** Currently assumes a single EntryPoint. If ERC-4337 evolves with new EntryPoint versions, the paymaster may need to support multiple.

---

## Useful Links

### Pull Requests (chronological)

| Date | Title | Link |
|------|-------|------|
| 2025-01-22 | Pre deposited mode for MEE EP | https://github.com/bcnmy/mee-contracts/pull/6 |
| 2025-01-27 | Node Paymaster | https://github.com/bcnmy/mee-contracts/pull/7 |
| 2025-03-19 | Sponsorship mode | https://github.com/bcnmy/mee-contracts/pull/31 |
| 2025-03-31 | New base struct for node PM contract | https://github.com/bcnmy/mee-contracts/pull/32 |
| 2025-04-04 | Add Paymaster factory for Nodes | https://github.com/bcnmy/mee-contracts/pull/33 |
| 2025-04-08 | Remove implied cost mode | https://github.com/bcnmy/mee-contracts/pull/34 |
| 2025-04-18 | fix: isContract => is notEOA | https://github.com/bcnmy/mee-contracts/pull/35 |
| 2025-04-18 | fix: add sanity check before refund calculation | https://github.com/bcnmy/mee-contracts/pull/36 |
| 2025-07-11 | feat: node eoa master => userOp.pmAndData (closed) | https://github.com/bcnmy/mee-contracts/pull/40 |
| 2025-07-15 | Feat/whitelist worker eoas | https://github.com/bcnmy/mee-contracts/pull/41 |

### Repository

- **Current location:** https://github.com/bcnmy/stx-contracts (contracts directory)
- **Previous location:** https://github.com/bcnmy/mee-contracts

### Documentation

- Node Paymaster Factory deployment docs: https://github.com/bcnmy/abstract-docs/pull/32
