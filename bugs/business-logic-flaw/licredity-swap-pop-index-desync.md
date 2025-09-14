# Index Desynchronization via Swap-and-Pop in Position Fungibles

* **Severity**: Critical
* **Source**: [Cyfrin Audit Report](https://github.com/solodit/solodit_content/blob/main/reports/Cyfrin/2025-09-01-cyfrin-licredity-v2.0.md#swap-and-pop-without-index-fix-up-corrupts-positions-fungible-array)
* **Affected Contract**: [Position.sol](https://github.com/Licredity/licredity-v1-core/blob/e8ae10a7d9f27529e39ca277bf56cef01a807817/src/types/Position.sol)
* **Vulnerability Type**: State Corruption / Invariant Violation / Phantom Asset

## Summary

The `Position` struct maintains two synchronized representations of collateral:

1. `fungibles[]` — a compact array of assets in the position.
2. `fungibleStates[asset]` — a mapping with each asset's `{ index, balance }`.

The invariant is:

```text
fungibles[k] == asset   ⇔   fungibleStates[asset].index == k+1
```

When a fungible's balance reaches zero, `removeFungible` uses **swap-and-pop** to compact the array. However, the implementation **forgets to update the moved element's index in `fungibleStates`**.

This causes the array and mapping to fall out of sync, leading to:

* Removal of one token unexpectedly deleting another.
* "Ghost balances" (mapping shows positive balance but asset missing from array).
* Mis-accounting in loops that value or liquidate positions.

## A Better Explanation (With Simplified Example)

### Intended Behavior

* **Deposit**:

  * `fungibles.push(asset)`
  * `fungibleStates[asset] = { index: fungibles.length, balance: amount }`

* **Remove fungible**:

  * If balance > 0 → update balance in `fungibleStates`.
  * If balance == 0 → compact array with swap-and-pop:

    ```solidity
    fungibles[index-1] = fungibles[len-1]
    fungibles.pop()
    fungibleStates[moved].index = index
    ```

### What Actually Happens (Bug)

The actual `removeFungible` assembly code does:

```solidity
if (index != len) {
    // overwrite removed slot with the last element
    sstore(dataSlot + (index-1), sload(dataSlot + (len-1)))
}
sstore(dataSlot + (len-1), 0)
sstore(slot, len-1)
```

❌ Missing: `fungibleStates[moved].index = index`.

Thus, the moved element keeps its stale index (the old tail position).

### Why This Matters

* Invariant breaks: array and mapping disagree.
* Iterations over `fungibles[]` under-count or mis-attribute assets.
* Future removals may **erase the wrong fungible**, corrupting accounting.
* Critical downstream logic (health checks, liquidation, withdrawals) may operate on wrong balances.

### Concrete Walkthrough (Alice & Bob)

* **Setup**:

  ```text
  fungibles = [A, B, C]
  index(A)=1, index(B)=2, index(C)=3
  ```

* **Remove A**:

  * Code swaps in `C` → fungibles = \[C, B].
  * But index(C) stays at 3 (should be 1).

* **Later remove C**:

  * Function trusts `index(C)=3`, length=2.
  * Attempts swap/pop with index beyond array end.
  * Result: B silently disappears, mapping still shows balance.

👉 Position is now corrupted: ghost balances + wrong array contents.

## Vulnerable Code Reference

### `removeFungible` (assembly portion)

```solidity
if iszero(eq(index, len)) {
    sstore(add(dataSlot, sub(index, 1)), sload(add(dataSlot, sub(len, 1))))
}
sstore(add(dataSlot, sub(len, 1)), 0)
sstore(slot, sub(len, 1))
```

No fix-up for the moved element's index.

## Proof of Concept

From `LicredityUnlockPosition.t.sol` (abridged):

```solidity
// deposit two fungibles
deposit(native, 0.5 ether); // fungibles[0]
deposit(token, 1 ether);    // fungibles[1]

// remove native fully (index 1)
withdraw(native);

// Assert:
// fungibles[0] == token (moved down) ✅
// fungibleStates[token].index == 2 (stale) ❌
```

**Assertion**: mapping state remains unchanged (index not fixed), proving invariant break.

## Recommended Mitigation

1. **Fix index after swap-and-pop**:

    ```solidity
    Fungible moved = fungibles[len-1];
    fungibles[index-1] = moved;
    fungibles.pop();
    fungibleStates[moved].index = index; // re-sync
    ```

2. **Invariant tests**: Add assertions that after any deposit/removal,

   ```text
   forall k: fungibleStates[fungibles[k]].index == k+1
   ```

3. **Avoid inline assembly unless necessary**: A Solidity-level swap-and-pop is easier to reason about and less error-prone.

## Pattern Recognition Notes

* **Index/Mirror State Coupling**: When one data structure mirrors another (array vs. mapping), any mutation must update both sides. Missing even a single fix-up corrupts invariants.
* **Swap-and-Pop Hazards**: Common array compaction pattern, but always requires updating any external index trackers.
* **Phantom Entries**: When mappings retain positive balances but the array loses the element, valuations will silently diverge.
* **Invariant Enforcement**: Simple assertions in unit tests (e.g., `∀a: states[a].index matches array slot`) would have caught this early.

### Quick Recall (TL;DR)

* **Bug**: `removeFungible` forgets to update moved element's index after swap-and-pop.
* **Impact**: Array and mapping desync → phantom balances, wrong removals, corrupt accounting.
* **Fix**: Update `fungibleStates[moved].index = index` after swap. Add invariant tests.
