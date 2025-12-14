# Deterministic Pool Address Hijack via `tx.origin` on Blast (CREATE2 Collision Under Reorg)

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2024-03-abracadabra-money-findings/issues/211)
* **Affected Contract**: [Factory.sol](https://github.com/code-423n4/2024-03-abracadabra-money/blob/1f4693fdbf33e9ad28132643e2d6f7635834c6c6/src/mimswap/periphery/Factory.sol#L81-L90)
* **Vulnerability Type**: Identity Spoofing / CREATE2 Collision / Misuse of `tx.origin` / Reorg-Based Address Hijack

## Summary

The Abracadabra MagicLP Factory creates new AMM pools using **CREATE2** with a salt derived from:

```solidity
salt = keccak256(creator = tx.origin, implementation, baseToken, quoteToken, fee, i, k)
```

The design **assumes** `tx.origin` is the user's unique address, ensuring each CREATE2 deployment maps to a predictable, user-specific pool address.

However, on the **Blast** chain (an optimistic rollup), most transactions share a **sequencer-level `tx.origin`**, not the actual user. This causes multiple unrelated users — including attackers — to produce the **same salt** for identical pool parameters.

During a **block reorg**, a legitimate user's CREATE2 deployment can be dropped from the chain. An attacker can then replay the same parameters, causing their transaction to be mined first. Since Blast reuses the same `tx.origin`, the attacker's call produces the **same deterministic address**. The attacker becomes the deployer, gains full control of the MagicLP instance, and can steal any funds the victim sends to the predicted address.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. **User computes** the MagicLP pool's deterministic address using:

    ```solidity
    predictDeterministicAddress(user, base, quote, fee, i, k)
    ```

    where `creator = tx.origin ≈ user`.

2. **User calls `create()`**, Factory derives:

    ```solidity
    salt = hash(tx.origin, implementation, parameters)
    ```

    and deploys the MagicLP pool at the expected deterministic address.

3. **User deposits liquidity** into the newly created pool.

### What Actually Happens (Bug)

1. On the Blast network

    * Many transactions share a **fixed sequencer-origin address** (e.g., `0x4b16…`), not the true user's EOA.
    * Thus, unrelated users produce **identical `tx.origin`** values.

2. The CREATE2 salt now becomes

    ```solidity
    salt = hash(0xSequencerOrigin, implementation, parameters)
    ```

    making it **identical for all users** deploying the same pool configuration.

3. During a **reorg**, the victim's deployment can be dropped

    * The attacker sees the victim's intended pool parameters.
    * They submit their own transaction first.
    * CREATE2 deploys the MagicLP pool **at the SAME address**.
    * Attacker becomes the owner.

**Result:**  
Any funds the victim later deposits into the predicted pool address are fully controlled by the attacker.

### Why This Matters

1. CREATE2 relies on **salt uniqueness** to avoid collisions.

2. The Factory incorrectly relies on `tx.origin`, which is:

    * Not stable on optimistic rollups
    * Not user-unique
    * Fully controlled by chain infrastructure (sequencer)

3. This allows a realistic **address hijack attack** during:

    * Reorgs  
    * Transaction replacement  
    * Sequencer reordering  
    * Victim transaction drops

Because attackers can front-run reorg replacements using the **same salt**, they can steal all assets deposited to the pool.

### Concrete Walkthrough (Alice & Mallory)

#### Setup

Blast sequencer assigns `tx.origin = 0x4b16…` to all user transactions.

#### Alice's Steps

1. Alice predicts the pool address:

    ```solidity
    salt = hash(0x4b16…, USDC, WETH, 20, i, k)
    address = CREATE2(salt)
    ```

2. Alice submits the real `create()` transaction.
3. Chain includes it initially → pool deployed.

#### Reorg

Blast experiences a mild reorg:

* Alice's block is dropped.
* Her `create()` disappears.

#### Mallory attacks

1. Mallory copies Alice's parameters.
2. Mallory submits:

    ```solidity

    tx.origin = 0x4b16…     // same as Alice
    baseToken = USDC
    quoteToken = WETH
    fee = 20
    i, k same

    ```

3. Salt matches Alice’s:

    ```solidity
    salt(Mallory) = salt(Alice)
    ````

4. Mallory's transaction is mined after reorg.
5. Mallory becomes **the deployer** of the pool at Alice’s predicted address.

#### **Alice deposits**

Alice trusts the previously computed deterministic address and deposits liquidity.

→ Mallory now controls all liquidity.  
→ Full theft.

> **Analogy**: Alice reserves a locker using a code, but the gym changes the lock after a schedule reorganization.  
> Anyone who knows the code can now take the locker first.

## Vulnerable Code Reference

**1) Misuse of `tx.origin` in identity-sensitive salt computation**

```solidity
function create(...) external returns (address clone) {
 address creator = tx.origin; // ❌ Should not be used for identity

 bytes32 salt = _computeSalt(
     creator, baseToken_, quoteToken_, lpFeeRate_, i_, k_
 );

 clone = LibClone.cloneDeterministic(address(implementation), salt);
 IMagicLP(clone).init(...);
}
```

**2) Salt includes `creator = tx.origin` instead of `msg.sender`**

```solidity
function _computeSalt(
    address sender_,
    address baseToken_,
    address quoteToken_,
    uint256 lpFeeRate_,
    uint256 i_,
    uint256 k_
) internal view returns (bytes32) {
    return keccak256(abi.encodePacked(
        sender_, implementation, baseToken_, quoteToken_, lpFeeRate_, i_, k_
    ));
}
```

**3) No protection against `CREATE2` collisions under reorgs**

Factory assumes:

* `tx.origin` is unique per user
* No other caller can reuse it
* Reorg conditions do not cause address reuse

All incorrect on Blast.

## Recommended Mitigation

1. **Replace `tx.origin` with `msg.sender`** for salt derivation:

    ```solidity
    address creator = msg.sender;
    ```

2. **Optionally allow user-specified salt**
   Users may provide their own entropy to avoid chain-level interference.

3. **Document that deterministic addresses are unsafe on optimistic rollups** without including user-controlled entropy.

4. **Implement replay / duplicate protection**
   Check if a pool already exists for the parameters to prevent hijacks.

## Pattern Recognition Notes

* **`tx.origin` Misuse**
  Never use `tx.origin` for authentication, identity, or deterministic computation. Sequencer-level forwarding breaks assumptions.

* **CREATE2 Collision Risk**
  CREATE2 must use values controlled by the user or contract state — not external chain infrastructure.

* **Rollup Reorg Sensitivity**
  Optimistic rollups (Blast, Optimism, Arbitrum) frequently reorder or drop transactions. Deterministic deployments across reorgs are therefore vulnerable unless salts include user-specific entropy.

* **Predictability Attack Surface**
  Any deterministic address prediction flow becomes unsafe when attackers can replay the same salt post-reorg.

* **Environment-Aware Security**
  Deployment context matters. The same logic may be safe on Canto (Tendermint finality) but unsafe on Blast (probabilistic finality).

### Quick Recall (TL;DR)

* **Bug**: Factory uses `tx.origin` for CREATE2 salt, but on Blast many users share the same `tx.origin`.
* **Impact**: During reorgs, attackers can deploy the *same* MagicLP pool address before the victim, hijacking ownership → **fund theft**.
* **Fix**: Use `msg.sender` for salt; avoid relying on sequencer-controlled values for deterministic addresses; add collision protections.
