# User-Controlled AMM Pool Mismatch Enables Theft of Protocol stETH Fees

* **Severity**: High
* **Source**: [Sherlock](https://github.com/sherlock-audit/2022-10-illuminate-judging/issues/47)
* **Affected Contract**: [Lender.sol](https://github.com/sherlock-audit/2022-10-illuminate/blob/main/src/Lender.sol#L572)
* **Vulnerability Type**: Business Logic Flaw / Input Validation / Asset-Pool Mismatch

## Summary

Several fixed-income DeFi protocols, including **Illuminate**, allow users to supply **both** the declared *underlying asset* and the *AMM pool or adapter* used for swapping — **without validating that they correspond to the same token**.

Because the AMM pool implicitly defines **which asset is actually spent**, an attacker can lie about the underlying token, supply a pool that swaps a **different asset**, and force the protocol to swap **its own stored assets (protocol fees)** instead of the user's deposit.

By additionally setting `minAmountOut = 0` and sandwiching the transaction, the attacker can extract **nearly all accumulated stETH protocol fees**.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. **User deposits an underlying token** (e.g., DAI).
2. Protocol:

   * Takes a small **fee**
   * Swaps the remaining amount on an AMM
   * Receives principal / yield tokens
3. Protocol mints corresponding tokens to the user.

> **Invariant (assumed)**:
> "The protocol only swaps the asset the user actually deposited."

### What Actually Happens (Bug)

* The protocol:

  * Trusts the user-supplied **underlying token address**
  * Trusts the user-supplied **AMM pool**
  * Does **not verify that the pool trades the declared underlying**
* The AMM router:

  * Does **not receive the token address explicitly**
  * Infers the token being swapped **from the pool**
* Therefore:

  * **The pool — not `underlying` — determines which asset is spent**

This allows an attacker to **deposit one token** while forcing the protocol to **swap a different token it already holds**.

### Concrete Walkthrough (Alice & Mallory)

#### Setup

* Protocol has accumulated **100 stETH** as protocol fees.
* Protocol has approved an external AMM router to spend stETH.
* Mallory controls a large amount of **DAI**.

#### Step 1: Mallory calls `lend()`

She supplies the following **lying inputs**:

| Parameter          | Value                |
| ------------------ | -------------------- |
| `underlying (u)`   | DAI                  |
| `amount (a)`       | 100 DAI              |
| `pool`             | **stETH AMM pool** ❌ |
| `minAmountOut (r)` | `0` ❌                |

#### Step 2: Protocol processes deposit

* Pulls **100 DAI** from Mallory
* Records a small DAI fee
* Prepares to swap the remaining amount

So far, everything looks normal.

#### Step 3: Protocol performs the swap (critical failure)

```solidity
IAPWineRouter(x).swapExactAmountIn(
    pool,          // attacker-chosen stETH pool
    ...,
    lentAmount,
    r = 0,
    ...
);
```

What actually happens:

* The AMM pool expects **stETH**
* The protocol **pays with its own stETH fees**
* The user's DAI is **never swapped**

#### Step 4: Sandwich extraction

Because `minAmountOut = 0`:

1. Mallory front-runs to push stETH price unfavorably
2. Protocol swaps **100 stETH → ~0 output**
3. Mallory back-runs to restore price

#### Result

| Party        | Outcome                         |
| ------------ | ------------------------------- |
| Protocol     | Loses ~100 stETH (fees drained) |
| Mallory      | Gains ~100 stETH                |
| Mallory cost | Gas + temporary capital         |

> **Key Insight**:
> The stETH did not disappear — it was transferred to the attacker via manipulated AMM pricing.

### Why This Matters

* Fees accumulate naturally → attack becomes profitable over time
* No special permissions required
* No revert, no failed transaction
* Can be repeated until protocol fee balance is zero
* Looks like a "normal" user interaction

This is **theft**, not griefing.

## Vulnerable Code Reference

### 1) User-controlled asset declaration

```solidity
function lend(
    uint8 p,
    address u,     // underlying (user supplied)
    ...
    address x,     // router (user supplied)
    address pool   // pool (user supplied)
)
```

No validation ensures `pool` actually trades `u`.

### 2) Swap call derives asset from pool, not from `u`

```solidity
IAPWineRouter(x).swapExactAmountIn(
    pool,
    apwinePairPath(),   // does not encode token addresses
    apwineTokenPath(),  // only relative indices (0 / 1)
    lent,
    r,                  // minAmountOut = 0 allowed
    address(this),
    d,
    address(0)
);
```

The router infers **which token is spent** from the pool.

### 3) Protocol has pre-existing approval & balance

```text
Protocol already:
• Holds stETH as fees
• Has approved router to spend stETH
```

This makes unintended spending possible.

## Recommended Mitigation

1. **Strict asset-pool validation (primary fix)**
   Before swapping:

   * Read pool tokens
   * Enforce `tokenIn == underlying`

2. **Bind asset to swap call**
   Use swap interfaces that:

   * Explicitly specify input token
   * Reject ambiguous pool-only execution

3. **Enforce sane slippage bounds**

```solidity
require(minAmountOut > 0, "Invalid slippage");
```

4. **Reduce implicit approvals**

   * Approve exact amounts
   * Revoke approvals after use

5. **Add invariant tests**

   * "Protocol never swaps tokens it did not receive in this call"

## Pattern Recognition Notes

* **Asset-Execution Mismatch**: If users control *what you claim to swap* and *how you swap*, they control which asset you pay with.
* **Implicit Asset Resolution**: AMM pools define token flows — never trust external context to infer assets safely.
* **Fee Vault Exposure**: Accumulated protocol fees are high-value targets when execution paths are user-controlled.
* **Slippage as an Attack Enabler**: Allowing `minAmountOut = 0` converts logic bugs into extractable value.
* **Cross-Protocol Repetition**: The same design flaw appeared in multiple independent protocols — a strong signal of a systemic pattern.

## Quick Recall (TL;DR)

* **Bug**: User supplies a pool that swaps a different asset than the declared underlying.
* **Impact**: Protocol swaps **its own stETH fees** instead of user funds.
* **Exploitability**: Guaranteed with `minAmountOut = 0` + sandwich.
* **Fix**: Enforce asset-pool consistency and bind swap inputs explicitly.
