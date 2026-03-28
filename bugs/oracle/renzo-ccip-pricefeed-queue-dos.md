# Global Price Feed DoS via CCIP Message Revert in Price Receiver

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2024-04-renzo-findings/issues/519)
* **Affected Contract**: [CCIPReceiver.sol](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/Bridge/L2/PriceFeed/CCIPReceiver.sol#L66)
* **Vulnerability Type**: Denial of Service (DoS) / Cross-Chain Queue Poisoning / Oracle Validation

## Summary

Renzo uses **Chainlink CCIP** to send **price updates from L1 → L2**. These updates are processed sequentially on the L2 receiver contract.

However, the receiver performs **strict validation checks** and **reverts** when price inputs are invalid (e.g., price deviation > 10%, invalid timestamp, zero price).

Because **Chainlink CCIP processes messages sequentially**, a single failed message can enter **manual execution mode**. After the **8-hour smart execution window**, all subsequent messages become **blocked** until the failed message succeeds.

Since invalid price messages **cannot succeed**, the queue becomes permanently stuck, resulting in **global price update DoS** across the Renzo protocol.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. L1 sends price updates through CCIP
2. L2 receiver validates price
3. If valid → update price
4. If invalid → revert

### Normal Flow

```text
Message 1 → Success
Message 2 → Success
Message 3 → Success
```

Everything works normally.

## What Actually Happens (Bug)

Renzo's receiver **reverts** on invalid input:

```solidity
if (_price == 0) revert InvalidZeroInput();

if (price deviation > 10%) revert InvalidOraclePrice();

if (_timestamp <= lastPriceTimestamp) revert InvalidTimestamp();

if (_timestamp > block.timestamp) revert InvalidTimestamp();
```

This creates a **problem with CCIP message sequencing**.

## Why This Matters (CCIP Behavior)

Chainlink CCIP processes messages **in order**.

If one message fails:

```text
Message 1 → Success
Message 2 → FAIL
Message 3 → BLOCKED
Message 4 → BLOCKED
```

After **8 hours**, CCIP requires **Message 2** to succeed first.

But Message 2 **will always fail** → permanent DoS.

## Concrete Walkthrough (Alice & Oracle)

### Step 1 — Normal Operation

```yaml
Last Price = 100
```

L1 sends:

```yaml
Price = 105
```

Valid (<10% change) → accepted

### Step 2 — Invalid Price Sent

L1 sends:

```yaml
Price = 150
```

Deviation:

```yaml
150 - 105 = 45 (>10%)
```

Contract reverts:

```solidity
revert InvalidOraclePrice();
```

Message becomes **failed**.

### Step 3 — Queue Gets Stuck

Now:

```text
Message 2 → Failed
Message 3 → Blocked
Message 4 → Blocked
```

After 8 hours:

* CCIP requires Message 2 to succeed first
* But Message 2 **always fails**
* Queue permanently stuck

## Why This Is Dangerous

This breaks:

* ❌ Price updates
* ❌ Exchange rate calculations
* ❌ Minting logic
* ❌ Withdrawal calculations
* ❌ Protocol accounting

Essentially:

**Renzo L2 price system becomes frozen**

## Vulnerable Code Reference

### 1) CCIP Receiver Reverting on Validation

```solidity
function _ccipReceive(
    Client.Any2EVMMessage memory any2EvmMessage
) internal override whenNotPaused {
    address _ccipSender = abi.decode(any2EvmMessage.sender, (address));

    if (_ccipSender != xRenzoBridgeL1)
        revert InvalidSender(xRenzoBridgeL1, _ccipSender);

    if (_ccipSourceChainSelector != ccipEthChainSelector)
        revert InvalidSourceChain(ccipEthChainSelector, _ccipSourceChainSelector);

    (uint256 _price, uint256 _timestamp) =
        abi.decode(any2EvmMessage.data, (uint256, uint256));

    xRenzoDeposit.updatePrice(_price, _timestamp);
}
```

### 2) Strict Price Validation Causing Permanent Reverts

```solidity
function _updatePrice(uint256 _price, uint256 _timestamp) internal {

    if (_price == 0) {
        revert InvalidZeroInput();
    }

    if (
        (_price > lastPrice && (_price - lastPrice) > (lastPrice / 10)) ||
        (_price < lastPrice && (lastPrice - _price) > (lastPrice / 10))
    ) {
        revert InvalidOraclePrice();
    }

    if (_timestamp <= lastPriceTimestamp) {
        revert InvalidTimestamp(_timestamp);
    }

    if (_timestamp > block.timestamp) {
        revert InvalidTimestamp(_timestamp);
    }

    lastPrice = _price;
    lastPriceTimestamp = _timestamp;
}
```

## Recommended Mitigation

### 1) Handle Invalid Messages Gracefully (Primary Fix)

Instead of reverting:

```solidity
revert InvalidOraclePrice();
```

Use:

```solidity
emit InvalidPriceIgnored(_price);
return;
```

This prevents queue blockage.

### 2) Try/Catch Pattern in Receiver

```solidity
try xRenzoDeposit.updatePrice(_price, _timestamp) {
} catch {
    emit PriceUpdateFailed(_price, _timestamp);
}
```

This ensures:

* Message does not revert
* Queue continues processing

### 3) Add Fallback Price Handling

Instead of reverting:

* Ignore invalid prices
* Keep last known good price

### 4) Add Monitoring

Emit events for:

* Invalid price
* Timestamp issues
* Failed message execution

This improves observability.

## Pattern Recognition Notes

### Cross-Chain Queue Poisoning

If:

* Messages processed sequentially
* Receiver reverts

Then:

**Single bad message can freeze entire system**

### Never Revert in Cross-Chain Receivers

Cross-chain receivers should:

* Never revert
* Handle errors gracefully
* Emit events instead

### Oracle Validation DoS

Strict validation logic:

```solidity
revert InvalidOraclePrice();
```

Can cause:

* System halt
* Data pipeline freeze

Better approach:

* Ignore invalid data
* Continue processing

### Cross-Chain Design Rule

> "Cross-chain receivers should be fail-safe, not fail-closed."

## Quick Recall (TL;DR)

* **Bug**: Receiver reverts on invalid price
* **Impact**: Failed CCIP message blocks queue
* **Result**: Global price update DoS
* **Fix**: Never revert — handle gracefully
