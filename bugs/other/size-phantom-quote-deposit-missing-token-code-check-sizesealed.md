# Phantom Quote Deposit via Missing Token Code Check in SizeSealed

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2022-11-size-findings/issues/48)
* **Affected Contract**: [SizeSealed.sol](https://github.com/code-423n4/2022-11-size/blob/main/src/SizeSealed.sol#L163)
* **Vulnerability Type**: Asset Validation / Phantom Deposit / Trust Assumption

## Summary

`SizeSealed.bid()` transfers the bidder's `quoteToken` into the contract using Solmate's `SafeTransferLib`. However, the library **does not verify that the token address contains contract code**. If the provided `quoteToken` address has no deployed contract, the transfer call can succeed while transferring **zero tokens**.

Because the auction records the bid as valid without verifying the actual token balance increase, an attacker can submit **fake funded bids**. Later, when honest users deposit real tokens, the attacker can withdraw or cancel and receive refunds backed by other users' funds, resulting in theft.

This is a classic **phantom deposit / empty-token honeypot** vulnerability similar in class to the Qubit Finance exploit.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. **Bid**: Bidder submits `quoteAmount`.
2. Contract pulls tokens via `safeTransferFrom`.
3. Contract assumes tokens were received.
4. Bid is considered fully funded.

### What Actually Happens (Bug)

* `SafeTransferLib` does **not check `address(token).code.length > 0`**.
* If the token is not deployed yet:
  * The call can succeed,
  * but **no tokens move**.
* The contract still records the bid as funded.

This creates **fake collateralized bids**.

### Why This Matters

* Auction accounting assumes every bid is backed by real tokens.
* A malicious bidder can create **zero-cost bids**.
* Later honest deposits can be drained via refunds or settlement.
* The attack is especially realistic due to **cross-chain deterministic token addresses**.

## Concrete Walkthrough (Alice & Bob)

### Setup

* Token **Q** exists on another chain but **not yet deployed locally**.
* Its future address is predictable.
* Alice controls two wallets: Alice1 (seller) and Alice2 (bidder).

### Step-by-Step Attack

1. **Alice creates auction**

    Alice1 creates an auction with:

    ```text
    quoteToken = Q (not deployed yet)
    ```


    No validation fails.

2. **Alice submits fake bid**

    Alice2 calls:

    ```solidity
    bid(quoteAmount = 1000e18)
    ````

    Inside:

    ```solidity
    SafeTransferLib.safeTransferFrom(
        ERC20(a.params.quoteToken),
        msg.sender,
        address(this),
        quoteAmount
    );
    ```

    Because token Q has **no code**:

    * transfer call returns success ✅
    * contract receives **0 tokens** ❌
    * bid recorded as funded ❌❌

3. **Token Q later launches**

    Now Q is deployed at the same address.

4. **Bob submits real bid**

    Bob bids:

    ```text
    1001e18 Q
    ```

    Now the transfer is real.

    Contract balance:

    ```text
    = 1001 Q
    ```

    But accounting believes:

    ```text
    Alice: 1000
    Bob:   1001
    Total: 2001
    ```

5. **Alice cancels her bid**

    Alice2 calls `cancelBid()`.

    Contract refunds:

    ```text
    1000 Q
    ```

    Those tokens come from **Bob's real deposit**.

6. **Alice cancels auction**

    Alice1 retrieves her base tokens.

### Final State

**Alice gains:**

* stolen quote tokens from Bob
* her original base tokens back

**Bob loses:**

* real quote tokens
* no auction outcome

> **Analogy**: The system accepts a "bank transfer receipt" without checking the bank exists. Later, when real money arrives from honest users, the attacker cashes out using the fake receipt.

## Vulnerable Code Reference

### 1) Unsafe token transfer in `bid()`

```solidity
SafeTransferLib.safeTransferFrom(
    ERC20(a.params.quoteToken),
    msg.sender,
    address(this),
    quoteAmount
);
```

❌ No check that `quoteToken` is a deployed contract

❌ No balance-delta verification

❌ Assumes transfer success implies funds received

### 2) Solmate warning (root cause)

Solmate explicitly documents:

```text
/// Note: none of the functions check that a token has code.
```

The caller (SizeSealed) must enforce this — but does not.

## Recommended Mitigation

### 1. Verify token has code (primary fix)

Before accepting bids:

```solidity
if (address(a.params.quoteToken).code.length == 0) {
    revert InvalidToken();
}
```

This prevents phantom tokens entirely.

### 2. Balance-delta verification (strong defense)

Mirror the safety pattern used in `createAuction()`:

```solidity
uint256 beforeBal = token.balanceOf(address(this));
safeTransferFrom(...);
uint256 afterBal = token.balanceOf(address(this));

if (afterBal - beforeBal != quoteAmount) {
    revert UnexpectedBalanceChange();
}
```

This protects against:

* non-standard tokens
* fee-on-transfer tokens
* phantom tokens

### 3. Consider token allowlisting (defense-in-depth)

If protocol design permits:

* whitelist supported quote tokens
* avoid arbitrary token addresses

This removes the entire attack surface.

### 4. Prefer hardened transfer wrappers

If using Solmate:

* always pair with code-length checks
* or migrate to OZ SafeERC20 + explicit validations

## Pattern Recognition Notes

* **Phantom Deposit Pattern**: Any system that trusts token transfers without verifying balance changes is vulnerable.
* **Missing Code Existence Check**: Calling ERC20 functions on EOAs or undeployed addresses can silently succeed.
* **Cross-Chain Address Assumptions**: Deterministic token addresses across chains make this attack practical in the wild.
* **Library Trust Misuse**: Minimal libraries (like Solmate) shift safety responsibility to the caller — forgetting this creates critical bugs.
* **Ingress Validation Failure**: Always validate asset reality at the moment funds enter the system.
* **Economic Invariant Violation**: Protocol assumes "recorded deposits == real assets," which must always be enforced on-chain.

### Quick Recall (TL;DR)

* **Bug**: `bid()` trusts `SafeTransferLib` without verifying token contract exists.
* **Impact**: Attacker can create fake funded bids and later steal real users' quote tokens.
* **Fix**: Check `token.code.length > 0` and verify balance delta after transfer.
