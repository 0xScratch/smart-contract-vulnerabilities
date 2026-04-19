# Missing Owner Initialization via Incorrect Ownable Usage in Upgradeable Contract

* **Severity**: High
* **Source**: [Code4rena](https://github.com/code-423n4/2021-10-covalent-findings/issues/45)
* **Affected Contract**: [DelegatedStaking.sol](https://github.com/code-423n4/2021-10-covalent/blob/ded3aeb2476da553e8bb1fe43358b73334434737/contracts/DelegatedStaking.sol#L62-L63)
* **Vulnerability Type**: Misconfiguration / Access Control / Upgradeable Contract Initialization Bug

## Summary

The `DelegatedStaking` contract is designed to be deployed using a proxy (upgradeable pattern), but it incorrectly inherits from Ownable instead of OwnableUpgradeable.

Because proxy-based deployments **do not execute constructors**, the `owner` variable is never initialized. Additionally, the `initialize()` function does not set the owner manually.

As a result, all `onlyOwner` functions become permanently inaccessible, effectively locking administrative control of the contract.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. Contract is deployed via proxy.
2. `initialize()` acts as a constructor replacement.
3. Ownership should be set to the deployer/admin.
4. Admin can call `onlyOwner` functions.

### What Actually Happens (Bug)

* The contract inherits from:

  * Ownable

* This contract relies on a constructor:

  ```solidity
  constructor() {
      owner = msg.sender;
  }
  ```

* But in proxy deployments:

  * ❌ Constructor is never executed
  * ❌ `owner` is never set

* The `initialize()` function:

  ```solidity
  function initialize(uint128 minStakedRequired) public initializer {
      // no owner initialization
  }
  ```

👉 So after deployment:

```solidity
owner = address(0);
```

### Why This Matters

* All `onlyOwner` functions check:

  ```solidity
  require(msg.sender == owner);
  ```
* Since `owner = address(0)`:

  * No real user can satisfy this condition
  * Admin functions become permanently unusable

👉 This leads to:

* Loss of protocol control
* Inability to upgrade/configure contract
* Potential permanent system mismanagement

### Concrete Walkthrough (Deployer Scenario)

* **Deployment**:

  * Proxy deploys contract
  * Constructor is skipped
  * `owner = address(0)`

* **Initialization**:

  * Deployer calls `initialize()`
  * No ownership logic inside → still `address(0)`

* **Admin Action Attempt**:

  ```solidity
  onlyOwner function called
  ```

  → Fails because:

  ```solidity
  msg.sender != address(0)
  ```

👉 Result:
**All admin functionality is permanently locked**

> **Analogy**:
> Imagine a system where the "admin role" is supposed to be assigned during setup. But the setup script never runs, and no backup step assigns the admin. Now the system has admin-only controls — but no admin exists to use them.

## Vulnerable Code Reference

**1) Incorrect Ownable import**

```solidity
import "@openzeppelin/contracts/access/Ownable.sol";
```

**2) Missing constructor execution (implicit issue)**

```solidity
// constructor never runs in proxy deployment
```

**3) Missing owner initialization in `initialize()`**

```solidity
function initialize(uint128 minStakedRequired) public initializer {
    // Missing: owner initialization
}
```

## Recommended Mitigation

1. **Use upgradeable ownership module**

Replace:

* Ownable

With:

* OwnableUpgradeable
* Initializable

2. **Initialize ownership properly**

```solidity
function initialize(uint128 minStakedRequired) public initializer {
    __Ownable_init(); // sets owner = msg.sender
}
```

3. **General upgradeable contract rule**

* Never rely on constructors
* Always move constructor logic into `initialize()`
* Always call parent initializers (`__Ownable_init`, etc.)

4. **Add tests for ownership**

* Verify `owner != address(0)` after initialization
* Ensure `onlyOwner` functions are callable

## Pattern Recognition Notes

* **Constructor Dependency in Proxy Systems**
  Contracts relying on constructors will silently fail in upgradeable deployments.

* **Initialization Omissions**
  Missing initialization logic can leave critical variables (like `owner`) unset.

* **Access Control Lockout**
  Misconfigured ownership can permanently disable admin functionality.

* **Library Mismatch**
  Mixing standard and upgradeable libraries (e.g., `Ownable` vs `OwnableUpgradeable`) is a common source of bugs.

* **Silent Failures**
  These issues don't revert during deployment — they manifest later as broken functionality.

## Quick Recall (TL;DR)

* **Bug**: Used `Ownable` (constructor-based) in a proxy contract
* **Root Cause**: Constructor never runs + no manual initialization
* **Impact**: `owner = address(0)` → all `onlyOwner` functions unusable
* **Fix**: Use `OwnableUpgradeable` and call `__Ownable_init()` in `initialize()`
