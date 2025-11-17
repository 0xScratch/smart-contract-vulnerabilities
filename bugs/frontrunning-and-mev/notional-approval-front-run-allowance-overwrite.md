# Approval Front-Running via Allowance Overwrite in `NoteERC20`

* **Severity**: Medium
* **Source**: [OpenZeppelin Audit - Notional Finance (M-01)](https://www.openzeppelin.com/news/notional-v2-audit-governance-contracts)
* **Affected Contract**: [NoteERC20.sol](https://github.com/notional-finance/contracts-v2/blob/c37c89c9729b830637558a09b6f22fc6a735da64/contracts/external/governance/NoteERC20.sol)
* **Vulnerability Type**: Allowance Manipulation / Race Condition / Front-Running

## Summary

`NoteERC20` implements a standard ERC-20 style `approve()` function that **overwrites** the spender's allowance with a new value:

```solidity
allowances[msg.sender][spender] = amount;
```

This pattern is a **well-known ERC-20 race condition**: if a user changes an allowance from value **A → B**, a malicious spender can **front-run** the user's allowance change and spend the **old allowance (A)** before the update is mined. After the update lands, the attacker still holds the **new allowance (B)** and can spend that too — resulting in a **double-spend** of allowances.

The NOTE token acknowledges this issue in comments but does **not** mitigate it. The recommended mitigation is to use `increaseAllowance` / `decreaseAllowance` patterns, which eliminate the front-run window by changing allowances **incrementally** instead of overwriting them.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. **Owner sets allowance**
   Bob is approved to spend 100 NOTE from Alice's account:

   ```solidity
   approve(bob, 100)
   ```

2. **Owner later changes allowance**
   Alice wants to reduce Bob's allowance to 10:

   ```solidity
   approve(bob, 10)
   ```

   This should simply replace Bob's allowance with a new lower value.

### What Actually Happens (Bug)

Because `approve()` **overwrites** the allowance, an attacker can exploit the race condition:

* Alice submits `approve(bob, 10)`

* Transaction enters mempool

* Bob sees it

* Bob sends:

  ```solidity
  transferFrom(alice → bob, 100)  // uses old allowance
  ```

  and sets a **higher gas price** to get mined **before** Alice's transaction.

* Alice's `approve(10)` confirms after Bob's transfer
  → new allowance = 10

* Bob now also spends the new allowance:

  ```solidity
  transferFrom(alice → bob, 10)
  ```

### Why This Matters

* Bob successfully drains **old allowance (100) + new allowance (10)**.
* Alice intended to *reduce* Bob's allowance but instead loses **even more tokens**.
* The issue is not theoretical — it has been exploited historically on ERC-20 tokens.
* Any NOTE holder interacting with a malicious or compromised spender is at risk.

### Concrete Walkthrough (Alice & Bob)

* **Initial State**: Alice approved Bob for **100 NOTE**.

* **Alice wants to lower allowance**: submits:

  ```solidity
  approve(bob, 10)
  ```

* **Bob attacks**:

  1. Monitors mempool

  2. Sees Alice's allowance-change transaction

  3. Sends:

     ```solidity
     transferFrom(alice → bob, 100)  // old allowance
     ```

     with high gas.

     → This executes FIRST.

  4. Alice's transaction then overwrites allowance to **10**.

  5. Bob sends:

     ```solidity
     transferFrom(alice → bob, 10)   // new allowance
     ```

**Result:** Bob drains **110 NOTE**, which is more than Alice ever intended to allow.

> **Analogy:**
> You try to reduce someone's withdrawal limit from ₹1000 to ₹100.
> They see you making the change, rush to the bank, withdraw the full ₹1000 first — **then** still withdraw the new ₹100.

## Vulnerable Code Reference

### 1) Direct overwrite of allowance

```solidity
function approve(address spender, uint256 rawAmount) external returns (bool) {
    uint96 amount = _safe96(rawAmount, ...);
    allowances[msg.sender][spender] = amount; // overwritten directly
    emit Approval(msg.sender, spender, amount);
    return true;
}
```

No checks, no incremental updates, no allowance-reset requirement.

### 2) No requirement to set allowance to zero first

OpenZeppelin recommends a pattern like:

```solidity
require(allowances[msg.sender][spender] == 0, "Must reset allowance to 0 first");
```

NOTE token does not enforce this.

### 3) `transferFrom` consumes allowances as usual

Meaning a malicious spender can immediately exploit any old allowance before it changes.

```solidity
uint96 newAllowance = spenderAllowance - amount; // if old allowance available
```

## Recommended Mitigation

### 1. Implement `increaseAllowance` and `decreaseAllowance` (OpenZeppelin recommendation)

These avoid overwriting and instead adjust the allowance relative to its current value:

```solidity
function increaseAllowance(address spender, uint256 addedValue) external;
function decreaseAllowance(address spender, uint256 subtractedValue) external;
```

### 2. OR require zeroing allowance before changing it

```solidity
require(
    rawAmount == 0 || allowances[msg.sender][spender] == 0,
    "Must reset allowance to zero before updating"
);
```

This removes the front-run window.

### 3. Educate users via documentation

NOTE token users relying on third-party contracts should be warned:

* Do not change allowance from value A → B directly
* Always set allowance to 0 first
* Or allow frontends to use the proper incremental pattern

### 4. Optional defensive update

Emit warnings or events when allowances are overwritten, helping auditors detect abnormal behavior.

## Pattern Recognition Notes

* **Race Conditions in Approval**
  ERC-20's `approve()` is widely known to be dangerous when overwriting an existing allowance. The NOTE token inherits this pattern unchanged.

* **Mempool Monitoring Attacks**
  Any allowance change entered into the mempool can be observed and exploited before miners include it.

* **Overwriting vs. Incremental Updates**
  Overwriting a value creates a window for double-spending. Incremental updates (increase/decrease) avoid this.

* **Spender-As-Attacker Model**
  Vulnerability assumes spender might act maliciously — a valid model if interacting with DEXs, routers, or compromised addresses.

* **Ethereum Economics**
  Higher-gas transactions allow attackers to reorder execution — enabling this classic front-run pattern.

### Quick Recall (TL;DR)

* ERC-20 `approve()` overwrites allowances.
* A spender can front-run an allowance change and spend both old + new values.
* NOTE token inherits this unsafe behavior.
* Use `increaseAllowance` / `decreaseAllowance`, or enforce zero-first approval to fix.
