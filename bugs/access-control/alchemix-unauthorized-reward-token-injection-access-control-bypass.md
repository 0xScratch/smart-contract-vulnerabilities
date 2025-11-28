# Unauthorized Reward Token Injection via `notifyRewardAmount` in Alchemix Bribe Contract

* **Severity**: Medium
* **Source**: [Immunefi](https://github.com/immunefi-team/Past-Audit-Competitions/blob/main/Alchemix/31462%20-%20%5BSC%20-%20Medium%5D%20Alchemix%20%20addReward%20access%20control%20can%20be%20bypas....md)
* **Affected Contract**: [`Bribe.sol`](https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Bribe.sol) (Not sure whether it is in the same vulnerable state)
* **Vulnerability Type**: Access Control Bypass / State Poisoning / Governance Manipulation

## Summary

The `Bribe` contract restricts reward-token registration using an access-controlled function `addRewardToken()`, which only the **authorized gauge** may call. This ensures that even though many tokens may be **whitelisted** by the voter contract, only the gauge controls which tokens actually become **active bribe reward tokens**.

However, the function `notifyRewardAmount()` also calls the internal `_addRewardToken()` — but **without any access control**. It only checks that `token` is whitelisted and `amount > 0`.

As a result, **any user** may add **any whitelisted token** into the bribe's reward list by donating even **1 wei**. This bypasses intended gauge-only permissions, **poisons the reward token array**, breaks index-based admin operations, and enables front-running attacks against `swapOutRewardToken()`.

## A Better Explanation (With Simplified Example)

### Intended Behavior

The protocol separates responsibilities:

1. **Voter contract** — maintains a **whitelist** of allowed tokens.

2. **Gauge** — chooses which whitelisted tokens to activate as **reward tokens** by calling:

   ```solidity
   addRewardToken(token)  // only gauge
   ```

3. When rewards are distributed:

   * Users transfer tokens to the Bribe contract.
   * Only previously-added tokens appear in the `rewards[]` list.
   * Admin may perform maintenance using index-based operations such as:

     ```solidity
     swapOutRewardToken(oldIndex, oldToken, newToken)
     ```

### What Actually Happens (Bug)

`notifyRewardAmount()` contains this unprotected logic:

```solidity
require(IVoter(voter).isWhitelisted(token));
_addRewardToken(token);
IERC20(token).safeTransferFrom(msg.sender, address(this), amount);
```

This means:

✔ Any whitelisted token
✔ With any positive amount
→ **Gets automatically added to the reward list**

This fully bypasses:

```solidity
require(msg.sender == gauge);
```

found in `addRewardToken()`.

### Why This Matters

* The gauge's authority to decide reward tokens is nullified.

* Anyone can inject arbitrary whitelisted tokens into the reward list.

* Since tokens are stored in an **array**, this changes indexes:

  ```text
  rewards[]
  ^ attacker can push new entries anytime
  ```

* Admin functions like `swapOutRewardToken(index, ...)` become exploitable — an attacker may front-run a swap and shift array indexes, forcing the admin to swap the **wrong** token or causing a revert.

### Concrete Walkthrough (Alice & Mallory)

* **Setup**:
  Suppose `rewards = [TOKEN_A, TOKEN_B]`.

* **Admin wants**: swap token at index `1` (TOKEN_B).

* **Mallory attack**:

  ```solidity
  notifyRewardAmount(USDC, 1); // USDC is whitelisted
  ```

  * Adds USDC to `rewards` → now:

    ```text
    rewards = [TOKEN_A, TOKEN_B, USDC]
    ```

* Before admin's tx executes, the index of TOKEN_B changes (still 1, but now other swaps may be targeted or scripts expecting fixed indices break).

* More importantly, repeated injections allow arrays to grow unexpectedly and DOS admin intentions.

This is classic **index poisoning** + **front-running** + **access-control bypass**.

## Vulnerable Code Reference

### 1) Intended restricted entry point

```solidity
function addRewardToken(address token) external {
    require(msg.sender == gauge, "not being set by a gauge");
    _addRewardToken(token);
}
```

### 2) Unsafe unrestricted entry point

```solidity
function notifyRewardAmount(address token, uint256 amount) external lock {
    require(amount > 0);
    require(IVoter(voter).isWhitelisted(token));
    _addRewardToken(token);  // <-- UNGUARDED
    IERC20(token).safeTransferFrom(msg.sender, address(this), amount);
    ...
}
```

Since `_addRewardToken()` pushes to the array:

```solidity
if (!isReward[token]) {
    rewards.push(token);  // <-- array poisoning
}
```

anyone can expand the rewards list.

## Impact Details

1. **Unauthorized reward token registration**
   Anyone can add any whitelisted token, bypassing gauge authority.

2. **State poisoning (array pollution)**
   Attacker can add dozens of tokens, making UIs, off-chain scripts, or integrators misinterpret the reward set.

3. **Front-running attack on `swapOutRewardToken()`**
   Since `swapOutRewardToken` uses **indexes**, newly added tokens change the index layout—causing:

   * Admin operations to target wrong tokens
   * Undesired swaps
   * Potential reverts
   * Temporary DoS on reward-maintenance flows

4. **Ecosystem-side harm**
   External protocols querying the reward list may interpret bogus entries.

Although funds are not directly stolen, governance and integrity of the bribe system are compromised.

## Recommended Mitigation

### Primary Fix

Remove automatic reward-token addition inside `notifyRewardAmount`:

```solidity
// Remove this line
_addRewardToken(token);
```

Require that tokens must be **previously added** via the authorized gauge-only method.

### Additional Hardening

* Add explicit checks:

  ```solidity
  require(isReward[token], "reward token not registered");
  ```

* Consider event‐logging for rejected reward attempts.

* Validate that `rewards.length < MAX_REWARD_TOKENS` is not externally force-triggerable.

* Add invariant tests ensuring that reward registration is only possible through the gauge entrypoint.

## Pattern Recognition Notes

* **Access-Control Bypass via Shared Internal Function**
  If both privileged and unprivileged functions call the same sensitive internal function, the internal function becomes the real access-control boundary. `_addRewardToken()` was originally meant to be "gauge-only" but became globally accessible.

* **Array-Based State Poisoning**
  Appending elements to publicly observable arrays without authorization allows attackers to manipulate off-chain logic and index-based on-chain flows.

* **Front-Running Index Manipulation**
  Any contract that depends on fixed array positions (e.g., admin calling `swapOutRewardToken(index)`) is vulnerable to attackers pushing new elements before the operation executes.

* **Whitelist ≠ Permission**
  Whitelisting is often misunderstood as "permission to interact," but not "permission to modify global reward state." This bug stemmed from conflating these two concepts.

* **Always isolate reward-token registration**
  Reward-token addition is governance-sensitive; it must be protected behind a single authoritative entry point.

### Quick Recall (TL;DR)

* **Bug**: `notifyRewardAmount()` calls `_addRewardToken()` without access control.
* **Impact**: Anyone can add whitelisted reward tokens → array pollution → front-running → admin mis-swaps → DoS in maintenance operations.
* **Fix**: Remove reward-token addition from `notifyRewardAmount()` and keep it gauge-only.
