# Reorg Attack on Position Clone Creation via Predictable CREATE Address Derivation

* **Severity**: Medium  
* **Source**: [Code4rena](https://github.com/code-423n4/2023-04-frankencoin-findings/issues/155)
* **Affected Contract**: [`PositionFactory.sol`](https://github.com/code-423n4/2023-04-frankencoin/blob/1022cb106919fba963a89205d3b90bf62543f68f/contracts/PositionFactory.sol)
* **Vulnerability Type**: Address Predictability / Reorganization (Reorg) Attack / Front-Running / Potential Fund Theft

## Summary  

The `PositionFactory` contract deploys new Position clone contracts using the standard **CREATE** opcode. This makes the resulting clone address **fully predictable** based only on the factory's address and its current nonce.  

In case of a **blockchain reorganization (reorg)** — which can occur on Ethereum (though rare, ~once a year) and much more frequently/deeper on L2s/rollups — an attacker can front-run a user's position creation transaction. The attacker deploys a clone at the **same predictable address** on a temporary fork. If the user's transaction confirms after the reorg, they deploy to an address the attacker already interacted with (e.g., by sending funds or setting ownership). Funds or collateral sent to this "new" position can then be withdrawn or controlled by the attacker, leading to **theft of user funds**.

## A Better Explanation (With Simplified Example)  

### Intended Behavior  

1. User calls a function on `PositionFactory` (e.g. `createClone` or similar) to deploy a new **Position** clone contract that holds their collateral and mints ZCHF against it.  
2. The factory uses **CREATE** opcode → new address = keccak256(sender=factory, nonce=factory.nonce).  
3. User receives the new position address and proceeds to fund it with collateral.  
4. Everything works normally in a normal chain (no reorg).  

### What Actually Happens (Bug)  

* Address is **100% deterministic** from factory address + nonce.  
* Reorgs create temporary forks where nonce can differ briefly.  
* Attacker sees user's pending tx in mempool → calculates next address → deploys clone themselves on the fork (or sends funds to it).  
* Reorg resolves → user's tx executes → deploys to **same address** attacker targeted.  
* User's collateral/transfers go to an address attacker can interact with → **funds stolen**.  

### Why This Matters  

* Even rare on Ethereum mainnet, reorgs are common and deeper on many L2s (e.g., Polygon historical 100+ block reorgs, Optimism/Arbitrum occasional ones).  
* Frankencoin planned L2 deployments → risk increases significantly.  
* Users / frontends that pre-compute or trust position addresses before confirmation are at risk.  
* Leads to **direct loss of user collateral/funds** under realistic (though low-probability) conditions → classified as **Medium** in similar audits (e.g., RabbitHole, PoolTogether cases).  

### Concrete Walkthrough (Alice & Mallory)  

* **Setup**: Factory nonce = 146; next predictable clone address = `0xabc...123`.  
* **Mallory attack**: Mallory monitors mempool, sees Alice's pending `createClone` tx.  
  During a short reorg/fork:  
  * Mallory calls factory → creates clone at nonce=147 → owns `0xabc...123`.  
  * Mallory sends 1 wei ETH or tiny tokens to `0xabc...123` (or waits for future interactions).  
* **Reorg resolves** → real chain has nonce=146.  
* Alice's tx executes → factory nonce → 147, deploys clone at **same** `0xabc...123`.  
* Alice sends valuable collateral to her "new" position → Mallory withdraws/steals it (owns the clone from the attacker's view in history).  

> **Analogy**: Like booking a hotel room number based on a counter. During a temporary power outage (reorg), someone else quickly books the same room number. When power returns, you get assigned the same room — but the other person already has the key and can take your luggage.

## Vulnerable Code Reference  

**Deployment uses standard CREATE (no salt)**  

```solidity
// PositionFactory.sol ~L44 (approximate from original report)
function createClone(/* params */) external returns (address) {
    // ...
    Position clone = new Position();               // ← Uses CREATE opcode (nonce-based)
    // or: address clone = address(new Position(...));
    // No CREATE2 with salt including msg.sender or unique ID
    // ...
}
```

→ Address derived purely from factory address + current nonce (no user control).  

## Recommended Mitigation  

1. **Switch to CREATE2 with user-controlled salt** (primary fix)  

    ```solidity
    bytes32 salt = keccak256(abi.encode(msg.sender, collateralToken, /* other unique params */));
    address clone = Clones.cloneDeterministic(implementation, salt);  // or manual CREATE2
    // Now only the caller can reliably predict/create the address → reorg front-run impossible
    ```

2. Use OpenZeppelin's `Clones` library with `cloneDeterministic` for safety.  
3. **Additional hardening**:  
   * Avoid relying on pre-computed addresses in UI/frontend.  
   * Emit events with created address and require confirmation before funding.  
   * If multi-chain, use chain-specific salts.  
4. **Add tests**: Include reorg simulation tests (e.g., via Foundry cheatcodes) to verify no address collision possible.  

## Pattern Recognition Notes  

* **Nonce-based Address Predictability**: Standard CREATE makes addresses guessable → vulnerable to reorg + front-running.  
* **Reorg Front-Running Class**: Very common Medium finding since 2021 (seen in PoolTogether, RabbitHole, many factories).  
* **Lack of Sender-Bound Salt**: CREATE2 salt should include `msg.sender` or unique caller data to prevent impersonation.  
* **L2 Amplification**: Risk jumps from "rare" to "plausible" on rollups with frequent/deeper reorgs.  
* **Recoverability**: Hard to recover stolen funds post-exploit → preventive design is critical.  

## Quick Recall (TL;DR)  

* **Bug**: Position clones deployed with CREATE → address predictable from nonce → reorg allows attacker to claim same address.  
* **Impact**: User sends collateral to attacker-controlled clone → **theft of funds** (Medium severity).  
* **Fix**: Use **CREATE2** with salt including `msg.sender` + unique params; prevent predictable front-running.  
