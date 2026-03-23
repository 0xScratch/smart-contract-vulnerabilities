# Admin-Controlled Token Address Allows Infinite Mint → Bridge Drain

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2022-02-skale-findings/issues/35)
* **Affected Contract**: [TokenManagerEth.sol](https://github.com/skalenetwork/ima-c4-audit/blob/main/contracts/schain/TokenManagers/TokenManagerEth.sol#L45-L49)
* **Vulnerability Type**: Access Control / Trust Assumption Violation / Bridge Integrity

## Summary

`TokenManagerEth` allows the `DEFAULT_ADMIN_ROLE` to arbitrarily update the `ethErc20` token address via `setEthErc20Address()`. This breaks a critical bridge invariant: **the authenticity of the token representation on the SChain**.

An admin can replace the legitimate token with a malicious contract that allows **infinite minting**, then call `exitToMain(amount)` to withdraw real funds from the Ethereum-side `DepositBox`. Since the bridge trusts SChain state, this results in a **complete drain of funds**.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. **Deposit (Ethereum → SKALE)**

   * User deposits tokens into `DepositBox` on Ethereum
   * Equivalent tokens are minted on SKALE

2. **Withdraw (SKALE → Ethereum)**

   * User burns tokens on SKALE
   * Bridge verifies and releases real tokens from Ethereum

👉 Assumption:

> Tokens on SKALE correctly represent real locked funds on Ethereum

### What Actually Happens (Bug)

* During initialization, `msg.sender` is granted `DEFAULT_ADMIN_ROLE`
* Admin can call:

```solidity
function setEthErc20Address(IEthErc20 newEthErc20Address)
```

* This allows replacing the trusted token with **any arbitrary contract**

### Why This Is Dangerous

The bridge logic assumes:

> "If tokens are burned on SKALE, they must be legitimate"

But the admin can:

* Replace token with malicious contract
* Mint unlimited tokens
* Burn them via `exitToMain()`

👉 Ethereum side cannot distinguish fake vs real

### Concrete Walkthrough (Alice & Mallory)

* **Setup**:

  * DepositBox on Ethereum holds **100 ETH worth of tokens**
  * SKALE mirrors token supply

* **Mallory (malicious admin)**:

  1. Deploys fake ERC20 with infinite mint
  2. Calls:

     ```solidity
     setEthErc20Address(maliciousToken);
     ```
  3. Mints **100 tokens** (backed by nothing)
  4. Calls:

     ```solidity
     exitToMain(100);
     ```

* **Bridge behavior**:

  * Sees burn event on SKALE
  * Assumes legitimacy
  * Releases **100 ETH from DepositBox**

* **Result**:

> Mallory drains entire bridge balance using fake tokens

> **Analogy**:
> A bank vault releases cash if you return deposit receipts. The admin secretly changes what counts as a "valid receipt" and prints unlimited fake ones.

## Vulnerable Code Reference

### 1) Admin role assigned during initialization

```solidity
_setupRole(DEFAULT_ADMIN_ROLE, msg.sender);
```

👉 Single entity gets full control

### 2) Arbitrary token replacement

```solidity
function setEthErc20Address(IEthErc20 newEthErc20Address) external override {
    require(hasRole(DEFAULT_ADMIN_ROLE, msg.sender), "Not authorized caller");
    require(ethErc20 != newEthErc20Address, "Must be new address");
    ethErc20 = newEthErc20Address;
}
```

👉 No validation → any contract allowed

### 3) Bridge exit relies on token correctness

```solidity
exitToMain(amount);
```

👉 Assumes:

* Tokens burned are legitimate
* Supply is correctly backed

❌ Broken once token is replaced

## Recommended Mitigation

### 1. **Remove mutable token address (Primary Fix)**

```solidity
// Remove entirely
setEthErc20Address(...)
```

👉 Token mapping should be immutable after initialization

### 2. **Enforce strong trust boundaries**

* Token address must be:

  * Immutable
  * Or set once during deployment

### 3. **If flexibility is required**

Use:

* Timelock + governance
* Multi-sig approval
* Strict validation (e.g., known token registry)

### 4. **Bridge-level invariant enforcement**

* Ensure:

  * Token supply on SKALE cannot exceed deposits
  * Cross-chain accounting checks exist

## Pattern Recognition Notes

* **Mutable Critical Dependency**
  Allowing admin to modify core system dependencies (token, oracle, validator) breaks trust assumptions.

* **Bridge Trust Amplification**
  Bridges already rely on trust between chains. Adding admin-controlled parameters introduces **compounded trust risk**.

* **Fake Asset Injection**
  If attacker can define what represents value, they can:

  * Mint fake assets
  * Redeem real ones

* **"Trusted Role" Fallacy**
  Even if admin is trusted:

  * Keys can be compromised
  * Governance can be attacked
  * Insider threats exist

👉 Critical invariants must not depend on trust

* **Invariant Violation Pattern**

> "Asset representation ≠ asset backing"

Whenever:

* Representation layer (token) is mutable
* Backing layer (funds) is fixed

👉 High risk of drain

## Quick Recall (TL;DR)

* **Bug**: Admin can replace token contract
* **Impact**: Mint fake tokens → withdraw real funds → full drain
* **Root Cause**: Critical bridge invariant controlled by admin
* **Fix**: Make token address immutable / non-upgradable
