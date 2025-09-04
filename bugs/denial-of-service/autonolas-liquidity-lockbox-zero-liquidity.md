# Global Withdraw DoS via Zero‑Liquidity Position in Liquidity Lockbox

* **Severity**: High
* **Source**: [Code4rena](https://github.com/code-423n4/2023-12-autonolas/blob/2a095eb1f8359be349d23af67089795fb0be4ed1/lockbox-solana/solidity/liquidity_lockbox.sol#L84) / [One Bug Per Day](https://www.onebugperday.com/v1/755)
* **Affected Contract**: [`liquidity_lockbox.sol`](https://github.com/code-423n4/2023-12-autonolas/blob/2a095eb1f8359be349d23af67089795fb0be4ed1/lockbox-solana/solidity/liquidity_lockbox.sol)
* **Vulnerability Type**: Denial of Service (DoS) / Input Validation / Queue Poisoning

## Original Bug Description

>## Lines of code
>
>[https://github.com/code-423n4/2023-12-autonolas/blob/2a095eb1f8359be349d23af67089795fb0be4ed1/lockbox-solana/solidity/liquidity_lockbox.sol#L84](https://github.com/code-423n4/2023-12-autonolas/blob/2a095eb1f8359be349d23af67089795fb0be4ed1/lockbox-solana/solidity/liquidity_lockbox.sol#L84)
>
>## Vulnerability details
>
>## Impact
>
>It won't be possible to withdraw any LP token after doing a deposit of
> 0 liquidity, leading to withdrawals being effectively freezed.
>
>## Proof of Concept
>
>In
>
>[liquidity_lockbox, function withdraw](https://github.com/code-423n4/2023-12-autonolas/blob/2a095eb1f8359be349d23af67089795fb0be4ed1/lockbox-solana/solidity/liquidity_lockbox.sol#L221C1-L225C10)
>
>```solidity
>        ...
>
>        uint64 positionLiquidity = mapPositionAccountLiquidity[positionAddress];
>        // Check that the token account exists
>        if (positionLiquidity == 0) {
>            revert("No liquidity on a provided token account");
>        }
>
>        ...
>```
>
>the code checks for the existence of a position via the recorded liquidity. This is a clever idea, as querying a non-existant value from a mapping will return 0. However, in `deposit`, due to a flawed input validation, it is possible to make positions with 0 liquidity as the only check being done is for liquidity to not be higher than `type(uint64).max`:
>
>[liquidity_lockbox, function _getPositionData](https://github.com/code-423n4/2023-12-autonolas/blob/2a095eb1f8359be349d23af67089795fb0be4ed1/lockbox-solana/solidity/liquidity_lockbox.sol#L94C1-L97C10)
>
>```solidity
>        ...
>
>        // Check that the liquidity is within uint64 bounds
>        if (positionData.liquidity > type(uint64).max) {
>            revert("Liquidity overflow");
>        }
>
>        ...
>```
>
>As it will pass the input validation inside `_getPositionData`, the only way for such a tx to revert is in the [transfer](https://github.com/code-423n4/2023-12-autonolas/blob/2a095eb1f8359be349d23af67089795fb0be4ed1/lockbox-solana/solidity/liquidity_lockbox.sol#L164)/[mint](https://github.com/code-423n4/2023-12-autonolas/blob/2a095eb1f8359be349d23af67089795fb0be4ed1/lockbox-solana/solidity/liquidity_lockbox.sol#L171), which are low-level calls with no checks for success, as stated in my report `Missing checks for failed calls to the token program will corrupt user's positions`.
>
>Due to the reasons above, this deposit with 0 liquidity will be treated as a valid one and will be stored inside the `mapPositionAccountLiquidity` and `positionAccounts` arrays. If we add the fact that withdrawals are done by looping **LINEARLY** through **positionAccounts**:
>
>[liquidity_lockbox, function withdraw](https://github.com/code-423n4/2023-12-autonolas/blob/2a095eb1f8359be349d23af67089795fb0be4ed1/lockbox-solana/solidity/liquidity_lockbox.sol#L192C1-L323C6)
>
>```solidity
>    function withdraw(uint64 amount) external {
>        address positionAddress = positionAccounts[firstAvailablePositionAccountIndex]; // @audit linear loop
>        
>        ...
>
>        uint64 positionLiquidity = mapPositionAccountLiquidity[positionAddress];
>        // Check that the token account exists
>        if (positionLiquidity == 0) { // @audit it will revert here once it reaches the flawed position
>            revert("No liquidity on a provided token account");
>        }
>
>        ...
>
>        if (remainder == 0) { // @audit if the liquidity after the orca call is 0, close the position and ++ the index
>            ...
>
>            // Increase the first available position account index
>            firstAvailablePositionAccountIndex++; // @audit it won't reach here as the revert above will roll-back the whole tx
>        }
>    }
>```
>
>It can be seen that once it encounters such a "fake" deposit with 0 liquidity provided, it will always revert due to the existence check. As there is no other way to update `firstAvailablePositionAccountIndex` to bypass the flawed position, withdrawals will be completely freezed.
>
>## Recommended Mitigation Steps
>
>Just check for the supplied liquidity to not be 0 in
>
>[liquidity_lockbox, function _getPositionData](https://github.com/code-423n4/2023-12-autonolas/blob/2a095eb1f8359be349d23af67089795fb0be4ed1/lockbox-solana/solidity/liquidity_lockbox.sol#L94C1-L97C10)
>
>```diff
>        ...
>+       // Check that the liquidity > 0
>+       if (positionData.liquidity == 0) {
>+           revert("Liquidity cannot be 0");
>+       }
>
>        // Check that the liquidity is within uint64 bounds
>        if (positionData.liquidity > type(uint64).max) {
>            revert("Liquidity overflow");
>        }
>
>        ...
>```
>
>## Assessed type
>
>DoS

## Summary

`liquidity_lockbox` stores Orca Whirlpool LP positions (NFTs) and mints **bridged tokens** equal to the position's **liquidity**. During withdrawal, the contract linearly iterates through stored positions and **uses the recorded liquidity as the existence check**.

Because deposits **do not reject zero‑liquidity positions**, an attacker can deposit a valid position with `liquidity == 0`. This zero‑liquidity entry gets recorded in on‑chain state and, when encountered during withdrawal, the function **reverts** on the `positionLiquidity == 0` guard. Since the iteration pointer only advances after successful processing, the system gets stuck on the poisoned entry and **all withdrawals are permanently frozen**.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. **Deposit**: User deposits an Orca LP NFT. Contract reads `liquidity` from the position and records it:

   * `mapPositionAccountLiquidity[position] = liquidity`
   * `positionAccounts[numPositionAccounts++] = position`
   * Mints `liquidity` amount of bridged tokens to the user.
2. **Withdraw**: User burns bridged tokens; contract walks the `positionAccounts` array from `firstAvailablePositionAccountIndex`, decrementing position liquidity and closing positions when they reach zero. After fully processing a position, it increments `firstAvailablePositionAccountIndex` to move past it.

### What Actually Happens (Bug)

* `_getPositionData` **only** checks that `liquidity <= type(uint64).max` and does **not** require `liquidity > 0`.
* Therefore, a deposit where the underlying Orca position reports `0` liquidity **passes validation** and is **stored**.
* In `withdraw`, the code pulls the next position address from `positionAccounts[firstAvailablePositionAccountIndex]`, looks up `positionLiquidity`, and **reverts if it equals 0**. Because state updates occur **after** the critical operations, the revert prevents advancing the index, so the poisoned entry is encountered **every time** → **system‑wide DoS**.

### Why This Matters

* A single zero‑liquidity deposit can **brick the withdrawal pipeline for everyone**.
* DoS persists until a code change or manual state intervention occurs (no in‑protocol bypass).
* The risk is amplified by missing success checks on token calls during `deposit` (even a failed mint/transfer could still result in a recorded entry).

### Concrete Walkthrough (Alice & Mallory)

* **Setup**: `firstAvailablePositionAccountIndex = 0`; `positionAccounts = []`.
* **Mallory attack**: Mallory deposits an LP NFT where `liquidity = 0`.

  * Validation passes (no `== 0` check).
  * Contract records: `positionAccounts[0] = posMallory`; `mapPositionAccountLiquidity[posMallory] = 0`.
* **Alice withdraws**: `withdraw(amount)` reads `positionAddress = positionAccounts[0] = posMallory`.

  * Looks up `positionLiquidity = mapPositionAccountLiquidity[posMallory] = 0`.
  * Hits guard: `if (positionLiquidity == 0) revert("No liquidity on a provided token account");` → **revert**.
  * Because the revert happens **before** `firstAvailablePositionAccountIndex++`, the pointer never advances. Every subsequent attempt hits the same poisoned entry and reverts again → **global withdraw freeze**.

> **Analogy**: A checkout line that moves one customer at a time. An attacker inserts a "ghost customer" (zero items but invalid ticket). The cashier halts on the ghost, never scanning anyone behind. Since the line only advances after a successful checkout, the entire queue stalls forever.

## Vulnerable Code Reference

### 1. Missing `liquidity > 0` validation in `_getPositionData`**

```solidity
// _getPositionData(...)
// ...
// Check that the liquidity is within uint64 bounds
if (positionData.liquidity > type(uint64).max) {
    revert("Liquidity overflow");
}
// Missing: require(positionData.liquidity > 0)
// ...
```

### 2. Deposit records zero‑liquidity entries**

```solidity
// deposit()
Position positionData = _getPositionData(tx.accounts.position, tx.accounts.positionMint.key);
uint64 positionLiquidity = uint64(positionData.liquidity);
// ...
mapPositionAccountLiquidity[positionAddress] = positionLiquidity; // can be 0
positionAccounts[numPositionAccounts] = positionAddress;          // stored in iteration queue
numPositionAccounts++;
```

### 3. Withdraw uses liquidity as existence check and never advances past poisoned entries**

```solidity
// withdraw(uint64 amount)
address positionAddress = positionAccounts[firstAvailablePositionAccountIndex];
uint64 positionLiquidity = mapPositionAccountLiquidity[positionAddress];
if (positionLiquidity == 0) {
    revert("No liquidity on a provided token account"); // permanent stall on poisoned entry
}
// ...
if (remainder == 0) {
    // close position and advance pointer
    firstAvailablePositionAccountIndex++; // never reached when revert above triggers
}
```

## Recommended Mitigation

1. **Reject zero‑liquidity deposits** (primary fix)

    ```solidity
    // _getPositionData(...)
    if (positionData.liquidity == 0) {
        revert("Liquidity cannot be 0");
    }
    ```

2. **Defensive programming in `withdraw`**: Consider skipping zero‑liquidity entries rather than reverting, to prevent permanent stalls from unexpected state:

    ```solidity
    if (positionLiquidity == 0) {
        // skip and advance pointer (optionally, emit an event for observability)
        firstAvailablePositionAccountIndex++;
        return; // or continue processing next position in a looped form
    }
    ```

3. **Check external token program call results**: Ensure SPL token transfers/mints/burns are asserted to succeed so the contract state cannot be corrupted by failed side‑effects.

4. **Add invariants and tests**: Unit tests for deposit with `liquidity == 0` should fail; property tests asserting `firstAvailablePositionAccountIndex` cannot be blocked by malformed entries.

## Pattern Recognition Notes

* **Sentinel Conflation**: Using `0` as both "no mapping entry" and a potentially valid value invites logic errors. Always validate that a value can never legally be zero if you rely on zero to mean "absent."
* **Queue Poisoning via Linear Iteration**: Systems that process arrays by a moving index are vulnerable when a single bad element can permanently prevent pointer advancement.
* **Missing External Call Assertions**: When side‑effects are critical (token transfers/mints), lack of success checks can desynchronize state from reality and exacerbate failures.
* **Boundary Validation at Ingress**: Enforce invariants early (on deposit) rather than relying on downstream code to cope with malformed inputs.
* **Recoverability**: Where possible, design withdraw flows to be **skip‑capable** (tolerant to bad entries) and add admin/emergency mechanisms to quarantine or purge malformed positions.

## Quick Recall (TL;DR)

* **Bug**: Deposit accepts `liquidity == 0` → zero‑liquidity position gets queued.
* **Impact**: Withdraw hits that entry, reverts on `== 0`, never advances → **global DoS**.
* **Fix**: Require `liquidity > 0` on deposit; optionally make withdraw skip zero entries; assert external token calls succeed.
