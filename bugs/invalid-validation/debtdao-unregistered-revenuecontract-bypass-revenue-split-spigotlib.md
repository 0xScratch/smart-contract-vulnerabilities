# Revenue Split Bypass via Unregistered Revenue Contract in Spigot

* **Severity**: High
* **Source**: [Code4rena](https://github.com/code-423n4/2022-11-debtdao-findings/issues/119)
* **Affected Contract**: [SpigotLib.sol](https://github.com/debtdao/Line-of-Credit/blob/f32cb3eeb08663f2456bf6e2fba21e964da3e8ae/contracts/utils/SpigotLib.sol)
* **Vulnerability Type**: Business Logic Flaw / Missing Validation / Revenue Bypass

## Summary

`SpigotLib.claimRevenue()` and `_claimRevenue()` fail to verify whether a provided `revenueContract` is registered before processing revenue. If an unregistered contract is passed, the logic assumes it is a **push-payment revenue contract** because `claimFunction` defaults to `0`.

Since unregistered contracts also return default settings where `ownerSplit = 0`, **all claimed revenue is sent to the borrower treasury instead of being split with lenders**.

This allows a borrower to repeatedly call `claimRevenue()` with a **fake revenue contract** and divert **100% of revenue to themselves**, bypassing the intended revenue split.

## A Better Explanation (With Simplified Example)

### Intended Behavior

Debt DAO's **Spigot** is designed to:

1. Collect revenue from configured revenue contracts
2. Split revenue between:

   * Lenders (escrow)
   * Borrower (treasury)

Example:

```text
Revenue = 1000 USDC
Owner Split = 30%

Lenders → 300 USDC
Borrower → 700 USDC
```

### What Actually Happens (Bug)

The contract **does not verify** that `revenueContract` exists:

```solidity
self.settings[revenueContract]
```

If not registered:

```yaml
ownerSplit = 0
claimFunction = 0
transferOwnerFunction = 0
```

Then `_claimRevenue()` treats this as **push payment**:

```solidity
if(self.settings[revenueContract].claimFunction == bytes4(0))
```

Revenue is calculated:

```solidity
claimed = existingBalance - self.escrowed[token];
```

Then in `claimRevenue()`:

```solidity
escrowedAmount = claimed * ownerSplit / 100;
```

Since:

```yaml
ownerSplit = 0
```

Result:

```yaml
escrowedAmount = 0
```

Therefore:

```text
100% → Borrower treasury
0% → Lenders
```

## Why This Matters

This vulnerability allows:

* Borrower to **steal all revenue**
* Lenders to **never receive repayment**
* Entire Debt DAO repayment logic to break

This is particularly dangerous for **push payment revenue tokens**.

## Concrete Walkthrough (Alice & Mallory)

### Setup

Configured revenue contract:

```text
ownerSplit = 30%
```

Spigot balance:

```text
Before = 1000 USDC
After revenue = 1500 USDC
```

Revenue:

```text
500 USDC
```

### Mallory Attack

Mallory (borrower) calls:

```solidity
claimRevenue(address(0), USDC)
```

Because:

```text
address(0) not registered
```

Contract assumes push payment:

```yaml
claimed = 1500 - 1000 = 500
```

Split:

```yaml
ownerSplit = 0
```

Final Result:

```text
Borrower → 500 USDC
Lenders → 0 USDC
```

Mallory can repeat this indefinitely.

## Vulnerable Code Reference

### 1) Missing Registration Validation

```solidity
if(self.settings[revenueContract].claimFunction == bytes4(0)) {
    claimed = existingBalance - self.escrowed[token];
}
```

No check exists to confirm:

```yaml
revenueContract is registered
```

### 2) Default Mapping Values Used

```solidity
uint256 escrowedAmount = claimed * self.settings[revenueContract].ownerSplit / 100;
```

For unregistered contract:

```yaml
ownerSplit = 0
```

This bypasses revenue split.

## Recommended Mitigation

### 1. Validate Registered Revenue Contract (Primary Fix)

```solidity
require(
    self.settings[revenueContract].transferOwnerFunction != bytes4(0),
    "Revenue contract not registered"
);
```

This ensures only configured revenue contracts can be used.

### 2. Defensive Programming

Alternatively:

```solidity
ISpigot.Setting memory setting = self.settings[revenueContract];
require(setting.ownerSplit > 0 || setting.claimFunction != bytes4(0));
```

### 3. Add Unit Tests

Test cases:

* claim with unregistered contract should revert
* claim with registered contract should pass

## Pattern Recognition Notes

### Missing Mapping Entry Validation

Mappings return default values:

```text
mapping[key] → default struct
```

If existence not validated:

```yaml
logic bypass
```

### Default Value Exploitation

Using:

```text
0 as sentinel value
```

Can cause logic bypass.

### Business Logic Manipulation

Not a technical bug — a **logic flaw**.

Often harder to detect.

### Push Payment Exploitation

Push-based revenue increases balance silently:

```yaml
balance increases
claim triggered manually
```

Common DeFi risk.

## Quick Recall (TL;DR)

* **Bug**: `claimRevenue()` accepts unregistered revenue contracts
* **Impact**: Borrower can steal 100% of revenue
* **Cause**: Missing validation + default mapping values
* **Fix**: Validate revenue contract registration before claim
