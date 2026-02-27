# Part 15 — PREP Mode (Provably Rootless EIP-7702 Proxy)

## Overview

PREP is a method for deploying smart accounts via EIP-7702 where no one knows the private key associated with the account's EOA address. By combining Nick's method (keyless execution) with EIP-7702 delegation, PREP produces accounts that inherit the best properties of both EOAs (same address everywhere, cryptographic determinism) and smart accounts (multi-sig, resource locks, time-locks, session keys, passkey signers). Nexus v2.0 is the first smart account to support the full PREP feature set.

## The Problem

When an EOA upgrades to a smart account via EIP-7702, the original private key remains a root-level override. The owner can:

1. **Re-delegate or remove delegation** — sign a new EIP-7702 authorization that points the EOA to a different contract (or to nothing), bypassing any smart account restrictions.
2. **Sign off-chain permits** — use the EOA key to sign `ERC20Permit` / `Permit2` messages, moving funds without ever touching the smart account logic.

This makes EIP-7702-upgraded EOAs fundamentally incompatible with multi-sigs, resource locks, time-locks, and other trust-dependent features — because any infrastructure provider knows the EOA key can override everything.

Additionally, the initialization of a delegated smart account is vulnerable to front-running: since EIP-7702 authorization tuples are visible in the mempool, a malicious actor could call `initialize()` on the newly delegated EOA before the legitimate owner, potentially setting themselves as the sole signer.

## How PREP Works

PREP solves both problems (root key override + front-running) by running the EIP-7702 authorization process *in reverse* — generating a valid signature first, then recovering the address from it.

### Step 1 — Construct the EIP-7702 Authorization Message

```
messageHash = keccak256(
  concat(0x05, rlp([0, smart_account_address, 0]))
)
```

- `chain_id = 0` → valid on all chains
- `smart_account_address` → Nexus singleton implementation
- `nonce = 0` → fresh account

### Step 2 — Derive `r` from `initData`

The `r` component of the ECDSA signature is set to the **hash of the initialization data**:

```
r = keccak256(initData)
```

Where `initData` encodes the desired initial configuration (owner address, validator module, bootstrap config, etc.).

**This is the core front-running protection:** since changing `r` changes the recovered address, there is exactly one valid `initData` for any given PREP address. Even if an attacker intercepts the authorization tuple, they can only initialize the account with the configuration the user intended.

### Step 3 — Construct `s` with Magic Prefix (Rootless Proof)

The `s` component is prefixed with a known magic value (`646d6a0000000...`), with the remaining bytes randomized:

```
s = concat(MAGIC_PREFIX, randomBytes(19))
```

The 13-byte magic prefix means brute-forcing a private key that produces this `s` for a given message would require ~2¹⁰³ operations — computationally infeasible.

### Step 4 — Recover the Account Address

```
signature = concat(r, s, v)   // v = 27 or 28, irrelevant
accountAddress = ecrecover(messageHash, signature)
```

Nobody knows the private key for `accountAddress`.

### Step 5 — Post the EIP-7702 Transaction

The authorization tuple `[chain_id, address, nonce, y_parity, r, s]` is included in a Type 4 transaction. The EVM validates the signature, sets the delegation — `accountAddress` is now a Nexus smart account.

### Step 6 — Initialize (Triggered on First UserOp)

Initialization happens automatically during the first UserOperation. The smart contract:

1. Recovers the address from the provided `r`, `s`, `v` values in the init block
2. Verifies the recovered address matches `address(this)`
3. Checks `s` has the magic prefix → sets the `rootless` flag in storage
4. Executes the `initData` (installs owner, validator, bootstrap config)

No separate signature from the user is required — the authorization itself already encodes the full initialization intent.

## The `initData → signature.r → address` Chain

This is the essential insight:

```
initData  →  r = hash(initData)  →  address = ecrecover(msg, r, s, v)
```

- The address is deterministically derived from the init configuration
- Changing *any* init parameter changes the address
- The account can only ever be initialized with the original `initData`
- The magic `s` prefix proves the key was never known

## Integration with Nexus Architecture

PREP slots into the existing Nexus modular architecture:

- **Default Validator** — the K1 or passkey validator installed during PREP initialization becomes the account's signing authority
- **Bootstrapper** — the `initData` references the bootstrapper contract that wires up modules (validators, executors, hooks) in a single call
- **ERC-7579 Modules** — all standard modules work identically on PREP accounts as on CREATE2-deployed accounts
- **`rootless` Storage Flag** — once set during initialization, any external contract or protocol can query whether the account is provably rootless, enabling trust for resource locks, multi-sigs, etc.

## Key Benefits

| Feature | CREATE2 Account | EIP-7702 EOA | PREP Account |
|---------|----------------|--------------|--------------|
| Same address all chains | ✅ | ❌ (per-chain auth) | ✅ |
| No root key override | ✅ | ❌ | ✅ |
| Multi-sig / Resource locks | ✅ | ❌ | ✅ |
| Gas-efficient deployment | ~150k gas | ~70k gas | **~30k gas** (up to 80% savings) |
| Unified SDK tooling | Separate flow | Separate flow | **Same flow as 7702 EOAs** |
| Zero-signature init | ❌ (needs deploy tx) | ❌ (needs init tx) | ✅ |

## Security Considerations

**Pashov Audit — L-02 (Length Check):** The audit identified a missing length validation on the init block data passed during PREP initialization. Without proper length checks, malformed `initData` could lead to unexpected behavior during `ecrecover`. This was addressed in PR #265.

**Front-running is structurally prevented:** Since `r = hash(initData)`, an attacker who intercepts the authorization can only replay it with the same init configuration — they cannot substitute their own owner.

**Not quantum-resistant:** Like all ECDSA-based schemes, a sufficiently powerful quantum computer could derive the private key from the public information. This is not PREP-specific — it applies to EIP-7702 and Nick's method in general. When Ethereum adds post-quantum signatures, PREP would need to adapt accordingly.

**Chain-specific nonce considerations:** Using `chain_id = 0` means the same authorization is valid on all chains. The `nonce = 0` assumption means this only works for fresh accounts — which is the intended use case.

## Caveats & Known Limitations

- **Not portable by default** — PREP accounts cannot be imported into wallets via seed phrase (no private key exists). A portability standard would require industry coordination around discoverability and ownership transfer mechanisms.
- **One-shot initialization** — the `nonce = 0` + `chain_id = 0` combination means PREP is designed for new account creation only, not for upgrading existing EOAs.
- **`ERC20Permit` safety depends on rootless proof** — protocols must check the `rootless` flag to confirm PREP status; the flag itself is trusted because of the cryptographic guarantee from the magic `s` prefix.

## Related PRs

- **#246** — Initial PREP implementation in Nexus
- **#265** — Fix for Pashov audit L-02 (init block length validation)

## References

- [PREP Deep Dive — Biconomy Blog](https://blog.biconomy.io/prep-deep-dive/)
- [Nick's Method — Keyless Execution Explained](https://medium.com/patronum-labs/nicks-method-ethereum-keyless-execution-168a6659479c)
- [EIP-7702 Specification](https://eips.ethereum.org/EIPS/eip-7702)
- [ERC-7851 — EOA Deactivation (related proposal)](https://eips.ethereum.org/EIPS/eip-7851)
