# Reentrancy-Based Reward Inflation via Collateral Ratio Manipulation in Angle Transmuter

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2023-06-angle-findings/issues/30)
* **Affected Contracts**:

  * [`Swapper.sol`](https://github.com/AngleProtocol/angle-transmuter/blob/9707ee4ed3d221e02dcfcd2ebaa4b4d38d280936/contracts/transmuter/facets/Swapper.sol)
  * [`Redeemer.sol`](https://github.com/AngleProtocol/angle-transmuter/blob/9707ee4ed3d221e02dcfcd2ebaa4b4d38d280936/contracts/transmuter/facets/Redeemer.sol)
  * [`SavingsVest.sol`](https://github.com/AngleProtocol/angle-transmuter/blob/9707ee4ed3d221e02dcfcd2ebaa4b4d38d280936/contracts/savings/SavingsVest.sol)
* **Vulnerability Type**: Reentrancy / Read-Only Reentrancy / Accounting Manipulation

## Summary

During collateral swaps, the `Swapper` contract transfers collateral tokens **before** minting new agTokens to the user.
If the transferred collateral token supports hooks (e.g., ERC777), a malicious user can **re-enter** the system mid-execution through this token transfer.

By calling `SavingsVest.accrue()` inside this reentrant hook, the attacker triggers a read-only query to `_transmuter.getCollateralRatio()` **before** the agTokens are officially minted.
This temporarily inflates the protocol's perceived collateralization ratio, causing `SavingsVest` to **mint excessive reward tokens**.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. **User swap (normal case)**

   * Alice swaps collateral (e.g., USDC) for agEUR.
   * The contract:

     1. Transfers the collateral from Alice to itself.
     2. Updates internal collateral tracking.
     3. Mints new agEUR tokens to Alice.
   * The collateral ratio is then:

     ```solidity
     collatRatio = totalCollateral / totalStablecoinsIssued
     ```

2. **Reward accrual**

   * When users call `SavingsVest.accrue()`, it queries `_transmuter.getCollateralRatio()` to determine reward rates.
   * If the system is 100% collateralized (`collatRatio = 1.0`), normal rewards are minted.

### What Actually Happens (Bug)

1. Inside `_swap()`, the function performs the token transfer **before** minting agTokens:

   ```solidity
   IERC20(tokenIn).safeTransferFrom(msg.sender, address(this), amountIn); // vulnerable call
   IAgToken(tokenOut).mint(to, amountOut);
   ```

2. If `tokenIn` is ERC777 (or any token with hooks), this transfer triggers a callback in Alice's contract.

3. In that callback, Alice calls:

   ```solidity
   SavingsVest.accrue();
   ```

4. `SavingsVest.accrue()` invokes:

   ```solidity
   (uint64 collatRatio, uint256 stablecoinsIssued) = _transmuter.getCollateralRatio();
   ```

5. Because `_swap()` hasn't yet minted the agTokens, `stablecoinsIssued` is **understated** →
   so:

   ```solidity
   collatRatio = totalCollateral / stablecoinsIssued
   ```

   returns an **artificially high** value.

6. If `collatRatio > 1.0`, the vesting logic interprets this as "overcollateralized" and **mints extra rewards** to Alice.

7. When `_swap()` resumes, it finally mints agTokens normally — leaving the system with **extra agTokens minted via the side-call**.

### Why This Matters

* Attackers can **mint unearned agTokens** by exploiting the sequence of operations and lack of reentrancy protection.
* The manipulation doesn't drain collateral directly, but it **inflates the stablecoin supply**, degrading peg stability and protocol solvency.
* Any contract or hook capable of re-entering mid-swap can trigger this, even without special privileges.

### Concrete Walkthrough (Alice's Attack Flow)

1. **Setup**:

   * `collatRatio = 1.0` (system fully collateralized).
   * Alice deploys a malicious ERC777 token with a custom `tokensReceived()` hook.

2. **Attack execution**:

   * Alice calls `Swapper.swap(tokenIn, agToken, amountIn)` using her malicious token.
   * `Swapper` calls:

     ```solidity
     tokenIn.safeTransferFrom(msg.sender, address(this), amountIn);
     ```

     → triggers `tokensReceived()` hook.

3. **Reentrant call from hook**:

   * Inside the hook, Alice calls `SavingsVest.accrue()`.
   * This queries `_transmuter.getCollateralRatio()` while `_swap()` hasn't minted agTokens yet.
   * The ratio appears > 1, leading to **over-minted rewards**.

4. **Back to swap**:

   * Execution returns to `_swap()`.
   * agTokens are finally minted, but now the total supply already includes the excess from step 3.

Result → Alice successfully inflates her agToken rewards.

## Vulnerable Code Reference

**1) Lack of `nonReentrant` guard in swap logic**

```solidity
if (permitData.length > 0) {
    PERMIT_2.functionCall(permitData);
} else if (collatInfo.isManaged > 0)
    IERC20(tokenIn).safeTransferFrom(
        msg.sender,
        LibManager.transferRecipient(collatInfo.managerData.config),
        amountIn
    );
else IERC20(tokenIn).safeTransferFrom(msg.sender, address(this), amountIn); //@audit reentrancy

IAgToken(tokenOut).mint(to, amountOut); // executed after transfer
```

### 2) Collateral ratio calculation vulnerable to timing mismatch

```solidity
collatRatio = uint64(
    totalCollateralization.mulDiv(BASE_9, stablecoinsIssued, Math.Rounding.Up)
);
```

### 3) Reward accrual depending on inflated ratio

```solidity
(uint64 collatRatio, uint256 stablecoinsIssued) = _transmuter.getCollateralRatio();
// Excess rewards minted if collatRatio > BASE_9 + BASE_6
```

## Recommended Mitigation

1. **Add `nonReentrant` modifier**
   Apply to:

   * `Swapper.swap()` and `Redeemer.redeem()` (state-changing functions).
   * `_transmuter.getCollateralRatio()` and any function callable during reward accrual.

2. **Reorder operations**
   Perform agToken minting **before** external token transfers if safe to do so, or move all sensitive accounting before potential reentrancy points.

3. **Use ERC20 over ERC777**
   Limit or disallow ERC777/other callback-enabled tokens as collateral to reduce attack surface.

4. **Add invariant checks**
   After every swap, ensure total collateral / stablecoin ratio remains within expected bounds.

## Pattern Recognition Notes

* **Reentrancy by Token Hooks**: External token standards like ERC777 allow callbacks that can disrupt internal flow if contracts don't use guards.
* **Read-Only Reentrancy**: Even view calls can be abused when contract state isn't finalized yet, producing misleading ratios or reward data.
* **Timing-Dependent Accounting**: Always finalize mint/burn updates before allowing cross-contract calls that depend on those values.
* **Cross-Module Coupling**: Shared state (e.g., collateral ratio) used across modules (Swapper ↔ SavingsVest) amplifies impact of inconsistent updates.
* **Defensive Order of Operations**: When state updates rely on external token transfers, either protect them with reentrancy guards or isolate sensitive logic from potential callbacks.

### Quick Recall (TL;DR)

* **Bug**: Reentrancy via ERC777 hooks lets attacker call `SavingsVest.accrue()` mid-swap.
* **Impact**: `getCollateralRatio()` reads outdated values → inflates rewards → **extra agTokens minted**.
* **Fix**: Add `nonReentrant` modifiers, reorder swap logic, and validate collateral token types.
