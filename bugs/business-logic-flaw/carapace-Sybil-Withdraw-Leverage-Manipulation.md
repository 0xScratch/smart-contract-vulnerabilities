# Sybil Withdrawal Requests Enable Leverage Manipulation via Flashloans

* **Severity**: High
* **Source**: [Sherlock](https://github.com/sherlock-audit/2023-02-carapace-judging/issues/116)
* **Affected Contract**: [ProtectionPool.sol](https://github.com/sherlock-audit/2023-02-carapace/blob/main/contracts/core/pool/ProtectionPool.sol#L992-L995)
* **Vulnerability Type**: Business Logic Flaw / Economic Exploit / Sybil Attack

## Summary

The protocol allows users to **request withdrawals based on their current SToken balance**, but **does not lock or commit those tokens** after the request.

This enables an attacker to:

* Reuse the **same STokens across multiple addresses**
* Create **multiple withdrawal requests exceeding actual deposits**
* Artificially inflate **withdrawable liquidity**

Although this does not directly drain funds, it enables a powerful exploit:

> Using flashloans, attackers can **temporarily inject large liquidity**, manipulate the **leverage ratio**, obtain **cheaper protection premiums**, and withdraw funds in the same transaction.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. **Deposit**:
   User deposits USDC → receives equivalent **STokens**

2. **Request Withdrawal**:
   User requests withdrawal → must hold sufficient STokens

3. **Withdraw (after delay)**:
   User burns STokens → receives underlying funds

👉 Key assumption:
Each withdrawal request corresponds to **actual locked capital**

### What Actually Happens (Bug)

* Withdrawal request only checks:

  ```solidity
  balance >= requestedAmount
  ```
* BUT:

  * Tokens are **not locked**
  * Tokens can be **transferred after request**

### Exploit Flow (Step-by-Step)

#### Step 1 — Initial Deposit

Attacker deposits:

```text
10,000 USDC → 10,000 STokens
```

#### Step 2 — First Withdrawal Request

```text
Address A → request withdraw (10k)
```

#### Step 3 — Reuse Tokens

```text
Transfer 10k STokens → Address B
Address B → request withdraw (10k)
```

#### Step 4 — Repeat (Sybil Attack)

Attacker repeats across multiple addresses:

| Address | Withdrawal Request |
| ------- | ------------------ |
| A       | 10k                |
| B       | 10k                |
| C       | 10k                |
| ...     | ...                |

Now:

```yaml
Total requested withdrawals = 100k
Actual deposit = 10k
```

⚠️ Protocol thinks **100k is withdrawable**, but only 10k exists.

---

### Step 5 — Flashloan Exploit

After withdrawal window opens:

1. Take flashloan:

   ```text
   100k USDC
   ```

2. Deposit via multiple addresses:

   ```yaml
   Inflate pool liquidity
   ```

3. Result:

   ```yaml
   Leverage ratio increases → premiums become cheaper
   ```

4. Buy protection at manipulated price

5. Withdraw all deposits immediately (allowed due to pre-approved withdrawals)

6. Repay flashloan

### Why This Matters

* No real capital required
* Temporary liquidity manipulation
* Direct economic impact on protocol

### Concrete Walkthrough (Attacker Scenario)

* Attacker deposits **10k**
* Creates **100k withdrawal rights** using Sybil addresses
* Waits for withdrawal cycle
* Executes:

```yaml
Flashloan → Deposit → Manipulate → Buy cheap protection → Withdraw → Repay
```

> **Analogy**:
> Imagine a system where you can reserve withdrawals without locking money.
> You show the same ₹10,000 balance to 10 different counters and reserve ₹1,00,000 total withdrawals.
> Later, you temporarily borrow money to satisfy all reservations, exploit pricing, and leave.

## Vulnerable Code Reference

**Withdrawal request only checks balance, not locking tokens**

```solidity
// Pseudocode from ProtectionPool.sol (~L992)
require(sToken.balanceOf(msg.sender) >= amount);
```

❌ Missing:

```solidity
// No locking / reservation of tokens
```

## Root Cause

**State inconsistency between:**

* Withdrawal requests (virtual claim)
* Actual token ownership (transferable)

### Core Issue

> Same asset is counted multiple times because it is **not reserved after commitment**

## Impact

### 1. Premium Manipulation

* Artificially high liquidity
* Lower premiums for attackers
* Loss for protection sellers

### 2. Pool Drain (Indirect)

When combined with other vulnerabilities:

* Attacker can **overbuy protection cheaply**
* If lending pool defaults → **massive payout**
* Potential full drain

### 3. Protocol DoS (Economic)

* Leverage ratio becomes unreliable
* Pricing mechanism breaks

## Recommended Mitigation

### 1. Lock STokens on Withdrawal Request (Primary Fix)

```solidity
// On requestWithdraw
lockedBalance[msg.sender] += amount;
```

OR

```solidity
// Transfer tokens to escrow
sToken.transferFrom(msg.sender, address(this), amount);
```

### 2. Track Reserved vs Available Balance

```solidity
availableBalance = totalBalance - lockedBalance;
```

Ensure:

```solidity
availableBalance >= newRequest
```

### 3. Prevent Token Reuse Across Addresses

* Use **non-transferable (or partially locked) tokens**
* Or enforce **cooldown restrictions**

### 4. Add Invariants & Tests

* Total withdrawal requests ≤ total supply
* Tokens used in withdrawal must be **non-transferable until executed**

## Pattern Recognition Notes

### 1. **Commit Without Lock**

* User commits to an action (withdraw)
* But asset remains usable → **double counting**

### 2. **Sybil Amplification**

* Single asset reused across multiple identities
* Turns small capital into large influence

### 3. **Flashloan Amplification**

* Temporary liquidity exploits **time-based assumptions**
* Especially dangerous in:

  * Pricing models
  * Ratios (like leverage)

### 4. **Economic vs Technical Bugs**

* No revert / overflow / reentrancy
* Pure **logic flaw in system design**

### 5. **State Desynchronization**

* Protocol assumes:

  ```yaml
  withdrawal request = locked funds
  ```
* Reality:

  ```yaml
  withdrawal request ≠ locked funds
  ```

## Quick Recall (TL;DR)

* **Bug**: Withdrawal request doesn't lock STokens
* **Exploit**: Reuse same tokens across addresses (Sybil)
* **Result**: Fake withdrawable liquidity
* **Attack**: Flashloan → inflate liquidity → cheaper premiums → withdraw
* **Fix**: Lock or escrow tokens at withdrawal request
