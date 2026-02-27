# Chapter 14 — Deployment Scripts

**Knowledge Transfer Document**
**Author:** Fil Makarov (@filmakarov)
**Date:** February 2026
**Repos:** `bcnmy/stx-contracts`, `bcnmy/nexus`, `bcnmy/mee-contracts`, `bcnmy/composability`

---

## Purpose

The deployment scripts are the infrastructure that deploys the entire Biconomy smart contract suite — Nexus, K1/STX Validator, Composability Module, Node Paymaster, and Smart Sessions — across all supported EVM chains. They use deterministic CREATE2 deployment via Safe Singleton Deployer and CreateX, ensuring the same bytecode + same salt produces the same contract address on every chain.

This document provides a high-level orientation for both script iterations and a practical guide on how to modify the scripts when new contracts are added. Each script iteration has its own detailed README in the repo — this document does not replace those READMEs but rather contextualizes them and explains the overall approach.

---

## Script Iterations

There have been two major iterations of the deployment infrastructure:

1. **Pre-unified (legacy):** Each repo had its own independent deployment script in a `scripts/bash-deploy/` directory.
2. **Unified (current):** A single deployment script in `bcnmy/stx-contracts` covers the entire suite, plus a separate `deploy-ss.sh` for Smart Sessions.

---

## Iteration 1 — Pre-Unified Scripts (Legacy)

### Context

Before the repo unification in November 2025, the contracts were split across three repositories, and each had its own deployment script:

- **mee-contracts:** [`bcnmy/mee-contracts` → `scripts/bash-deploy/`](https://github.com/bcnmy/mee-contracts/tree/deploy/v1.1.0/scripts/bash-deploy)
- **nexus:** [`bcnmy/nexus` → `scripts/bash-deploy/`](https://github.com/bcnmy/nexus/tree/deploy-v1.2.1/scripts/bash-deploy)
- **composability:** [`bcnmy/composability` → `scripts/bash-deploy/`](https://github.com/bcnmy/composability/tree/deploy/v1.0.0/scripts/bash-deploy)

### How to Use

Each script directory has its own `README.MD` with detailed instructions. The general workflow is:

1. Check out the appropriate `deploy/vX.X.X` branch of the repo.
2. Configure the `.env` file with RPC URLs, private keys, and chain-specific settings.
3. Run the bash deploy script, which orchestrates Foundry `forge script` calls.
4. Artifacts (deployed addresses, verification data) are output to the artifacts directory.

### Deployment Order Dependency

When using the legacy per-repo scripts, **deployment order matters:**

1. **Deploy K1 MEE Validator first** from `mee-contracts`. This must be deployed before Nexus because Nexus requires the default validator module address as a constructor argument. If you deploy Nexus before the validator exists on-chain, the deployment will fail.
2. **Deploy Nexus** from the `nexus` repo, providing the K1 MEE Validator address.
3. **Deploy Composability contracts** from the `composability` repo (order-independent relative to Nexus, but typically deployed alongside).

### When to Use

This iteration is now **legacy**. Use it only if you need to deploy a specific historical version from the old repos (e.g., Nexus v1.2.1 from the `nexus` repo, or MEE contracts v1.1.0 from `mee-contracts`). For any new deployment, use the unified script.

---

## Iteration 2 — Unified Script (Current)

### Context

After the unification of all contracts into `bcnmy/stx-contracts` (November 2025), a single deployment script handles the entire suite. This eliminated the deployment order issues of the legacy approach — the unified Foundry script manages all dependencies internally.

### Location

```
bcnmy/stx-contracts/
└── script/deploy/
    ├── README.MD                    # Detailed usage instructions
    ├── DeployStxContracts.s.sol     # Main Foundry deploy script
    ├── deploy-stx.sh               # Main entry point — run this to deploy
    ├── deploy-chain.sh              # Per-chain deployment (called by deploy-stx.sh)
    ├── build-artifacts.sh           # Builds and copies contract artifacts
    ├── deploy-ss.sh                 # Smart Sessions deployment (separate)
    ├── config.toml                  # Per-chain configuration
    ├── .env                         # RPC URLs, private keys, deployer settings
    ├── artifacts/                   # Pre-compiled bytecodes and verification data
    │   ├── Nexus/
    │   ├── NexusBootstrap/
    │   ├── NexusAccountFactory/
    │   ├── NexusProxy/
    │   ├── stx-validator/          # Validator + all submodules
    │   │   ├── StxValidator/
    │   │   └── submodules/
    │   │       ├── NoStxModeVerifier/
    │   │       ├── SimpleModeSubmodule/
    │   │       ├── PermitSubmodule/
    │   │       ├── TxSubmodule/
    │   │       ├── SafeAccountSubmodule/
    │   │       ├── EOAStatelessValidator/
    │   │       └── P256StatelessValidator/
    │   ├── ComposableExecutionModule/
    │   ├── ComposableStorage/
    │   ├── EtherForwarder/
    │   └── NodePaymasterFactory/
    └── util/
        ├── DeterministicDeployerLib.sol
        ├── DeployViaSafeDeployer.s.sol
        └── createx-hex/            # CreateX presigned deployment txs
```

### How to Use

Refer to `script/deploy/README.MD` in the repo for full step-by-step instructions (the README lives on the `deploy/vX.X.X` branches — e.g., check the [`deploy/v2.2.1` branch](https://github.com/bcnmy/stx-contracts/tree/deploy/v2.2.1/script/deploy)). The high-level workflow is:

1. **Check out a deploy branch.** Production deployments should use a `deploy/vX.X.X` branch (e.g., `deploy/v2.2.1`), not `develop`.
2. **Build artifacts (only if contracts changed).** Run `build-artifacts.sh` only when the Solidity source has changed and you're preparing a new deploy branch. When a `deploy/vX.X.X` branch is already ready, the artifacts are pre-committed — rebuilding them is **not recommended** as it risks introducing bytecode differences from compiler version or settings mismatches.
3. **Configure `.env`** with: deployer private key, RPC URLs per chain (as `RPC_{chainId}`), EntryPoint v0.7 deploy tx data (if EP not yet deployed on the target chain), and any chain-specific overrides.
4. **Configure `config.toml` and `foundry.toml` (only if adding a new chain).** See [Adding a New Chain](#adding-a-new-chain-to-the-deployment-config) below for details. If deploying to already-configured chains, skip this step.
5. **Run `deploy-stx.sh`** — this is the main entry point which orchestrates the deployment. Under the hood, it calls `deploy-chain.sh` for each configured chain, which:
   - Ensures prerequisite infrastructure (EntryPoint v0.7, Safe Singleton Deployer, CreateX) is present on the target chain, deploying them if needed.
   - Invokes `forge script DeployStxContracts` to deploy all contracts via CREATE2.
   - Handles verification if configured.
   - Outputs deployed addresses to the console.

### What Gets Deployed

The unified script deploys the following contracts (in dependency order, managed automatically):

| Contract | Description | Constructor Dependencies |
|----------|-------------|------------------------|
| StxValidator submodules (7 contracts) | Mode verifiers + stateless validators | None (standalone singletons) |
| StxValidator | Main validator module | Submodule addresses |
| Nexus | Smart account implementation | EntryPoint, StxValidator address, validator init data |
| NexusBootstrap | Bootstrapper for account init | StxValidator address, validator init data |
| NexusAccountFactory | CREATE2 factory for account proxies | Nexus implementation address, factory owner |
| NexusProxy | Reference proxy instance | Via NexusAccountFactory |
| ComposableExecutionModule | Composability as ERC-7579 module | EntryPoint address |
| ComposableStorage | Storage for composable execution | None |
| EtherForwarder | ETH forwarding utility | None |
| NodePaymasterFactory | Factory for node paymasters | None |

### Key Design Principles

- **CREATE2 determinism:** Same bytecode + same salt = same address across all chains. This is intentional and critical for the multi-chain architecture.
- **Idempotent deployment:** The script checks if each contract already exists at the expected address before deploying. If already deployed, it logs the address and skips.
- **Selective deployment:** The `config.toml` specifies which contracts to deploy per chain. Contracts not in the list are skipped but their addresses are still computed (since other contracts may depend on them).
- **Artifact-based:** Contracts are deployed from pre-compiled JSON artifacts, not compiled during the deployment run. This ensures bytecode consistency and avoids compiler version mismatches.

### Adding a New Chain to the Deployment Config

When you need to deploy to a chain that isn't already configured, two files need updating:

#### 1. `foundry.toml` — Add the RPC endpoint

Add an entry under `[rpc_endpoints]` so Foundry knows how to reach the chain:

```toml
[rpc_endpoints]
# existing chains...
new_chain = "${RPC_12345}"  # references the RPC_<chainId> variable from .env
```

#### 2. `config.toml` — Add the chain configuration

Add a block for the new chain ID with all required settings. The structure follows this pattern:

```toml
[12345]
name = "New Chain"

[12345.bool]
is_testnet = "false"
verify = "true"              # whether to verify on block explorer
via_blockscout = "false"     # set to "true" if the chain uses Blockscout instead of Etherscan
skip_createx = "false"       # set to "true" if CreateX can't be deployed (rare)

[12345.string]
# List which contracts to deploy on this chain
contracts = "StxValidator,Nexus,NexusBootstrap,NexusAccountFactory,NexusProxy,ComposableExecutionModule,ComposableStorage,EtherForwarder,NodePaymasterFactory"
```

The `contracts` list controls which contracts get deployed. Typically you deploy the full suite, but you can omit contracts if they're not needed on a particular chain. The script will still compute their expected addresses (for dependency resolution) but won't attempt deployment.

#### 3. `.env` — Add the RPC URL

Add the RPC URL for the new chain:

```bash
RPC_12345=https://rpc.newchain.io
```

Refer to the `script/deploy/README.MD` (on the deploy branch) for the full list of available configuration options and any chain-specific overrides (gas settings, etc.).

---

## Smart Sessions Deployment Script (`deploy-ss.sh`)

### Context

Smart Sessions contracts originate from the `erc7579/smartsessions` repository. Rather than importing the full Smart Sessions source code into `stx-contracts` and compiling it there, the deployment uses **pre-compiled artifacts** — the bytecode is taken from the `smartsessions` repo's build output and deployed via CREATE2.

The `deploy-ss.sh` script (added in [PR #17](https://github.com/bcnmy/stx-contracts/pull/17), December 2025) handles this deployment as a standalone bash script within the same `script/deploy/` directory.

### How to Use

The script has its own usage instructions within the file. The workflow is similar to the main deploy script: configure environment variables, point to the pre-compiled Smart Sessions artifacts, and run the script for the target chain(s). It uses the same deterministic deployment approach (CREATE2 via Safe Singleton Deployer) to ensure address consistency across chains.

---

## How to Modify the Script — Case Study: Adding STX Validator Support

When a new contract (or a replacement for an existing one) needs to be added to the deployment, the script requires modifications across multiple files. The transition from `K1MeeValidator` to `StxValidator` (done in [PR #18](https://github.com/bcnmy/stx-contracts/pull/18)) is a good real-world illustration. The diff between the `develop` and `feat/stx-validator` branches shows exactly what changed.

### Overview of Changes

The following files in `script/deploy/` were modified:

| File | What Changed |
|------|-------------|
| `DeployStxContracts.s.sol` | Main Foundry script — new contract definitions, salts, deployment functions, constructor args |
| `build-artifacts.sh` | Artifact building — new directories and copy commands for the new contracts |
| `deploy-chain.sh` | Bash orchestrator — minor fixes (Blockscout verification support, `cast send` flag fix) |
| `deploy-ss.sh` | Minor formatting fixes (unrelated to the StxValidator change) |
| `artifacts/stx-validator/` | New directory tree with pre-compiled artifacts for all submodules |

### Step-by-Step Walkthrough

#### 1. Define Salts for New Contracts

Each deterministically deployed contract needs a unique CREATE2 salt. In `DeployStxContracts.s.sol`, the old single salt was replaced with 8 new salts:

```solidity
// Old: single salt for the monolithic validator
bytes32 constant MEE_K1_VALIDATOR_SALT = 0x00000000...;

// New: salt for the main StxValidator + one per submodule
bytes32 constant STX_VALIDATOR_SALT = 0x00000000...;
bytes32 constant NO_STX_MODE_VERIFIER_SALT = 0x00...01;
bytes32 constant SIMPLE_MODE_VERIFIER_SALT = 0x00...02;
bytes32 constant PERMIT_MODE_VERIFIER_SALT = 0x00...03;
bytes32 constant TX_MODE_VERIFIER_SALT = 0x00...04;
bytes32 constant SAFE_ACCOUNT_SUBMODULE_SALT = 0x00...05;
bytes32 constant EOA_STATELESS_VALIDATOR_SALT = 0x00...06;
bytes32 constant P256_STATELESS_VALIDATOR_SALT = 0x00...07;
```

For production, the main validator salt should be "mined" for a vanity address. Submodule salts can be arbitrary since users don't interact with them directly.

#### 2. Add Bytecode Storage Variables

The script loads contract bytecodes from pre-compiled artifacts in `setUp()`. Add a storage variable for each new contract:

```solidity
// Old: single bytecode variable
bytes private meeK1ValidatorBytecode;

// New: main contract + all submodules
bytes private stxValidatorBytecode;
bytes private noStxModeVerifierBytecode;
bytes private simpleModeSubmoduleBytecode;
bytes private permitSubmoduleBytecode;
bytes private txSubmoduleBytecode;
bytes private safeAccountSubmoduleBytecode;
bytes private eoaStatelessValidatorBytecode;
bytes private p256StatelessValidatorBytecode;
```

And load them in `setUp()`:

```solidity
stxValidatorBytecode = vm.getCode("script/deploy/artifacts/stx-validator/StxValidator/StxValidator.json");
noStxModeVerifierBytecode = vm.getCode("script/deploy/artifacts/stx-validator/submodules/NoStxModeVerifier/NoStxModeVerifier.json");
// ... one per submodule
```

#### 3. Add Tracking Structs and Mappings

If the new contract has dependencies (like StxValidator depends on submodule addresses), add a struct to track deployed addresses:

```solidity
struct DeployedSubmodules {
    address noStxModeVerifier;
    address simpleModeVerifier;
    address permitModeVerifier;
    address txModeVerifier;
    address safeAccountSubmodule;
    address eoaStatelessValidator;
    address p256StatelessValidator;
}

mapping(uint256 => DeployedSubmodules) internal deployedSubmodulesPerChain;
```

Update the existing `DeployedContracts` struct to reference the new contract instead of the old one:

```solidity
struct DeployedContracts {
    address stxValidator;  // was: address meeK1Validator;
    // ... rest unchanged
}
```

#### 4. Write Deployment Functions

Create deployment and address-calculation functions for the new contract. The pattern is:

- **`computeSubmoduleAddresses()`** — pure address computation (no deployment), used for dry runs and dependency resolution.
- **`deployAllSubmodules(chainId)`** — deploys each submodule if not already present, with idempotent check-then-deploy logic:

```solidity
function deployAllSubmodules(uint256 chainId) internal {
    address expected;

    expected = DeterministicDeployerLib.computeAddress(noStxModeVerifierBytecode, NO_STX_MODE_VERIFIER_SALT);
    if (expected.code.length == 0) {
        expected = DeterministicDeployerLib.broadcastDeploy(noStxModeVerifierBytecode, NO_STX_MODE_VERIFIER_SALT);
        console.log("    NoStxModeVerifier deployed:", expected);
    } else {
        console.log("    NoStxModeVerifier already deployed:", expected);
    }
    deployedSubmodulesPerChain[chainId].noStxModeVerifier = expected;

    // ... repeat for each submodule
}
```

- **`deployStxValidator(chainId)`** — deploys the main contract using submodule addresses as constructor args:

```solidity
function deployStxValidator(uint256 chainId) internal returns (address) {
    SubmoduleAddresses memory submoduleAddresses = SubmoduleAddresses({
        noStxModeVerifier: deployedSubmodulesPerChain[chainId].noStxModeVerifier,
        simpleModeVerifier: deployedSubmodulesPerChain[chainId].simpleModeVerifier,
        // ... all submodules
    });
    bytes memory args = abi.encode(submoduleAddresses);
    address stxValidator = DeterministicDeployerLib.broadcastDeploy(
        stxValidatorBytecode, args, STX_VALIDATOR_SALT
    );
    return stxValidator;
}
```

#### 5. Update Constructor Arguments for Dependent Contracts

The StxValidator has a different init data format than the old K1MeeValidator. Contracts that take the validator address as a constructor arg (Nexus, NexusBootstrap) need updated encoding:

```solidity
// Old: simple address packing
bytes memory args = abi.encode(ENTRYPOINT_ADDRESS, meeK1ValidatorAddress, abi.encodePacked(EEEEEE_ADDRESS));

// New: StxValidator requires stateless validator address in init data
function _buildStxValidatorInitData(address eoaStatelessValidator) internal pure returns (bytes memory) {
    return abi.encodePacked(
        eoaStatelessValidator,
        uint8(0), // no safe senders
        abi.encodePacked(EEEEEE_ADDRESS) // impossible-to-sign owner for implementation
    );
}

bytes memory stxValidatorInitData = _buildStxValidatorInitData(submoduleAddresses.eoaStatelessValidator);
bytes memory args = abi.encode(ENTRYPOINT_ADDRESS, stxValidatorAddress, stxValidatorInitData);
```

#### 6. Update the Dispatch Logic in `deployContracts()`

The main deployment loop dispatches by contract name. Update the entry for the old contract:

```solidity
// Old:
if (nameMatches("K1MeeValidator")) {
    deployedContractsPerChain[chainId].meeK1Validator = deployK1MeeValidator();
}

// New: deploy submodules first, then the main validator
if (nameMatches("StxValidator")) {
    deployAllSubmodules(chainId);
    deployedContractsPerChain[chainId].stxValidator = deployStxValidator(chainId);
}
```

Note: Submodule deployment is handled internally — the bash script (`deploy-chain.sh`) only needs to know about "StxValidator" as a single deployable unit.

#### 7. Update `build-artifacts.sh`

Add new artifact directories and copy commands for each new contract:

```bash
# Create directories
mkdir -p ./artifacts/stx-validator/StxValidator
mkdir -p ./artifacts/stx-validator/submodules/NoStxModeVerifier
mkdir -p ./artifacts/stx-validator/submodules/SimpleModeSubmodule
# ... one per submodule

# Copy compiled artifacts
cp ../../out/StxValidator.sol/StxValidator.json ./artifacts/stx-validator/StxValidator/.
cp ../../out/NoStxModeVerifier.sol/NoStxModeVerifier.json ./artifacts/stx-validator/submodules/NoStxModeVerifier/.
# ... one per submodule

# Create verification artifacts
forge verify-contract --show-standard-json-input $(cast address-zero) StxValidator \
    > ./artifacts/stx-validator/StxValidator/verify.json
forge verify-contract --show-standard-json-input $(cast address-zero) NoStxModeVerifier \
    > ./artifacts/stx-validator/submodules/NoStxModeVerifier/verify.json
# ... one per submodule
```

#### 8. Summary of the Pattern

When adding a new contract to the deployment:

1. **Define a salt** in `DeployStxContracts.s.sol`.
2. **Add bytecode variable** and load it from artifacts in `setUp()`.
3. **If the contract has sub-components**, add a tracking struct + mapping.
4. **Write `deploy*()` and `calculate*Address()` functions** following the existing pattern.
5. **Update `deployContracts()` dispatch loop** to handle the new contract name.
6. **Update constructor args** for any contracts that depend on the new one.
7. **Update `build-artifacts.sh`** to copy and verify the new contract.
8. **Run `build-artifacts.sh`** to generate the initial artifacts, then commit them.

---

## Related PRs

| Date | Title | Link |
|------|-------|------|
| 2025-12-19 | feat: add ss deployment scripts | https://github.com/bcnmy/stx-contracts/pull/17 |
| 2025-11-18 | deploy: add chains and polish deploy script | https://github.com/bcnmy/stx-contracts/pull/10 |
| 2025-11-17 | Deploy script | https://github.com/bcnmy/stx-contracts/pull/8 |
| 2024-12-10 | Add links to deploy script | https://github.com/bcnmy/nexus/pull/225 |

The STX Validator deployment changes are part of [PR #18 (Feat/stx validator)](https://github.com/bcnmy/stx-contracts/pull/18), which also includes the contract code itself.

---

## Key Caveats & Tips

- **Always use a `deploy/vX.X.X` branch** for production deployments. The `develop` branch may contain work-in-progress changes that haven't been audited.
- **`.env` configuration is critical.** Wrong RPC URL or private key means failed deployment. Double-check chain IDs.
- **CREATE2 determinism** means that if you change any constructor argument, the deployed address changes. This cascades — changing the validator address changes the Nexus address, which changes the factory address, which changes the proxy address.
- **Some chains need special gas handling.** The `deploy-chain.sh` script has configurable gas suffix logic (`GAS_SUFFIX_SEND`, `GAS_SUFFIX_DEPLOY`) and supports chains that require explicit gas pricing.
- **Blockscout verification** is supported via the `via_blockscout` flag in `config.toml` for chains that use Blockscout instead of Etherscan.
- **CreateX deployment** uses pre-signed transactions at different gas limits (3M, 5M, 10M, 30M). The script auto-selects based on the gas estimate and funds the deployer EOA if needed.
- **Idempotent by design.** Re-running the script on a chain where contracts are already deployed will skip existing contracts and only deploy missing ones.
