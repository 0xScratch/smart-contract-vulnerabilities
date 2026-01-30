# Oracle Data Corruption via Storage Key Collision in Ekubo Oracle

* **Severity**: Medium
* **Source**: [Code4rena](https://code4rena.com/audits/2025-11-ekubo/submissions/F-21)
* **Affected Contract**: [Oracle.sol](https://github.com/code-423n4/2025-11-ekubo/blob/main/src/extensions/Oracle.sol)
* **Vulnerability Type**: Storage Collision / Data Corruption / Oracle Integrity Failure

## Summary

The Ekubo `Oracle` extension manually derives storage slots for oracle data using **raw address arithmetic** instead of cryptographic hashing.
Specifically:

* Oracle **metadata (`Counts`)** is stored at storage slot `token`
* Oracle **snapshots (`Snapshot[index]`)** are stored at slot `(token << 32) | index`

This storage layout allows a **collision** between:

* the `Counts` struct of one token (`tokenA`), and
* the `Snapshot[0]` struct of another token (`tokenB`),

when `tokenA == (tokenB << 32)`.

If a victim token (`tokenB`) has **8 leading zero hex characters** in its address, an attacker can deliberately choose a second address (`tokenA`) that collides with `Snapshot[0]` of `tokenB`. Initializing an oracle pool for `tokenA` overwrites historical oracle data of `tokenB`, permanently corrupting its price and liquidity history.

This breaks the correctness guarantees of the oracle and all downstream consumers (e.g., ERC-7726, `PriceFetcher`) that rely on Ekubo's historical oracle data.

## A Better Explanation (With Simplified Example)

### Intended Behavior

For each token tracked by the oracle:

1. **Counts (metadata)** is stored once:

   * current snapshot index
   * total snapshot count
   * snapshot capacity
   * last update timestamp

   Stored at:

   ```css
   storage[token]
   ```

2. **Snapshots (historical observations)** are stored in a circular buffer:

   * timestamp
   * `tickCumulative`
   * `secondsPerLiquidityCumulative`

   Stored at:

   ```pgsql
   storage[(token << 32) | index]
   ```

The design assumes these two storage namespaces **never overlap**.

### What Actually Happens (Bug)

Because storage slots are computed via **bit manipulation instead of hashing**, the following equality can hold:

```ini
Counts(tokenA) slot == Snapshot(tokenB, 0) slot
```

This happens when:

```ini
tokenA = tokenB << 32
```

This is possible **iff** `tokenB` has at least **32 leading zero bits** (8 leading zero hex characters), so that the left shift still fits into a 160-bit Ethereum address.

Crucially:

* `tokenA` **does not need to be a real ERC-20 contract**
* The oracle never validates that a token address has deployed code
* Any arbitrary address can be used to initialize oracle state

As a result, initializing a pool for `tokenA` writes a `Counts` struct directly into the storage slot that previously held `Snapshot[0]` for `tokenB`.

### Concrete Walkthrough (Alice & Mallory)

#### Setup

* Victim token (`tokenB`):

  ```text
  0x0000000000000000000000000000000000000064
  ```

  (has 8 leading zero hex characters)

* Attacker computes:

  ```bash
  tokenA = tokenB << 32
         = 0x0000000000000000000000000000006400000000
  ```

#### Step 1 — Legitimate pool initialization (Alice)

Alice initializes an oracle pool for `tokenB`.

Oracle writes:

```pgsql
storage[tokenB]                = Counts(tokenB)
storage[(tokenB << 32) | 0]    = Snapshot(tokenB, 0)
```

Everything is correct so far.

#### Step 2 — Malicious pool initialization (Mallory)

Mallory initializes an oracle pool for `tokenA`.

Oracle writes:

```pgsql
storage[tokenA] = Counts(tokenA)
```

But:

```ini
tokenA == (tokenB << 32)
```

So this write **overwrites**:

```scss
Snapshot(tokenB, 0)
```

#### Step 3 — Resulting Corruption

The `Counts` struct overwrites the `Snapshot` struct **field-by-field**:

| Snapshot Field                  | Overwritten By                              |
| ------------------------------- | ------------------------------------------- |
| `timestamp`                     | `Counts.index` (usually `0`)                |
| `secondsPerLiquidityCumulative` | `Counts.count`, `capacity`, `lastTimestamp` |
| `tickCumulative`                | Zeroed                                      |

The oracle now contains a **synthetic, non-historical snapshot** that violates all oracle invariants.

### Why This Matters

This is not just "bad data" — it breaks **oracle logic assumptions**:

1. **Binary search breaks**
   `searchRangeForPrevious` assumes snapshots are monotonically increasing by timestamp. Overwriting a snapshot timestamp with `0` violates this invariant, causing incorrect snapshot selection or unexpected reverts.

2. **Wrong extrapolation base**
   `extrapolateSnapshot` may extrapolate from corrupted cumulative values, producing incorrect TWAPs, liquidity averages, and volatility metrics.

3. **Poisoned helper functions**
   Helpers that read `Snapshot[0]` (e.g., earliest timestamp, maximum observation period) return garbage values, misleading downstream consumers into querying unsafe historical ranges.

> Even though TWAPs use *differences* of cumulative values, injecting a corrupted snapshot changes **which snapshot is chosen** and **which linear segment is used**, so the error does not reliably cancel out.

## Vulnerable Code Reference

**1) Counts stored directly at `token`**

```solidity
Counts c;
assembly ("memory-safe") {
    c := sload(token)
}
```

**2) Snapshots stored at `(token << 32) | index`**

```solidity
assembly ("memory-safe") {
    last := sload(or(shl(32, token), index))
}
```

**3) No domain separation between metadata and snapshots**

There is no hashing or namespace separation between:

```
Counts(token)
Snapshot(token, index)
```

Both rely on raw address arithmetic.

## Recommended Mitigation

### 1️⃣ Use hashed storage slots (primary fix)

Derive snapshot slots using `keccak256`, matching Solidity mapping semantics:

```solidity
function getSnapshotSlot(address token, uint32 index) private pure returns (bytes32 slot) {
    assembly ("memory-safe") {
        mstore(0, token)
        mstore(32, index)
        slot := keccak256(0, 64)
    }
}
```

Replace all `(token << 32) | index` based `sload` / `sstore` with this slot.

### 2️⃣ Avoid raw-address storage keys for metadata

Similarly, store `Counts` under a hashed namespace:

```solidity
keccak256("oracle.counts", token)
```

This ensures future extensibility and collision safety.

### 3️⃣ Add invariants and tests

* Property test: initializing pools for distinct tokens must never mutate each other's oracle state.
* Fuzz test: random addresses with varying leading zeros must not cause collisions.

## Pattern Recognition Notes

* **Manual Storage Slot Construction**: Bit-packing storage keys without hashing is extremely error-prone in the EVM's flat storage model.
* **Address-as-Key Footguns**: Ethereum addresses are just numbers; shifting or masking them can easily create valid but unintended aliases.
* **Missing Domain Separation**: Different logical data (metadata vs. history) must never share the same numeric key space.
* **Oracle Invariant Fragility**: Oracle correctness depends not only on values, but on ordering and monotonicity assumptions.
* **Downstream Blast Radius**: Even if the core protocol does not consume the oracle for safety checks, external consumers may rely on it for pricing, risk, and liquidation logic.

## Quick Recall (TL;DR)

* **Bug**: Oracle uses `(token << 32) | index` for snapshots and `token` for metadata.
* **Attack**: Choose `tokenA = tokenB << 32` when `tokenB` has 8 leading zero hex characters.
* **Impact**: `Counts(tokenA)` overwrites `Snapshot(tokenB, 0)` → corrupted oracle history.
* **Fix**: Use `keccak256`-derived storage slots (mapping-style), never raw address arithmetic.
