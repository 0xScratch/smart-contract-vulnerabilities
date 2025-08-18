# Incorrect `gscAllowance` Accounting & ERC20 Allowance Overwrite Risk in ArcadeTreasury

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2023-07-arcade-findings/issues/85) / [One Bug Per Day](https://www.onebugperday.com/v1/303)
* **Affected Contract**: [ArcadeTreasury.sol](https://github.com/code-423n4/2023-07-arcade/blob/f8ac4e7c4fdea559b73d9dd5606f618d4e6c73cd/contracts/ArcadeTreasury.sol)
* **Vulnerability Type**: Business Logic / Accounting Error

## Original Bug Description

>## Lines of code
>
>[https://github.com/code-423n4/2023-07-arcade/blob/f8ac4e7c4fdea559b73d9dd5606f618d4e6c73cd/contracts/ArcadeTreasury.sol#L391](https://github.com/code-423n4/2023-07-arcade/blob/f8ac4e7c4fdea559b73d9dd5606f618d4e6c73cd/contracts/ArcadeTreasury.sol#L391)
>
>## Vulnerability details
>
>## Impact
>
>direct use of `IERC20(token).approve(spender, amount);` causes the same `spender` allowances to be overridden by each other
>
>## Proof of Concept
>
>In the `gscApprove()` method it is possible to give `spender` a certain allowance
>
>The code is as follows:
>
>```solidity
>    function gscApprove(
>        address token,
>        address spender,
>        uint256 amount
>    ) external onlyRole(GSC_CORE_VOTING_ROLE) nonReentrant {
>        if (spender == address(0)) revert T_ZeroAddress("spender");
>        if (amount == 0) revert T_ZeroAmount();
>
>        // Will underflow if amount is greater than remaining allowance
>@>      gscAllowance[token] -= amount;
>
>        _approve(token, spender, amount, spendThresholds[token].small);
>    } 
>
>    function _approve(address token, address spender, uint256 amount, uint256 limit) internal {
>        // check that after processing this we will not have spent more than the block limit
>        uint256 spentThisBlock = blockExpenditure[block.number];
>        if (amount + spentThisBlock > limit) revert T_BlockSpendLimit();
>        blockExpenditure[block.number] = amount + spentThisBlock;
>
>        // approve tokens
>@>      IERC20(token).approve(spender, amount);
>
>        emit TreasuryApproval(token, spender, amount);
>    }    
>```
>
>From the above code we can see that when executed `gscApprove` consumes `gscAllowance[]` and ultimately uses `IERC20(token).approve();` to give the `spender` allowance
>
>Since the direct use is `IERC20.approve(spender, amount)`, the amount of the allowance is overwritten, whichever comes last
>
>In the other methods `approveSmallSpend`, `approveMediumSpend`, `approveLargeSpend` also use `IERC20(token).approve();`, which causes them to override each other if targeting the same spender.
>
>Even if there is a malicious `GSC_CORE_VOTING_ROLE`, it is possible to execute `gscApprove(amount=1 wei)` after `approveLargeSpend()` to reset to an allowance of only `1 wei`.
>
>The recommendation is to use accumulation to avoid, intentionally or unintentionally, overwriting each other
>
>## Recommended Mitigation Steps
>
>```solidity
>    function _approve(address token, address spender, uint256 amount, uint256 limit) internal {
>        // check that after processing this we will not have spent more than the block limit
>        uint256 spentThisBlock = blockExpenditure[block.number];
>        if (amount + spentThisBlock > limit) revert T_BlockSpendLimit();
>        blockExpenditure[block.number] = amount + spentThisBlock;
>
>        // approve tokens
>-      IERC20(token).approve(spender, amount);
>+      uint256 old = IERC20(token).allowance(address(this),spender);
>+      IERC20(token).approve(spender,  old + amount);
>
>        emit TreasuryApproval(token, spender, amount);
>    } 
>```
>
>## Assessed type
>
>Context

## Summary

Arcade's **GSC (Governance Smart Contract) allowance system** tracks how much spending power a given proposal has. Internally, this is done using the `gscAllowance` mapping.

The problem is two-fold:

1. **Internal Accounting Bug** - decreasing an allowance can underflow the `gscAllowance` mapping, preventing allowances from being reduced properly. This can create a **DoS risk** where proposals cannot adjust their limits downward.
2. **External ERC20 Allowance Overwrite** - because ERC20 `approve()` allows overwriting approvals rather than only increasing/decreasing safely, it can cause mismatches between **Arcade's internal gscAllowance tracking** and the actual **ERC20 allowance state**.

Together, these lead to **unexpected behavior in allowance management**, weakening governance control and possibly locking or mismanaging treasury funds.

## A Better Explanation (With Simplified Example)

### Intended Behavior

* Governance should be able to set an allowance for a proposal (e.g., 100 USDC).
* If later reduced (e.g., down to 40 USDC), the system should safely reflect that.
* Internal `gscAllowance` (Arcade's bookkeeping) and the external ERC20 `allowance` (actual token approval) must always remain in sync.

### What Actually Happens (Bug)

1. **Underflow Risk in `gscAllowance`**

   * When decreasing an allowance, the subtraction can underflow.
   * Example: `gscAllowance[proposal] = 50`. If governance tries to reduce by 60, it underflows, blocking further adjustments.

2. **ERC20 Allowance Overwrite Quirk**

   * Standard ERC20 tokens let you call `approve(spender, newAmount)` which **replaces** the previous allowance.
   * If Arcade assumes allowances are always additive/subtractive, the overwrite can desync the actual ERC20 allowance from Arcade's internal `gscAllowance` records.

### Why This Matters

* **Governance Inconsistency** - governance may think a proposal has a certain limit when the ERC20 actually enforces something else.
* **DoS Risk** - allowance reductions may fail due to underflow, meaning governance cannot effectively limit or revoke access.
* **Unexpected Spending Outcomes** - external ERC20 overwrite behavior can be exploited to bypass internal checks or cause proposals to behave unpredictably.

## Vulnerable Code Reference

The issue centers around `gscAllowance` bookkeeping inside `ArcadeTreasury.sol` and how it interacts with ERC20 `approve()`.

**Illustrative snippet (simplified):**

```solidity
mapping(address => uint256) public gscAllowance;

function decreaseGscAllowance(address proposal, uint256 amount) external {
    gscAllowance[proposal] -= amount;  // ❌ risk of underflow
    token.approve(proposal, gscAllowance[proposal]); 
}
```

* If `amount > gscAllowance[proposal]`, this underflows.
* If ERC20 `approve` overwrites allowances unexpectedly, `gscAllowance` and ERC20 allowance diverge.

## Recommended Mitigation

### Option A — Increase-only semantics (recommended, simple)

Make all approval paths increase allowances (not set). Use OZ's `SafeERC20` (already imported):

```solidity
// using SafeERC20 for IERC20;

function _approve(address token, address spender, uint256 amount, uint256 limit) internal {
    uint256 spentThisBlock = blockExpenditure[block.number];
    if (amount + spentThisBlock > limit) revert T_BlockSpendLimit();
    blockExpenditure[block.number] = amount + spentThisBlock;

    IERC20(token).safeIncreaseAllowance(spender, amount); // additive, safer
    emit TreasuryApproval(token, spender, amount);
}
```

**Notes**:

* `safeIncreaseAllowance` handles non-standard return values and is widely compatible.
* If your OZ version supports it and you need exact-set behavior somewhere, use `forceApprove` (resets to 0 then sets new), e.g.:

  ```solidity
  IERC20(token).forceApprove(spender, newAmount);
  ```

  Otherwise, do the two-step: `approve(spender, 0)` then `approve(spender, newAmount)`.

### Option B — "Never lower" guard (keep exact numbers, but forbid shrinking)

Keep set-style approvals, but refuse to set a value below current unless a special, high-quorum function is used:

```solidity
function _approve(address token, address spender, uint256 amount, uint256 limit) internal {
    uint256 spentThisBlock = blockExpenditure[block.number];
    if (amount + spentThisBlock > limit) revert T_BlockSpendLimit();
    blockExpenditure[block.number] = amount + spentThisBlock;

    uint256 current = IERC20(token).allowance(address(this), spender);
    // Prevent accidental/malicious shrinking
    if (amount < current) revert T_InvalidAllowance(amount, current);

    // Use forceApprove (OZ >= 4.9) or zero-then-set pattern
    IERC20(token).approve(spender, 0);
    IERC20(token).approve(spender, amount);

    emit TreasuryApproval(token, spender, amount);
}
```

Then add a separate `decreaseAllowance(...)` function with a higher quorum (e.g., large) to intentionally reduce allowances when truly needed.

### Option C — Split responsibilities

* Let Core Voting paths set exact allowances (with zero-then-set or forceApprove).
* Let GSC path only increase (additive) up to its own allowance (never decrease).
* This prevents GSC from shrinking big governance approvals while still allowing them to "top up" small amounts.

## Pattern Recognition Notes

This issue reflects two recurring vulnerability patterns:

1. **Internal vs External State Divergence**

   * When protocols maintain **internal accounting (like `gscAllowance`)** alongside **external token state (ERC20 allowances, balances, etc.)**, they must always be reconciled.
   * If one diverges, governance/security assumptions break.

2. **ERC20 Approval Quirks**

   * Standard `approve(spender, amount)` **overwrites** the allowance, which can cause desyncs when the protocol assumes additive/subtractive semantics.
   * Safer ERC20s implement `increaseAllowance` and `decreaseAllowance`.

3. **Underflow/Overflow Risks in Accounting**

   * Always guard against subtraction underflow when decreasing tracked values.
   * Even if Solidity 0.8+ reverts on underflow, the **business logic failure (inability to decrease)** still matters.

4. **Governance-Sensitive Logic**

   * Allowance systems tied to governance must be **extra strict**, since they control treasury spending.
   * Bugs here can lead to **DoS on governance actions** or **fund mismanagement**.

5. **How to Spot Similar Bugs**

   * Look for mappings that track allowances/limits.
   * Compare them to how the external token/asset enforces its own state.
   * Check subtraction logic for underflows or missed safety checks.
   * Review any place where `approve()` is used instead of safer patterns.
