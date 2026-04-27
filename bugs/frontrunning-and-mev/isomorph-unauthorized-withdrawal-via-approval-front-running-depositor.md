# Unauthorized Withdrawal via Approval Front-Running in Depositor

* **Severity**: High
* **Source**: [Sherlock](https://github.com/sherlock-audit/2022-11-isomorph-judging/issues/47)
* **Affected Contract**: [Depositor.sol](https://github.com/sherlock-audit/2022-11-isomorph/blob/main/contracts/Velo-Deposit-Tokens/contracts/Depositor.sol#L119-L127)
* **Vulnerability Type**: Access Control / Front-Running / Approval Exploit

## Summary

`Depositor.withdrawFromGauge` is a **public function** that allows anyone to trigger withdrawal of funds tied to a deposit NFT. The function burns the NFT (via `depositReceipt.burn`) and transfers the withdrawn tokens to `msg.sender`.

Although the burn operation checks whether the caller is **approved or owner**, the check is performed against the **Depositor contract**, not the user. Since users must **approve the Depositor contract in a separate transaction before withdrawing**, this creates a **front-running window**.

An attacker can monitor approvals and call `withdrawFromGauge` before the legitimate user, causing the contract to burn the user's NFT and transfer funds to the attacker.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. **Deposit**:

   * User deposits tokens into the protocol
   * Receives an NFT representing ownership (deposit receipt)

2. **Withdraw**:

   * User approves Depositor contract for their NFT
   * Calls `withdrawFromGauge`
   * Contract:

     * Burns NFT
     * Withdraws funds
     * Sends tokens back to user

### What Actually Happens (Bug)

* `withdrawFromGauge` is **public** → anyone can call it

* Tokens are sent to:

  ```solidity
  msg.sender
  ```

  instead of the NFT owner

* NFT burn check:

  ```solidity
  require(_isApprovedOrOwner(msg.sender, _NFTId))
  ```

  is evaluated with:

  * `msg.sender = Depositor contract`

* Once user approves the contract:

  * Contract becomes authorized to burn NFT
  * But **ANY external caller can trigger the flow**

### Why This Matters

* Approval and withdrawal are split into **two transactions**
* This creates a **race condition (front-running window)**
* Bots can easily monitor approvals and exploit instantly

### Concrete Walkthrough (Alice & Mallory)

* **Setup**: Alice owns NFT representing 100 tokens

#### Step 1: Alice prepares withdrawal

```solidity
approve(Depositor, NFT_ID)
```

* Depositor contract is now authorized

#### Step 2: Mallory (attacker bot)

* Observes approval in mempool
* Calls:

```solidity
withdrawFromGauge(NFT_ID)
```

#### Step 3: Contract execution

1. `depositReceipt.burn(NFT_ID)`

   * Passes because Depositor is approved

2. Funds are withdrawn

3. Transfer happens:

```solidity
AMMToken.transfer(msg.sender, amount);
```

👉 `msg.sender = Mallory`

### 💥 Result

* Alice's NFT → **burned** ❌
* Alice receives → **nothing** ❌
* Mallory receives → **all funds** ✅

> **Analogy**:
> You authorize a bank to access your locker.
> Before you reach the counter, someone else says "withdraw it for me."
> The bank complies because it already has permission.

## Vulnerable Code Reference

### 1) Public withdrawal function with no ownership check

```solidity
function withdrawFromGauge(uint256 _NFTId, address[] memory _tokens) public {
    uint256 amount = depositReceipt.pooledTokens(_NFTId);
    depositReceipt.burn(_NFTId);
    gauge.getReward(address(this), _tokens);
    gauge.withdraw(amount);
    AMMToken.transfer(msg.sender, amount);
}
```

### 2) Burn relies on contract approval, not caller ownership

```solidity
function burn(uint256 _NFTId) external onlyMinter {
    require(_isApprovedOrOwner(msg.sender, _NFTId), "ERC721: caller is not token owner or approved");
    _burn(_NFTId);
}
```

### 3) Missing ownership validation in withdraw

```solidity
// Missing:
require(depositReceipt.ownerOf(_NFTId) == msg.sender);
```

## Recommended Mitigation

### 1. Enforce ownership in withdrawal (primary fix)

```solidity
require(depositReceipt.ownerOf(_NFTId) == msg.sender);
```

### 2. Avoid two-step approval + action pattern

* Combine approval + withdrawal in a single transaction if possible
* Or use `safeTransferFrom` to move NFT first

### 3. Follow pull-based design carefully

* If using `msg.sender` as recipient → must enforce strict access control

### 4. Add front-running resistant patterns

* Consider signed approvals (EIP-712 style)
* Or commit-reveal mechanisms for sensitive actions

## Pattern Recognition Notes

* **Approval Front-Running**:
  Any workflow where:

  * Step 1 = approve
  * Step 2 = execute
    → is vulnerable unless tightly controlled

* **Incorrect Recipient (msg.sender misuse)**:
  If funds are sent to `msg.sender`, always verify:

  > "Is this the rightful owner?"

* **Access Control Missing on Critical Functions**:
  Public functions that move funds must always enforce ownership

* **Contract vs User Identity Confusion**:
  Internal calls may pass checks (contract is approved), but external caller is different

* **Race Condition via Mempool Visibility**:
  Anything observable (like approvals) can be exploited before completion

## Quick Recall (TL;DR)

* **Bug**: Public withdraw sends funds to `msg.sender` without ownership check
* **Exploit**: Attacker front-runs after approval
* **Impact**: Full theft of user funds
* **Fix**: Restrict withdrawal to NFT owner
