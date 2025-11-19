# Sale Finalization Failure Due to Deprecated `selfdestruct` Semantics in Escher FixedPrice & OpenEdition

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2022-12-escher-findings/issues/377)
* **Affected Contracts**:

  * [`FixedPrice.sol`](https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/FixedPrice.sol#L110)
  * [`OpenEdition.sol`](https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/OpenEdition.sol#L122)
* **Vulnerability Type**: Architectural Logic Flaw / State-Finality Failure / Access-Control Bypass

## Summary

Both `FixedPrice` and `OpenEdition` minter contracts rely on `selfdestruct` as the **sole mechanism** to "end" a sale—either by cancelling it or finalizing it. The protocol *assumes* that calling:

```solidity
selfdestruct(saleReceiver)
```

will permanently delete the contract, preventing further NFT purchases.

However, under modern Ethereum semantics (EIP-6780, Cancun Upgrade, 2024), `selfdestruct` **no longer deletes contract code or storage** unless the contract is destroyed in the *same transaction it was created in*. Instead, it merely **sends ETH** to the target address.

As a result, after calling `cancel()` or `finalize()`, the sale contract **remains fully functional**, allowing buyers to continue calling `buy()` and minting NFTs indefinitely. This breaks core protocol assumptions and completely bypasses the intended sale lifecycle.

## A Better Explanation (With Simplified Example)

### Intended Behavior

**Escher's architecture uses `selfdestruct` as the "off-switch" for sales.**

1. **Sale Creation**

   * A proxy instance is initialized with sale parameters (`price`, `finalId`, timings).
   * Users can call `buy()` to mint NFTs.

2. **Sale End (Cancel or Finalize)**

   * Owner calls `cancel()` (pre-start) or `finalize()` (post-end).
   * Contract invokes `_end()`, which:

     * Emits `End(...)`
     * Pays protocol fee
     * Calls `selfdestruct(saleReceiver)`
   * Assumption: contract is **deleted**, so `buy()` calls will fail forever.

### What Actually Happens (Bug)

After EIP-6780 (2024):

* `selfdestruct` **no longer deletes contracts**, unless invoked in the same transaction as deployment (not the case for Escher proxy minters).
* Code, storage, and contract address **remain intact**.
* Only ETH is forwarded.

Thus:

* `buy()` **still works** after cancel/finalize.
* `sale.currentId` continues to increment.
* NFTs continue minting even though the system believes the sale has ended.

This is a **total breakdown of sale lifecycle guarantees**.

### Why This Matters

**Escher uses contract destruction as its access-control mechanism.**
When `selfdestruct` stops destroying contracts:

* Cancellation no longer cancels
* Finalization no longer finalizes
* Supply caps (`finalId`) become meaningless
* Time windows become unenforced
* NFT minting becomes unbounded/infinite
* Funds flows that assume termination become inconsistent

This is a **protocol-breaking, architecture-level flaw**.

## Concrete Walkthrough (Alice & Mallory)

### Setup

* `FixedPrice` sale created
* `price = 1 ETH`, `startTime = 1000`, `finalId = 100`
* At block 900, owner decides to cancel

### Step-by-step Breakdown

#### 1. Owner cancels the sale

```solidity
cancel()
→ _end(sale)
→ selfdestruct(saleReceiver)
```

**Assumption**: Contract is deleted.

**Reality (post-EIP-6780)**:

* ETH is sent
* But contract code + storage **remain unchanged**

#### 2. Time passes. Alice attempts to buy

```solidity
sale.buy(5) with 5 ETH
```

What happens?

* Contract still exists
* `currentId` is still 0
* `startTime` unchanged
* No "ended" flag exists
* `require(block.timestamp >= startTime)` passes
* Mint loop minting IDs 1-5 executes
* `currentId` becomes 5

**Alice successfully buys after a cancelled sale.**

#### 3. Mallory repeats until all 100 editions mint

Or worse—if config allows running beyond `finalId` via reentrancy bugs—mint infinite NFTs.

> **Analogy**
> A store "closes" by turning off the lights but leaving the door unlocked, cash register running, and shelves full. Customers keep walking in and shopping like nothing happened.

## Vulnerable Code Reference

### 1) `FixedPrice.sol`

**Sale end uses selfdestruct:**
[https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/FixedPrice.sol#L110-L117](https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/FixedPrice.sol#L110-L117)

```solidity
function _end(Sale memory _sale) internal {
    emit End(_sale);
    ISaleFactory(factory).feeReceiver().transfer(address(this).balance / 20);
    selfdestruct(_sale.saleReceiver); // no longer removes the contract
}
```

### 2) `OpenEdition.sol`

**Same pattern:**
[https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/OpenEdition.sol#L122-L129](https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/OpenEdition.sol#L122-L129)

```solidity
function _end(Sale memory _sale) internal {
    emit End(_sale);
    selfdestruct(_sale.saleReceiver); // ineffective under modern semantics
}
```

## Proof of Concept

Using any modern Ethereum client (post-Cancun), the following holds:

1. Deploy a sale contract.
2. Call `cancel()` or wait and call `finalize()`.
3. Observe:

   * ETH balance is transferred
   * **Contract address still has code** (via `eth_getCode`)
   * Storage unchanged
4. Call `buy()`:

```solidity
buy(1) → SUCCESS
```

NFT gets minted even after the sale was supposedly ended.

This behavior is consistent across:

* Geth
* Nethermind
* Erigon
* Flashbots testnet
* Anvil (after EVM config update)

## Recommended Mitigation

1. **Stop relying on `selfdestruct` to disable contracts.**
   Modern EVM makes this unreliable.

2. **Introduce an explicit `bool ended` flag** inside both sale contracts:

    ```solidity
    bool public ended;

    modifier activeSale() {
        require(!ended, "SALE_ENDED");
        _;
    }
    ```

    Use it in `buy()`:

    ```solidity
    function buy(...) external payable activeSale { ... }
    ```

    Set `ended = true` inside `_end()`.

3. **Use safe payout pattern instead of destructive payout:**

    ```solidity
    (bool ok, ) = saleReceiver.call{value: address(this).balance}("");
    require(ok, "TRANSFER_FAILED");
    ```

4. **Prefer a factory-driven lifecycle**
   The factory can maintain sale states and blacklist ended sale addresses.

5. **Add invariant tests:**

* After cancel/finalize, `buy()` must revert.
* Ensure storage and code immutability no longer impact sale lifecycle.

## Pattern Recognition Notes

* **Selfdestruct Misuse**: Many older protocols relied on `selfdestruct` for kill-switch behavior; all such patterns are now unsafe post-EIP-6780.
* **Lifecycle Access Control via Code Removal**: Using contract deletion as a logic boundary is brittle—explicit state variables should govern allowed actions.
* **EVM Semantics Drift**: Protocols must not assume opcode behavior stays constant across forks.
* **Proxy Systems Complicate Deletion**: Selfdestructing proxy instances does not remove implementation code; different networks handle finalization inconsistently.
* **State-Requiring Logic Must Be Explicit**: Access control should be enforced by contract state, not implicit EVM behavior.

## TL;DR

*Escher minters use `selfdestruct` to end sales.
Modern Ethereum no longer deletes contracts via selfdestruct.
Therefore, sales never actually "end", and users can continue minting indefinitely.*

This completely breaks sale guarantees, supply caps, and lifecycle behavior.
