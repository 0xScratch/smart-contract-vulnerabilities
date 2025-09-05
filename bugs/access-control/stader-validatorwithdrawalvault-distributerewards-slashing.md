# Permissionless Reward Drain Allows Unfair Operator Slashing in ValidatorWithdrawalVault

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2023-06-stader-findings/issues/86) / [One Bug Per Day](https://www.onebugperday.com/v1/279)
* **Affected Contract**: [ValidatorWithdrawalVault.sol](https://github.com/code-423n4/2023-06-stader/blob/main/contracts/ValidatorWithdrawalVault.sol)
* **Vulnerability Type**: Access Control / Business Logic (economic griefing via improper balance accounting)

## Original Bug Description

>## Lines of code
>
>[https://github.com/code-423n4/2023-06-stader/blob/main/contracts/ValidatorWithdrawalVault.sol#L29-L83](https://github.com/code-423n4/2023-06-stader/blob/main/contracts/ValidatorWithdrawalVault.sol#L29-L83)
>
>## Vulnerability details
>
>## Impact
>
>ValidatorWithdrawalVault.distributeRewards can be called to make operator slashable. Attacker can call `distributeRewards` right before `settleFunds` to make `operatorShare < penaltyAmount`. As result validator will face loses.
>
>## Proof of Concept
>
>`ValidatorWithdrawalVault.distributeRewards` can be called by anyone. It's purpose is to distribute validators rewards among stakers protocol and operator. After the call, balance of `ValidatorWithdrawalVault` becomes 0.
>
>`ValidatorWithdrawalVault.settle` is called when validator is withdrawn from beacon chain. In this case balance of contract is used to [find operatorShare](https://github.com/code-423n4/2023-06-stader/blob/main/contracts/ValidatorWithdrawalVault.sol#L62) and in case if it's less than accrued penalty by validator, then operator is slashed.
>
>Because `distributeRewards` is permissionless, then next situation is possible.
>
>1. Operator decided to withdraw validator. At the moment of that call, balance of `ValidatorWithdrawalVault` is not 0 and operatorShare is 1 eth. Also validator accrued 4.5 eth of penalty.
>2. Malicious user sees when 32 eth of validator's deposit is sent to the ValidatorWithdrawalVault and frontruns it with `distributeRewards` call. This makes balance to be 32 eth.
>3. operatorShare will be 4 eth in this time(permisssionless) and penalty is 4.5, so user is slashed.
>4. In case if malicious user didn't call `distributeRewards`, then slash would not occur.
>
>Also in same way, permissioned operator can call `distributeRewards` to get his rewards, when he is going to be slashed. As permissioned validators are not forced to have collateral to be slashed. By this move he rescued his eraning as otherwise they would be sent to pool.
>
>## Tools Used
>
>VsCode
>
>## Recommended Mitigation Steps
>
>Maybe think about restrict access to `distributeRewards` function.
>
>## Assessed type
>
>Access Control

## Summary

`ValidatorWithdrawalVault.distributeRewards()` treats the entire vault balance as "rewards" and distributes it immediately. The function is (effectively) permissionless, so an attacker can call it at an adversarial time (e.g., right before `settleFunds()` runs). Because `settleFunds()` uses the **current vault balance** to compute `userShare`, `operatorShare`, and `protocolShare`, draining the vault immediately before settlement can make the computed `operatorShare` smaller than the accrued penalty, which triggers logic that slashes the operator (or forces SD collateral to be slashed). In short: **calling `distributeRewards()` at the wrong time lets an attacker manipulate settlement math and cause unfair slashing of the operator.**

## A Better Explanation (With Simplified Example)

This section walks through the exact contract logic and then a numeric example that shows how the attack flips a no-slash outcome into a slash.

### How the contract computes shares (short)

* `TOTAL_STAKED_ETH = staderConfig.getStakedEthPerNode()` — expected total stake covered by the vault (includes user deposit and operator collateral).
* `collateralETH = getCollateralETH(poolId, staderConfig)` — operator collateral portion.
* `usersETH = TOTAL_STAKED_ETH - collateralETH` — portion attributable to users (per the contract).
* `contractBalance = address(this).balance`.

`calculateValidatorWithdrawalShare()` (simplified):

1. If `contractBalance <= usersETH`:

   * All balance goes to users (`_userShare = contractBalance`) — operator and protocol get 0.
2. Else if `contractBalance <= TOTAL_STAKED_ETH`:

   * Users get their full `usersETH`.
   * Operator gets the leftover (`contractBalance - usersETH`).
3. Else (`contractBalance > TOTAL_STAKED_ETH`):

   * `totalRewards = contractBalance - TOTAL_STAKED_ETH`.
   * Start with `_operatorShare = collateralETH` and `_userShare = usersETH`.
   * Split `totalRewards` using `IPoolUtils.calculateRewardShare(poolId, totalRewards)` into `userReward`, `operatorReward`, and `protocolReward` and add them to the base shares.

So **operatorShare at settlement depends on the contract's current balance** — if rewards exist in the vault at settlement, a portion of them becomes operatorShare; if the rewards are pre-distributed (vault drained), operatorShare is smaller.

`settleFunds()` then obtains penalty via `getUpdatedPenaltyAmount(...)` and does:

```solidity
if (operatorShare < penaltyAmount) {
  ISDCollateral.slashValidatorSD(...);
  penaltyAmount = operatorShare;
}
userShare = userSharePrelim + penaltyAmount;
operatorShare = operatorShare - penaltyAmount;
```

So if `operatorShare < penaltyAmount`, the contract triggers `slashValidatorSD` and sets `penaltyAmount` to the currently-computed `operatorShare` (effectively making operator lose its available share and forcing additional collateral slashing via SD collateral logic).

### Why the attacker can flip the outcome

* `distributeRewards()` uses `totalRewards = address(this).balance` and then calls the same `IPoolUtils.calculateRewardShare(poolId, totalRewards)` to split and transfer value.
* Because it uses the *entire* vault balance, it will attempt to distribute everything in the vault (not just the reward portion above `TOTAL_STAKED_ETH`), so calling it right before settlement effectively empties the vault of the money that `settleFunds()` expects to use to cover operator share and penalties.
* `distributeRewards()` is callable by anyone (no effective access control preventing front-running in practice), which means a malicious actor can time a call to drain the vault just before `settleFunds()` runs.

### Step-by-step numeric example (concrete)

*Assumptions used in example (simple numbers to show the flip):*

* `TOTAL_STAKED_ETH = 33` (represents 32 ETH user stake + 1 ETH operator collateral for readability)
* `collateralETH = 1`
* `usersETH = 32`
* `existing rewards in vault before withdrawal = 5` (accumulated rewards)
* `penaltyAmount` (accrued) = 2 ETH
* Reward split for `totalRewards` (example assumption produced by `IPoolUtils.calculateRewardShare`): 70% users, 25% operator, 5% protocol. (This is only for the numeric example; the contract calls IPoolUtils to compute exact split.)

**Normal flow (no attack):**

1. Suppose the validator exit causes the expected stake (`TOTAL_STAKED_ETH = 33`) plus rewards to be present, so `contractBalance = TOTAL_STAKED_ETH + rewards = 33 + 5 = 38`.
2. `calculateValidatorWithdrawalShare()` sees `contractBalance > TOTAL_STAKED_ETH`, so:

   * `_operatorShare = collateralETH = 1`
   * `_userShare = usersETH = 32`
   * `totalRewards = 5` → reward split: `userReward = 3.5`, `operatorReward = 1.25`, `protocolReward = 0.25`.
   * Final shares: `_userShare = 32 + 3.5 = 35.5`, `_operatorShare = 1 + 1.25 = 2.25`, `_protocolShare = 0.25` (sum = 38).
3. `settleFunds()` compares `operatorShare = 2.25` with `penaltyAmount = 2`.

   * `operatorShare >= penaltyAmount` → **no SD-collateral slash**, settlement distributes funds normally.

**Attack flow (attacker calls `distributeRewards()` just before settlement):**

1. Attacker calls `distributeRewards()` while vault has `contractBalance = 38`.

   * `distributeRewards()` sets `totalRewards = address(this).balance = 38` and calls `IPoolUtils.calculateRewardShare(poolId, 38)` — in effect it attempts to split the whole vault balance as "rewards" and transfers out `userShare`, `operatorShare`, `protocolShare` accordingly (draining the vault). After the call the vault balance is 0 (or very small depending on gas/intermediate steps).
2. Now `settleFunds()` runs. `calculateValidatorWithdrawalShare()` reads `contractBalance = 0`.

   * Because `contractBalance <= usersETH (32)`, the function returns `_userShare = 0`, `_operatorShare = 0`, `_protocolShare = 0`.
3. `settleFunds()` computes `operatorShare < penaltyAmount` (0 < 2) → triggers `ISDCollateral.slashValidatorSD(...)` and sets `penaltyAmount = operatorShare (0)`.

   * Effect: operator receives no settlement funds and their SD collateral is slashed by the penalty module — the operator suffers losses because the vault was drained right before settlement.

**Net effect:** In the normal flow the operator would have had `operatorShare = 2.25` which covered the `2 ETH` penalty. In the attack flow the vault was drained and `operatorShare` at settlement appears as `0`, forcing a collateral slash (or at least an unjust loss). The only difference between the two flows is the permissionless `distributeRewards()` call.

### Note on sequence & front-running

The vulnerability relies on an attacker being able to call `distributeRewards()` in the window between when the final funds appear in the vault and when `settleFunds()` runs — this is a classic frontrunning/griefing window. Because `distributeRewards()` uses the raw `address(this).balance` for distribution instead of isolating the actual reward portion, it is able to drain funds that `settleFunds()` expects to exist.

## Vulnerable Code Reference

Key lines (conceptual):

* `distributeRewards()` uses vault balance directly:

    ```solidity
    uint256 totalRewards = address(this).balance;
    // ... split using pool utils and sendUserShare / sendOperatorShare / sendProtocolShare
    ```

* `calculateValidatorWithdrawalShare()` computes operator/user/protocol shares from the **current** balance:

    ```solidity
    uint256 usersETH = TOTAL_STAKED_ETH - collateralETH;
    uint256 contractBalance = address(this).balance;
    if (contractBalance <= usersETH) { _userShare = contractBalance; return; }
    else if (contractBalance <= TOTAL_STAKED_ETH) { _userShare = usersETH; _operatorShare = contractBalance - _userShare; return; }
    else { totalRewards = contractBalance - TOTAL_STAKED_ETH; _operatorShare = collateralETH; _userShare = usersETH; /* then split rewards */ }
    ```

* `settleFunds()` triggers SD-collateral slash if `operatorShare < penaltyAmount`.

(See full file: `ValidatorWithdrawalVault.sol` — starting at the `distributeRewards` function through `settleFunds` and `calculateValidatorWithdrawalShare`.)

## Recommended Mitigation

1. **Do not let `distributeRewards()` treat the entire `address(this).balance` as distributable rewards.**

   * Change the logic to compute *only the reward portion* before distributing. For example:

        ```solidity
        uint256 TOTAL_STAKED_ETH = staderConfig.getStakedEthPerNode();
        uint256 contractBalance = address(this).balance;
        uint256 distributableRewards = 0;
        if (contractBalance > TOTAL_STAKED_ETH) {
            distributableRewards = contractBalance - TOTAL_STAKED_ETH; // only rewards above base stake
        } else {
            revert NotEnoughRewardToDistribute();
        }
        ```

    Then split `distributableRewards` (not the full `contractBalance`). This prevents accidental distribution of posted stake or collateral that settleFunds expects.

2. **Restrict access to `distributeRewards()`** so only intended roles can call it (eg. `onlyOperatorRole` or `onlyStakePoolManager`). If a permissionless API is desired, add stronger invariants (see #1) and a time-delay/cooldown to avoid immediate front-running.

3. **Add an explicit "settling in progress" guard** around settlement flows. For instance, mark the vault as `settleInitiated` (or use `vaultSettleStatus`) and reject reward distributions once settlement begins. Example:

    ```solidity
    require(!settleInProgress, "settlement in progress");
    ```

4. **Use internal accounting rather than on-chain `balance` for reward tracking.** Track `accumulatedRewards` explicitly, update it when rewards are received, and have `distributeRewards()` operate only on the tracked `accumulatedRewards` counter. This prevents mixing "reward accounting" with raw balance (which can include deposits in transit).

5. **Emit events and add assertions/tests:**

   * Emit an event when `distributeRewards()` runs with `(balanceBefore, distributableRewards)`.
   * Unit/integration tests that simulate: (a) front-running calls to `distributeRewards()` during exit, and (b) settlement under various timing and balance permutations.

6. **Add a last-resort check in `settleFunds()`** to use a canonical source for reward accounting (e.g., stored `accumulatedRewards`) rather than trusting `address(this).balance`. If `settleFunds()` must trust `balance`, ensure `distributeRewards()` cannot drain the exact amounts that settlement expects.

## Pattern Recognition Notes

* **Access control mistakes + balance misuse are a common dangerous combo.** Functions that move funds must be careful to only move the intended subset of balance (rewards vs deposits vs collateral).
* **Front-running windows:** Any multi-step flow that depends on contract balance in multiple external transactions is vulnerable to reordering/front-running unless guarded by roles, delays, or atomic operations. Always think about whether an external caller can change the balance between steps.
* **Conflate "reward accounting" with `address(this).balance` at your peril.** Explicit counters or accounting variables avoid ambiguity about what funds are distributable.
* **Failure modes in settlement logic:** If settlement compares computed shares to penalties and triggers collateral slashes, test for scenarios where an attacker can artificially alter the computed shares by manipulating balances.
* **Permissioned vs permissionless operators:** Permissioned operators without collateral or with different economic protection can create edge-cases; ensure paths that depend on operator collateral handle permissioned validators correctly.
