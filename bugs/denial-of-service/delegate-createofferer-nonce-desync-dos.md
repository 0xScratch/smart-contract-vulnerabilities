# Nonce Desynchronization Leading to Denial of Service in `CreateOfferer.sol`

- **Severity**: Medium
- **Source**: [Code4rena](https://github.com/code-423n4/2023-09-delegate-findings/issues/94) / [One Bug Per Day](https://www.onebugperday.com/v1/541)
- **Affected Contract**: [CreateOfferer.sol](https://github.com/code-423n4/2023-09-delegate/blob/a6dbac8068760ee4fc5bababb57e3fe79e5eeb2e/src/libraries/CreateOffererLib.sol)
- **Vulnerability Type**: Denial of Service / Integration Error

## Original Bug Description

>## Lines of code
>
>[https://github.com/code-423n4/2023-09-delegate/blob/a6dbac8068760ee4fc5bababb57e3fe79e5eeb2e/src/libraries/CreateOffererLib.sol#L184](https://github.com/code-423n4/2023-09-delegate/blob/a6dbac8068760ee4fc5bababb57e3fe79e5eeb2e/src/libraries/CreateOffererLib.sol#L184)
>
>## Vulnerability details
>
>## Impact
>
>CreateOfferer.sol should not enforce the nonce incremented sequentially, otherwise user can DOS the contract by skipping order
>
>## Proof of Concept
>
>According to
>
>[https://github.com/ProjectOpenSea/seaport/blob/main/docs/SeaportDocumentation.md#contract-orders](https://github.com/ProjectOpenSea/seaport/blob/main/docs/SeaportDocumentation.md#contract-orders)
>
>>Seaport v1.2 introduced support for a new type of order: the contract order. In brief, a smart contract that implements the ContractOffererInterface (referred to as an "Seaport app contract" or "Seaport app" in the docs and a "contract offerer in the code) can now provide a dynamically generated order (a contract order) in response to a buyer or seller's contract order request.
>
>the CreateOfferer.sol aims to be comply with the interface ContractOffererInterface
>
>the life cycle of the contract life cycle here is here:
>
>[https://github.com/ProjectOpenSea/seaport/blob/main/docs/SeaportDocumentation.md#example-lifecycle-journey](https://github.com/ProjectOpenSea/seaport/blob/main/docs/SeaportDocumentation.md#example-lifecycle-journey)
>
>first the function _getGeneratedOrder is called,
>
>then after the order execution, the function ratifyOrder is triggered for contract (CreateOfferer.sol) to do post order validation
>
>In the logic of ratifyOrder, the nonce is incremented by calling this [line of code](https://github.com/code-423n4/2023-09-delegate/blob/a6dbac8068760ee4fc5bababb57e3fe79e5eeb2e/src/CreateOfferer.sol#L77) Helpers.processNonce
>
>```solidity
>function ratifyOrder(SpentItem[] calldata offer, ReceivedItem[] calldata consideration, bytes calldata context, bytes32[] calldata, uint256 contractNonce)
>  external
>  checkStage(Enums.Stage.ratify, Enums.Stage.generate)
>  onlySeaport(msg.sender)
>  returns (bytes4)
>{
>  Helpers.processNonce(nonce, contractNonce);
>  Helpers.verifyCreate(delegateToken, offer[0].identifier, transientState.receivers, consideration[0], context);
>  return this.ratifyOrder.selector;
>}
>```
>
>this is calling this [line of code](https://github.com/code-423n4/2023-09-delegate/blob/a6dbac8068760ee4fc5bababb57e3fe79e5eeb2e/src/libraries/CreateOffererLib.sol#L184)
>
>```solidity
>function processNonce(CreateOffererStructs.Nonce storage nonce, uint256 contractNonce) internal {
>  if (nonce.value != contractNonce) revert CreateOffererErrors.InvalidContractNonce(nonce.value, contractNonce);
>  unchecked {
>     ++nonce.value;
>  } // Infeasible this will overflow if starting point is zero
>}
>```
>
>the CreateOffererStructs.Nonce data structure is just [nonce](https://github.com/code-423n4/2023-09-delegate/blob/a6dbac8068760ee4fc5bababb57e3fe79e5eeb2e/src/libraries/CreateOffererLib.sol#L72)
>
>```solidity
>library CreateOffererStructs {
>    /// @notice Used to track the stage and lock status
>    struct Stage {
>        CreateOffererEnums.Stage flag;
>        CreateOffererEnums.Lock lock;
>    }
>
>    /// @notice Used to keep track of the seaport contract nonce of CreateOfferer
>    struct Nonce {
>        uint256 value;
>    }
>```
>
>but this is not how seaport contract track contract nonce
>
>on seaport contract, we are calling [_getGeneratedOrder](https://github.com/ProjectOpenSea/seaport/blob/539f0c18af85152aff9d64d90a55cf1627fd3e25/reference/lib/ReferenceOrderValidator.sol#L323)
>
>which calls [_callGenerateOrder](https://github.com/ProjectOpenSea/seaport/blob/539f0c18af85152aff9d64d90a55cf1627fd3e25/reference/lib/ReferenceOrderValidator.sol#L360)
>
>```solidity
>{
>  // Do a low-level call to get success status and any return data.
>  (bool success, bytes memory returnData) = _callGenerateOrder(
>     orderParameters,
>     context,
>     originalOfferItems,
>     originalConsiderationItems
>  );
>
>  {
>     // Increment contract nonce and use it to derive order hash.
>     // Note: nonce will be incremented even for skipped orders, and
>     // even if generateOrder's return data doesn't meet constraints.
>     uint256 contractNonce = (
>         _contractNonces[orderParameters.offerer]++
>     );
>
>     // Derive order hash from contract nonce and offerer address.
>     orderHash = bytes32(
>         contractNonce ^
>             (uint256(uint160(orderParameters.offerer)) << 96)
>     );
>  }
>```
>
>as we can see
>
>```solidity
>// Increment contract nonce and use it to derive order hash.
>// Note: nonce will be incremented even for skipped orders, and
>// even if generateOrder's return data doesn't meet constraints.
>uint256 contractNonce = (
>  _contractNonces[orderParameters.offerer]++
>);
>```
>
>nonce will be incremented even for skipped orders
>
>this is very important, suppose the low level call _callGenerateOrder return false, we are hitting the [else block](https://github.com/ProjectOpenSea/seaport/blob/539f0c18af85152aff9d64d90a55cf1627fd3e25/reference/lib/ReferenceOrderValidator.sol#L401)
>
>```solidity
> return _revertOrReturnEmpty(revertOnInvalid, orderHash);
>```
>
>this is calling _revertOrReturnEmpty
>
>```solidity
>function _revertOrReturnEmpty(
>  bool revertOnInvalid,
>  bytes32 contractOrderHash
>)
>  internal
>  pure
>  returns (
>     bytes32 orderHash,
>     uint256 numerator,
>     uint256 denominator,
>     OrderToExecute memory emptyOrder
>  )
>{
>  // If invalid input should not revert...
>  if (!revertOnInvalid) {
>     // Return the contract order hash and zero values for the numerator
>     // and denominator.
>     return (contractOrderHash, 0, 0, emptyOrder);
>  }
>
>  // Otherwise, revert.
>  revert InvalidContractOrder(contractOrderHash);
>}
>```
>
>clearly we can see that if the flag revertOnInvalid is set to false, then even the low level call return false, the nonce of the offerer is still incremented
>
>Where in the Seaport code does it set revertOnInvalid to false?
>
>When the seaport wants to combine multiple orders in this [line of code](https://github.com/ProjectOpenSea/seaport-core/blob/d4e8c74adc472b311ab64b5c9f9757b5bba57a15/src/lib/OrderCombiner.sol#L137)
>
>```solidity
>    function _fulfillAvailableAdvancedOrders(
>        AdvancedOrder[] memory advancedOrders,
>        CriteriaResolver[] memory criteriaResolvers,
>        FulfillmentComponent[][] memory offerFulfillments,
>        FulfillmentComponent[][] memory considerationFulfillments,
>        bytes32 fulfillerConduitKey,
>        address recipient,
>        uint256 maximumFulfilled
>    ) internal returns (bool[] memory, /* availableOrders */ Execution[] memory /* executions */ ) {
>        // Validate orders, apply amounts, & determine if they use conduits.
>        (bytes32[] memory orderHashes, bool containsNonOpen) = _validateOrdersAndPrepareToFulfill(
>            advancedOrders,
>            criteriaResolvers,
>            false, // Signifies that invalid orders should NOT revert.
>            maximumFulfilled,
>            recipient
>        );
>```
>
>we [calls](https://github.com/ProjectOpenSea/seaport-core/blob/d4e8c74adc472b311ab64b5c9f9757b5bba57a15/src/lib/OrderCombiner.sol#L256) _validateOrderAndUpdateStatus
>
>```solidity
>// Validate it, update status, and determine fraction to fill.
>(bytes32 orderHash, uint256 numerator, uint256 denominator) =
>_validateOrderAndUpdateStatus(advancedOrder, revertOnInvalid);
>```
>
>finally we [call](https://github.com/ProjectOpenSea/seaport-core/blob/d4e8c74adc472b311ab64b5c9f9757b5bba57a15/src/lib/OrderValidator.sol#L178C21-L178C37) the logic _getGeneratedOrder(orderParameters, advancedOrder.extraData, revertOnInvalid) below with parameter revertOnInvalid false
>
>```solidity
>  // If the order is a contract order, return the generated order.
>  if (orderParameters.orderType == OrderType.CONTRACT) {
>     // Ensure that the numerator and denominator are both equal to 1.
>     assembly {
>        // (1 ^ nd =/= 0) => (nd =/= 1) => (n =/= 1) || (d =/= 1)
>        // It's important that the values are 120-bit masked before
>        // multiplication is applied. Otherwise, the last implication
>        // above is not correct (mod 2^256).
>        invalidFraction := xor(mul(numerator, denominator), 1)
>     }
>
>     // Revert if the supplied numerator and denominator are not valid.
>     if (invalidFraction) {
>        revertBadFraction();
>     }
>
>     // Return the generated order based on the order params and the
>     // provided extra data. If revertOnInvalid is true, the function
>     // will revert if the input is invalid.
>     return _getGeneratedOrder(orderParameters, advancedOrder.extraData, revertOnInvalid);
> }
>```
>
>Ok what does this mean?
>
>suppose the CreateOfferer.sol fulfill two orders and create two delegate tokens
>
>the nonces start from 0 and then incremen to 1 and then increment to 2
>
>a user craft a contract with malformed minimumReceived and combine with another valid order to call OrderCombine
>
>as we can see above, when multiple order is passed in, the revertOnInvalid is set to false, so the contract order from CreateOfferer.sol is skipped, but the nonce is incremented
>
>then the nonce tracked by CreateOfferer.sol internally is out of sycn with the contract nonce in seaport contract forever
>
>then the CreateOfferer.sol is not usable because if the [ratifyOrder callback](https://github.com/ProjectOpenSea/seaport/blob/539f0c18af85152aff9d64d90a55cf1627fd3e25/reference/lib/ReferenceZoneInteraction.sol#L147) hit the contract, transaction revert [in this check](https://github.com/code-423n4/2023-09-delegate/blob/a6dbac8068760ee4fc5bababb57e3fe79e5eeb2e/src/libraries/CreateOffererLib.sol#L185)
>
>```solidity
> if (nonce.value != contractNonce) revert CreateOffererErrors.InvalidContractNonce(nonce.value, contractNonce);
>```
>
>## Tools Used
>
>Manual Review
>
>## Recommended Mitigation Steps
>
>I would recommend do not validate the order execution in the ractifyOrder call back by using the contract nonce, instead, validate the order using [orderHash](https://github.com/ProjectOpenSea/seaport/blob/539f0c18af85152aff9d64d90a55cf1627fd3e25/reference/lib/ReferenceZoneInteraction.sol#L151)
>
>```solidity
>if (
>ContractOffererInterface(offerer).ratifyOrder(
>orderToExecute.spentItems,
>orderToExecute.receivedItems,
>advancedOrder.extraData,
>orderHashes,
>uint256(orderHash) ^ (uint256(uint160(offerer)) << 96)
>) != ContractOffererInterface.ratifyOrder.selector
>)
>```
>
>## Assessed type
>
>DoS

## Summary

The CreateOfferer contract integrates with Seaport to facilitate the creation of delegate tokens through marketplace orders. It uses a nonce system to track and validate orders, ensuring uniqueness and preventing replays. However, due to a strict equality check in nonce validation, the contract's internal nonce can become desynchronized from Seaport's nonce when invalid orders are submitted and skipped (without reverting). This desynchronization causes all future valid orders to fail validation, resulting in a permanent denial of service (DoS) where no new orders can be processed or ratified.

## A Better Explanation (With Simplified Example)

### Intended Behavior

The CreateOfferer contract acts like a ticket booth at a fair, where each ticket has a unique number (the nonce) to track orders for creating delegate tokens (like lending out NFT usage rights). Seaport, the fair's organizer, also tracks these ticket numbers. When someone buys a ticket:

- The booth checks if the ticket number (Seaport's nonce) matches its own count exactly (e.g., both expect ticket #3).
- If it matches, the booth processes the order (creates delegate tokens), issues the ticket, and increments its count (to #4).
- This ensures tickets are used in order and can't be reused (preventing replay attacks).

The system assumes both the booth (CreateOfferer) and Seaport increment their counts only after a successful order, keeping them perfectly in sync.

### What Actually Happens (Bug)

Seaport increments its nonce *before* fully processing an order, even if the order is invalid and skipped (when `revertOnInvalid` is false, a Seaport feature for batch orders). CreateOfferer, however, only increments its nonce after a successful ratification and strictly requires the nonces to match exactly. If Seaport skips an invalid order:

- Seaport's nonce jumps ahead (e.g., from 3 to 4).
- CreateOfferer's nonce stays put (still 3) because the invalid order never reaches ratification.
- When a valid order comes with Seaport's new nonce (4), CreateOfferer expects 3, reverts, and never increments.

This mismatch locks the contract, as all future orders fail due to the nonce discrepancy.

### Simplified Example: The Broken Ticket Booth

Imagine you're at a fair buying ride tickets from a booth (CreateOfferer) managed by an organizer (Seaport). Each ticket has a number (nonce), and both track the next number expected.

1. **Normal Day**:

   - Booth's count: 10. Seaport's count: 10.
   - You buy a ticket for a ride (an order to delegate an NFT's usage).
   - Seaport sends ticket #10. Booth checks: 10 == 10, sells the ticket, creates the delegate token, and both increment to 11.
   - All good, you ride!

2. **Sneaky Attacker**:

   - An attacker submits two tickets together (Seaport's batch feature):
     - Ticket A: Invalid (e.g., asks for a "moon ride" the booth doesn't offer).
     - Ticket B: Valid (your real ticket for the Ferris wheel).
   - Seaport allows skipping invalids (`revertOnInvalid=false`).
   - For Ticket A:
     - Seaport checks ticket #10, but booth rejects it (invalid ride, reverts in `generateOrder` due to bad data, like wrong token type).
     - Seaport skips it but *increments to 11* anyway.
     - Booth doesn't increment (still 10) because it never ratified.
   - For Ticket B:
     - Seaport sends ticket #11 (valid order).
     - Booth checks: 10 (internal) != 11 (Seaport)? Revert! Your valid ticket is rejected.
   - Now:
     - Booth stuck at 10.
     - Seaport at 11, moving to 12, 13, etc., for future orders.
   - Next day, you try ticket #12. Booth still expects 10, rejects it. No one can buy tickets anymore—the booth is broken!

This desync happens because the booth is too picky (strict `nonce.value == contractNonce`), and Seaport is too eager (increments on skips).

### Why This Matters

- The booth (contract) can't sell tickets (process orders), halting delegate token creation.
- Users can't trade NFT rights, losing opportunities (e.g., lending an NFT's game access).
- No funds are stolen, but the system is unusable until fixed, which might need a costly redeploy.

## Vulnerable Code Reference

```solidity
// In CreateOffererHelpers library (used in ratifyOrder)
function processNonce(CreateOffererStructs.Nonce storage nonce, uint256 contractNonce) internal {
    if (nonce.value != contractNonce) revert CreateOffererErrors.InvalidContractNonce(nonce.value, contractNonce);
    unchecked {
        ++nonce.value;  // Increment only after exact match
    } // Infeasible this will overflow if starting point is zero
}

// In ratifyOrder function (called by Seaport)
function ratifyOrder(
    SpentItem[] calldata offer,
    ReceivedItem[] calldata consideration,
    bytes calldata context,
    bytes32[] calldata,
    uint256 contractNonce  // Nonce from Seaport
)
    external
    checkStage(Enums.Stage.ratify, Enums.Stage.generate)
    onlySeaport(msg.sender)
    returns (bytes4)
{
    Helpers.processNonce(nonce, contractNonce);  // Strict check here causes desync if Seaport has advanced prematurely
    Helpers.verifyCreate(delegateToken, offer[0].identifier, transientState.receivers, consideration[0], context);
    return this.ratifyOrder.selector;
}
```

## Recommended Mitigation

1. **Relax the Strict Equality Check to Allow Skips**

   Change the `processNonce` logic to prevent replays but tolerate skips by ensuring the provided nonce is greater than or equal to the internal one, then update the internal nonce accordingly.

   Change from

   ```solidity
   if (nonce.value != contractNonce) revert CreateOffererErrors.InvalidContractNonce(nonce.value, contractNonce);
   unchecked {
       ++nonce.value;
   }
   ```

   to something like

   ```solidity
   if (nonce.value > contractNonce) revert CreateOffererErrors.InvalidContractNonce(nonce.value, contractNonce);  // Prevent replays (old nonces)
   nonce.value = contractNonce + 1;  // Sync by jumping ahead if needed
   ```

2. **Add unit and integration tests simulating Seaport's behavior, including combined orders with invalids and** `revertOnInvalid=false`**, to verify nonce handling under skips.**

3. **Consider logging or emitting events on nonce updates or mismatches for better monitoring and debugging in production.**

## Pattern Recognition Notes

This vulnerability stems from a mismatch in nonce handling between integrated protocols, leading to desynchronization and DoS. Such integration errors are common in composable systems like DeFi, where assumptions about external behavior (e.g., Seaport's skipping logic) aren't fully accounted for. Here are key tips and patterns to recognize and avoid similar issues:

1. **Validate Nonce Systems in Integrations**

   - When integrating with external protocols (e.g., marketplaces like Seaport), thoroughly review how they handle nonces, especially in error or skip scenarios.
   - Assume nonces may advance asynchronously and design your system to be resilient (e.g., allow "jumping" ahead while preventing backward replays).
   - Avoid strict equality checks; use inequalities like `providedNonce >= internalNonce` to handle skips.

2. **Confirm Handling of Partial Failures or Skips**

   - Protocols like Seaport allow "non-reverting" modes for batched operations. Test integrations with these modes enabled.
   - Write down expected behaviors for valid, invalid, and skipped cases, ensuring nonces align in all paths.
   - For example, if an external system increments on attempt (not success), your contract should sync accordingly rather than assuming perfect alignment.

3. **Beware of Desynchronization Risks in Stateful Interactions**

   - Nonce mismatches often lead to DoS by "sticking" counters, similar to sequence number issues in networking.
   - This can occur in order books, oracles, or any callback-based system—watch for early increments in external flows.

4. **Separate Validation from Increment Logic**

   - Decouple nonce checks from processing: Validate first, then update regardless of skips if the nonce is valid/fresh.
   - Use mappings of used nonces (e.g., a set of consumed values) for more robust anti-replay without strict sequencing, though this increases storage costs.

5. **Write Unit Tests with Adversarial Scenarios**

   - Simulate attacks: Include tests with invalid orders, batches, and non-reverting modes.
   - Test edge cases like zero nonces, rapid submissions, and large nonce jumps.
   - Use fuzzing or property-based testing to check nonce invariants (e.g., "internal nonce always &lt;= external +1").

6. **Use Code Reviews and Static Analysis Tools Focused on Integrations**

   - During audits, flag nonce or counter logic in external calls for extra scrutiny.
   - Tools like Slither or MythX can detect reentrancy or state inconsistencies, but manual review is key for integration-specific bugs.

7. **Monitor System Behavior Post-Deployment**

   - Emit events on nonce validations, increments, or failures to track desyncs in real-time.
   - Implement monitoring alerts for stalled nonces or failed ratifications.
