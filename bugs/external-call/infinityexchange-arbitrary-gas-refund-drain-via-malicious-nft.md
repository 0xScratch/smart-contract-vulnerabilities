# Arbitrary WETH Drain via Attacker-Controlled Gas Consumption in NFT Transfers

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2022-06-infinity-findings/issues/74)
* **Affected Contract**: [InfinityExchange.sol](https://github.com/code-423n4/2022-06-infinity/blob/765376fa238bbccd8b1e2e12897c91098c7e5ac6/contracts/core/InfinityExchange.sol)
* **Vulnerability Type**: Improper Gas Accounting / Attacker-Controlled External Call / Arbitrary Fund Drain

## Summary

`InfinityExchange` implements a custom gas reimbursement mechanism for offchain order matching. During order execution, the protocol dynamically calculates the gas consumed and charges the buyer in WETH for the matcher's execution cost.

However, gas consumption during execution depends on external NFT contract behavior. Since NFT transfers invoke attacker-controlled contracts through `safeTransferFrom`, a malicious seller can deliberately deploy an ERC721 token that wastes excessive gas during transfer execution.

Because the exchange calculates reimbursement using:

```solidity
(startGasPerOrder - gasleft()) * tx.gasprice
```

the attacker can artificially inflate the computed gas cost. The protocol then pulls this inflated amount directly from the buyer's approved WETH balance via `safeTransferFrom`.

If the buyer has granted large or infinite WETH allowance to the exchange (a common pattern), the malicious NFT can cause the buyer to lose a massive amount of WETH far beyond the intended trade value.

## A Better Explanation (With Simplified Example)

### Intended Behavior

Infinity uses a `MATCH_EXECUTOR` that executes trades on behalf of users.

Flow:

1. Buyer signs an offchain buy order.
2. Seller signs a sell order.
3. `MATCH_EXECUTOR` matches them onchain.
4. Since the executor spends gas, the protocol reimburses execution cost from the buyer.

The protocol estimates reimbursement like this:

```solidity
uint256 gasCost =
    (startGasPerOrder - gasleft() + wethTransferGasUnits)
    * tx.gasprice;
```

Then pulls WETH from buyer:

```solidity
IERC20(weth).safeTransferFrom(
    buy.signer,
    address(this),
    protocolFee + gasCost
);
```

### What Actually Happens (Bug)

The protocol assumes gas usage is honest and predictable.

But NFT transfers execute arbitrary external contract logic:

```solidity
IERC721(item.collection).safeTransferFrom(from, to, tokenId);
```

A malicious NFT contract can intentionally consume enormous gas during transfer execution.

Since reimbursement is calculated *after* the external call using `gasleft()`, the attacker can artificially inflate:

```solidity
startGasPerOrder - gasleft()
```

This manipulated gas usage directly increases the amount of WETH charged to the buyer.

### Why This Matters

* Buyers commonly approve exchanges with infinite ERC20 allowance.
* The protocol imposes no user-defined cap on gas reimbursement.
* External malicious NFT contracts can manipulate gas accounting.
* Buyer may lose far more WETH than expected from a simple NFT purchase.

The dangerous architectural issue is:

> attacker-controlled execution cost is converted directly into token extraction.

### Concrete Walkthrough (Alice & Mallory)

#### Setup

* Alice approves the exchange for unlimited WETH:

```solidity
WETH.approve(exchange, type(uint256).max);
```

* Mallory deploys a malicious ERC721 contract.

#### Mallory Creates Malicious NFT Logic

Inside `safeTransferFrom`, Mallory adds extremely expensive logic:

```solidity
for (uint256 i = 0; i < 10000; i++) {
    uselessStorageReads();
}
```

or any other gas-burning computation.

#### Alice Purchases NFT

Alice unknowingly buys Mallory's NFT through Infinity.

The exchange executes:

```solidity
_transferMultipleNFTs(...)
```

which eventually calls:

```solidity
IERC721(item.collection).safeTransferFrom(...)
```

#### Malicious NFT Burns Massive Gas

Instead of normal transfer cost:

```text
~200k gas
```

the malicious contract consumes:

```text
millions of gas
```

#### Exchange Calculates Gas Refund

After transfer completes:

```solidity
uint256 gasCost =
    (startGasPerOrder - gasleft() + wethTransferGasUnits)
    * tx.gasprice;
```

Now `gasCost` becomes enormous because `gasleft()` was heavily reduced.

#### Buyer Gets Charged

The exchange then executes:

```solidity
IERC20(weth).safeTransferFrom(
    buy.signer,
    address(this),
    protocolFee + gasCost
);
```

Since Alice approved unlimited WETH, the protocol can pull an unexpectedly massive amount.

> The seller indirectly controls how much WETH the buyer pays by controlling gas consumption during NFT transfer execution.

### Analogy

Imagine a taxi meter where the passenger pays based on fuel usage after the ride.

Normally this is fine.

But now the driver secretly installs a machine that intentionally burns fuel during the trip. Since the passenger agreed to reimburse whatever fuel was used," the driver can massively inflate the bill.

The protocol made the same mistake:

> charging users based on attacker-influenced execution cost.

## Vulnerable Code Reference

**1) Gas reimbursement calculated from dynamic execution gas**

```solidity
uint256 gasCost =
    (startGasPerOrder - gasleft() + wethTransferGasUnits)
    * tx.gasprice;
```

This assumes gas usage is trustworthy.

**2) Attacker-controlled external NFT transfer**

```solidity
IERC721(item.collection).safeTransferFrom(
    from,
    to,
    item.tokens[i].tokenId
);
```

External NFT contracts can execute arbitrary expensive logic.

**3) Buyer charged arbitrary computed amount**

```solidity
IERC20(weth).safeTransferFrom(
    buy.signer,
    address(this),
    protocolFee + gasCost
);
```

No upper bound exists on `gasCost`.

**4) Vulnerable execution path**

```solidity
_transferMultipleNFTs(...)
    -> _transferNFTs(...)
        -> _transferERC721s(...)
            -> safeTransferFrom(...)
```

The gas reimbursement happens *after* this attacker-controlled call chain.

## Recommended Mitigation

### 1. Allow User-Specified Maximum Gas Refund (Primary Fix)

Require buyers to define a maximum acceptable reimbursement:

```solidity
require(gasCost <= maxGasRefund, "gas refund exceeded");
```

This prevents unbounded extraction.

### 2. Avoid Manual Gas Accounting

The safest design is:

* users execute their own orders,
* users naturally pay gas through Ethereum itself,
* protocol avoids reimbursement logic entirely.

Manual gas metering in Solidity is fragile and dangerous.

### 3. Never Trust Gas Usage After External Calls

Avoid patterns like:

```solidity
startGas - gasleft()
```

after attacker-controlled interactions.

External calls make gas usage adversarial.

### 4. Limit ERC20 Pulls

Avoid unlimited post-execution transfers derived from runtime computation.

Instead:

* pre-compute bounded fees,
* or escrow exact maximum values before execution.

### 5. Add Defensive Limits

For example:

```solidity
require(gasUsed <= MAX_ALLOWED_GAS);
```

to prevent extreme griefing.

## Pattern Recognition Notes

* **Attacker-Controlled Gas Accounting**: Any fee model derived from `gasleft()` after external calls becomes manipulable if attackers influence execution flow.
* **External Calls Are Adversarial**: NFT contracts are arbitrary code execution environments, not passive assets.
* **Infinite Approval Risk Amplification**: Unlimited ERC20 allowances magnify otherwise limited bugs into catastrophic fund drains.
* **Post-Execution Billing Is Dangerous**: Charging users after execution using dynamically computed values creates unbounded liability.
* **Protocol Reinventing Native Gas Economics**: Ethereum already provides a secure gas payment mechanism. Custom reimbursement systems frequently introduce accounting vulnerabilities.
* **Economic Manipulation via Resource Consumption**: Attackers can sometimes manipulate fees not by stealing directly, but by controlling resource usage that feeds into pricing/accounting formulas.

## Quick Recall (TL;DR)

* **Bug**: Protocol reimburses gas using dynamically measured execution cost.
* **Root Cause**: NFT transfers execute attacker-controlled external code that can intentionally waste gas.
* **Impact**: Buyer can be charged massive WETH amounts through inflated gas accounting.
* **Amplifier**: Infinite WETH approvals make draining feasible.
* **Fix**: Add max gas caps or avoid manual gas reimbursement entirely.
