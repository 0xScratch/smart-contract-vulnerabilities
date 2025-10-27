# Incorrect Vesting Interest Calculation Enables MEV Exploitation

* **Severity**: High
* **Source**: [Pashov Audit Group](https://github.com/pashov/audits/blob/master/team/md/LoopVaults-security-review_2025-04-30.md#h-01-incorrect-vesting-interest-calculation-enables-mev-attacks)
* **Affected Contract**: Vault accounting logic (`totalAssets`, `_vestingInterest`)
* **Vulnerability Type**: Accounting Error / MEV Exploitability / Incorrect Vesting Logic

## Summary

The vault uses a vesting mechanism to **smooth out interest accrual** between updates, aiming to reduce MEV risks. However, the implementation of `_vestingInterest()` inverts the logic:

* Right after an update, it reports **all interest as instantly available**,
* Then gradually reduces the reported interest to zero.

This creates a **critical MEV window** where bots can deposit/withdraw immediately after an update and steal unvested interest.

## A Better Explanation (With Simplified Example)

### Intended Behavior

The vault should release interest gradually over a `vestingDuration` period:

* At update: 0% interest released
* Halfway: 50% interest released
* End: 100% interest released

```text
Time →    0m    5m    10m
Interest  0% ---50%---100%
```

This way, no one can instantly grab all accrued interest.

### What Actually Happens (Bug)

Due to the inverted formula, `_vestingInterest()` does the opposite:

* At update: 100% interest appears available
* Halfway: only ~50% reported
* End: 0% reported

```test
Time →    0m    5m    10m
Interest 100% --50%---0%
```

This means **`totalAssets()` looks inflated right after update**, which makes the vault vulnerable.

### Why This Matters

Attackers (MEV bots) can monitor the blockchain for vault updates and:

1. **Deposit right after update** → their shares are valued against inflated `totalAssets()`.
2. **Withdraw later** → they walk away with a share of interest they didn't earn.

Even though the vesting mechanism was supposed to *prevent* MEV, the bug **creates the very attack vector it tried to avoid**.

### Concrete Walkthrough (Alice & Mallory)

* **Setup**: Vault has `vestingInterest = 100 tokens`, `vestingDuration = 10 minutes`.

* **Mallory's attack**:

  * Right after update (timestamp = `lastUpdate`), `_vestingInterest() == 0`.
  * So `totalAssets() = lastTotalAssets - 0 = lastTotalAssets + full interest`.
  * Vault looks like it has **100 extra tokens** ready.

* **Exploit**:

  * Mallory deposits now → gets shares as if those 100 tokens belonged to everyone.
  * Later, she withdraws → exits with more than her fair share, stealing yield from real users.

## Vulnerable Code Reference

### 1) Mis-calculated vesting function

```solidity
function _vestingInterest() internal view returns (uint256) {
    if (block.timestamp - lastUpdate >= vestingDuration) return 0;

    uint256 __vestingInterest =
        (block.timestamp - lastUpdate) * vestingInterest / vestingDuration;

    return __vestingInterest; // Wrong: ramps UP from 0 → full interest
}
```

### 2) `totalAssets` subtracts `_vestingInterest`

```solidity
function totalAssets() public view override returns (uint256) {
    return lastTotalAssets - _vestingInterest();
}
```

At update time, `_vestingInterest() = 0`, so the vault reports *all* interest as if vested.

## Recommended Mitigation

Change the vesting formula to decrease from full interest → 0 instead of the reverse:

```diff
- uint256 __vestingInterest =
-     (block.timestamp - lastUpdate) * vestingInterest / vestingDuration;
+ uint256 __vestingInterest =
+     (vestingDuration - (block.timestamp - lastUpdate)) * vestingInterest / vestingDuration;
```

This way:

* At update → `_vestingInterest = vestingInterest` → `totalAssets` unchanged.
* As time passes → `_vestingInterest` decreases, gradually releasing interest.
* At vesting end → `_vestingInterest = 0`, so full interest included.

---

## Pattern Recognition Notes

* **Vesting Logic Errors**: When designing linear release schedules, it's easy to invert "start at 0 → grow" vs. "start at full → decay." Always double-check boundary conditions (`t=0`, `t=vestingDuration`).
* **Boundary Condition Exploits**: MEV bots target predictable block timestamps where accounting flips sharply (here: right after update).
* **Accounting Transparency**: Functions like `totalAssets()` are critical for user share pricing; any misreporting directly enables value extraction.
* **Defense-in-Depth**: Vesting mechanisms must be tested with unit tests specifically for `t=0`, `t=mid`, `t=end` to catch inverted formulas.

### Quick Recall (TL;DR)

* **Bug**: `_vestingInterest` starts at 0 and ramps up, so right after update, `totalAssets` shows full interest.
* **Impact**: MEV bots can deposit/withdraw instantly after updates to steal yield.
* **Fix**: Reverse the formula so vesting starts at full interest and linearly decays to 0.
