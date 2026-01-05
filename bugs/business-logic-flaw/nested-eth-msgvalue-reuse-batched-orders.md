# ETH Deposit Inflation via msg.value Reuse in Batched Orders

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2021-11-nested-findings/issues/226)
* **Affected Contract**: [`NestedFactory.sol`](https://github.com/code-423n4/2021-11-nested/blob/main/contracts/NestedFactory.sol)
* **Vulnerability Type**: Funds Theft / msg.value Reuse / Batching / Multicall Exploitation

## Summary

The `NestedFactory` contract allows users to batch multiple investment orders (e.g., deposits/swaps to build or add to crypto portfolios) in a single transaction. When ETH (native currency, represented as `address(0)`) is used as an input token, the internal function `_transferInputTokens` relies on `msg.value` (the ETH sent with the entire transaction) to account for the deposited amount.

Because `msg.value` is constant throughout the transaction (it doesn't decrease or get "consumed" per call), and because the contract processes multiple orders in a loop (or via multicall batches), an attacker can reuse the same `msg.value` across many orders. This tricks the contract into crediting the user with far more portfolio value than the actual ETH sent, allowing the attacker to later withdraw/sell tokens and drain real ETH from the contract's holdings.

This is a classic **msg.value reuse in batch** vulnerability, amplified by the contract inheriting from OpenZeppelin's `Multicall`.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. **Deposit / Create Portfolio**: User sends ETH (`msg.value`) + array of orders (e.g., "use ETH to buy Token A, then Token B").
2. For each order, `_transferInputTokens` handles the input:
   * If input is ETH → assumes `msg.value` covers the required amount.
   * Contract uses its updated ETH balance for swaps/operators.
3. Portfolio gets credited with the correct value → user can later sell/withdraw.

### What Actually Happens (Bug)

* `msg.value` is fixed per transaction — it persists across internal calls/loops and even across `multicall` sub-calls.
* In a single transaction with multiple ETH-input orders (or multiple batched `create()`/`addTokens()` calls via multicall), each order sees the **same** `msg.value` as the "deposited" amount.
* Contract credits user with multiplied value (e.g., 10 orders × 1 ETH credit = 10 ETH value) but only 1 ETH was actually sent.
* Attacker later withdraws the inflated amount → drains contract's real ETH holdings.

### Why This Matters

* A single malicious transaction can extract arbitrary amounts of ETH from the contract (limited only by its balance and gas).
* Affects core functionality: creating/copying portfolios, depositing ETH.
* No need for reentrancy — pure transaction batching exploit.
* Amplifies because `NestedFactory` inherits `Multicall`: even single-order functions can be batched with reused `msg.value`.

### Concrete Walkthrough (Alice & Mallory)

* **Setup**: Contract holds some ETH from legitimate users.
* **Mallory attack**:
  1. Mallory sends 1 ETH (`msg.value = 1 ETH`).
  2. Calls `create()` or `addTokens()` with an array of **10 orders**, each requiring 1 ETH input (or uses `multicall` to batch 10 separate calls).
  3. Each order's `_transferInputTokens` sees `msg.value = 1 ETH` → credits Mallory's portfolio with **10 ETH** worth of value.
  4. Mallory performs swaps/builds portfolio → then calls `sellTokensToWallet()` to withdraw as ETH.
* **Result**: Mallory withdraws ~10 ETH (minus fees) while only sending 1 ETH → **steals 9 ETH** from contract (and scales up with more orders).

> **Analogy**: A bank teller who gives you $100 cashback for every $100 deposit you claim — but you only show the same $100 bill multiple times in one visit (because the teller doesn't check the vault each time). The teller hands out $1000 but the vault only lost $100 → empty vault fast.

## Vulnerable Code Reference

**1) No per-order ETH consumption / reliance on persistent msg.value**  

```solidity
// _transferInputTokens(...) excerpt (approx line ~462 in 2021 snapshot)
if (address(_inputToken) == ETH) {
    // Uses msg.value directly — no deduction, no check that msg.value covers cumulative orders
    // Assumes balance increase == _inputTokenAmount (but balance only increases once per tx)
}
```

**2) Batched processing in entry points**  

```solidity
// create() / addTokensToPortfolio() ~L103 & ~L119
_submitInOrders(_orders);  // loops over orders → multiple _transferInputTokens calls in same tx

// Multicall inheritance (L20)
contract NestedFactory is Multicall { ... }  // allows batching any payable calls with same msg.value
```

**3) Withdraw paths also affected** (sellTokensToWallet / sellTokensToNft ~L152 & ~L172)  

```solidity
_submitOutOrders(...);  // can involve ETH inputs → same msg.value issue if batched
```

## Recommended Mitigation

1. **Primary Fix: Ban batching for ETH operations**  
   Reject arrays with multiple orders when any order uses ETH as input.  

   ```solidity
   // In functions that take Order[] calldata _orders
   bool hasETH = false;
   for (uint i = 0; i < _orders.length; i++) {
       if (_orders[i].inputToken == ETH) {
           require(!hasETH, "ETH operations cannot be batched");
           hasETH = true;
       }
   }
   // Or simpler: require only 1 order if any ETH involved
   ```

2. **For Multicall**: Add special handling  
   Track cumulative expected ETH in a per-tx storage variable, or disable ETH in multicall entirely (e.g., require msg.value == 0 in multicall if any subcall is payable with ETH).

3. **Alternative: Use explicit ETH pull per call**  
   Require separate `msg.value` checks or use WETH everywhere (but breaks native ETH UX).

4. **Defensive checks**  
   In `_transferInputTokens`:  

   ```solidity
   if (address(_inputToken) == ETH) {
       require(msg.value >= _inputTokenAmount, "Insufficient ETH sent");
       // But this alone doesn't fully prevent batch reuse — combine with batch limit
   }
   ```

5. **Add tests & invariants**  
   * Fuzz tests for multi-order ETH deposits → revert or credit correctly.  
   * Invariant: Total credited ETH value ≤ cumulative msg.value across tx.  
   * Test multicall batches with ETH.

## Pattern Recognition Notes

* **msg.value in Loops/Batches**: Never use `msg.value` inside loops or repeated calls — it's per-transaction, not per-operation. Classic in payable batch functions.
* **Multicall + Payable Functions**: `Multicall` preserves `msg.value` across sub-calls → high risk for any contract with payable entry points.
* **Native ETH Special Handling**: ETH isn't ERC20 — no `transferFrom` → must handle explicitly and prevent reuse.
* **Assumption of Single-Call**: Many contracts assume one operation per tx → breaks with batching/multicall.
* **Fix Strategy**: Either forbid batching for risky assets (ETH), or track cumulative expectations with storage (expensive).

## Quick Recall (TL;DR)

* **Bug**: `msg.value` reused across multiple orders / multicall batches when depositing ETH → credits inflated portfolio value.
* **Impact**: Attacker drains contract ETH holdings by over-crediting then withdrawing.
* **Fix**: Disallow ETH batching (single order only for ETH inputs); handle multicall defensively; consider WETH-only for deposits.
* **Lesson**: Treat `msg.value` as a transaction-level resource — never rely on it per-operation in loops or batches.
