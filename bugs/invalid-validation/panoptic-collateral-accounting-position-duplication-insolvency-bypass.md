# Duplicate TokenId fingerprint collision → solvency bypass in `PanopticPool.sol`

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2024-04-panoptic-findings/issues/498) / [One Bug Per Day](https://www.onebugperday.com/v1/1221)
* **Affected Contract**: [PanopticPool.sol](https://github.com/code-423n4/2024-04-panoptic/blob/main/contracts/PanopticPool.sol)
* **Vulnerability Type**: Business Logic / Input Validation / Fingerprint overflow (XOR + counter)

## Original Bug Description

>## Lines of code
>
>[https://github.com/code-423n4/2024-04-panoptic/blob/main/contracts/PanopticPool.sol#L1367-L1391](https://github.com/code-423n4/2024-04-panoptic/blob/main/contracts/PanopticPool.sol#L1367-L1391)
>[https://github.com/code-423n4/2024-04-panoptic/blob/main/contracts/PanopticPool.sol#L887-L893](https://github.com/code-423n4/2024-04-panoptic/blob/main/contracts/PanopticPool.sol#L887-L893)
>
>## Vulnerability details
>
>## Impact
>
>The underlying issue is that `_validatePositionList()` does not check for duplicate tokenIds. Attackers can use this issue to bypass solvency checks, which leads to several impacts:
>
>1. Users can mint/burn/liquidate/forceExercise when they are insolvent.
>2. Users can settleLongPremium/forceExercise another user when the other user is insolvent.
>
>This also conflicts a main invariant stated in contest readme: `Users should not be allowed to mint/burn options or pay premium if their end state is insolvent`.
>
>## Bug Description
>
>First, let's see why `_validatePositionList` does not check for duplicate tokenIds. For a user position hash, the first 8 bits is the length of tokenIds which overflows, last 248 bits is the xor hash. However, we can easily add 256 tokenIds of the same kind to create a duplicate positionHash.
>
>For example: `Hash(key0, key1, key2) == Hash(key0, key1, key2, key0, key0, ..., 256 more key0)`. This way, we can add any tokenId we want while still arriving the same position hash.
>
>PanopticPool.sol
>
>```solidity
>    function _validatePositionList(
>        address account,
>        TokenId[] calldata positionIdList,
>        uint256 offset
>    ) internal view {
>        uint256 pLength;
>        uint256 currentHash = s_positionsHash[account];
>
>        unchecked {
>            pLength = positionIdList.length - offset;
>        }
>        // note that if pLength == 0 even if a user has existing position(s) the below will fail b/c the fingerprints will mismatch
>        // Check that position hash (the fingerprint of option positions) matches the one stored for the '_account'
>        uint256 fingerprintIncomingList;
>
>        for (uint256 i = 0; i < pLength; ) {
>>           fingerprintIncomingList = PanopticMath.updatePositionsHash(
>                fingerprintIncomingList,
>                positionIdList[i],
>                ADD
>            );
>            unchecked {
>                ++i;
>            }
>        }
>
>        // revert if fingerprint for provided '_positionIdList' does not match the one stored for the '_account'
>        if (fingerprintIncomingList != currentHash) revert Errors.InputListFail();
>    }
>```
>
>PanopticMath.sol
>
>```solidity
>    function updatePositionsHash(
>        uint256 existingHash,
>        TokenId tokenId,
>        bool addFlag
>    ) internal pure returns (uint256) {
>        // add the XOR`ed hash of the single option position `tokenId` to the `existingHash`
>        // @dev 0 ^ x = x
>
>        unchecked {
>            // update hash by taking the XOR of the new tokenId
>            uint248 updatedHash = uint248(existingHash) ^
>                (uint248(uint256(keccak256(abi.encode(tokenId)))));
>            // increment the top 8 bit if addflag=true, decrement otherwise
>            return
>                addFlag
>                    ? uint256(updatedHash) + (((existingHash >> 248) + 1) << 248)
>                    : uint256(updatedHash) + (((existingHash >> 248) - 1) << 248);
>        }
>    }
>```
>
>Then, let's see how duplicate ids can bypass solvency check. The solvency check is in `_validateSolvency()`, which is called by all the user interaction functions, such as mint/burn/liquidate/forceExercise/settleLongPremium. This function first checks for a user passed in `positionIdList` (which we already proved can include duplicates), then calls `_checkSolvencyAtTick()` to calculate the `balanceCross` (collateral balance) and `thresholdCross` (required collateral) for all tokens.
>
>The key is the collateral balance includes the premium that is collected for each of the positions. For most of the positions, the collected premium should be less than required collateral to keep this position open. However, if a position has been open for a long enough time, the fees it accumulated may be larger than required collateral.
>
>For this kind of position, we can duplicate it 256 times (or multiple of 256 times, as long as gas fee is enough), and make our collateral balance grow faster than required collateral. This can make a insolvent account "solvent", by duplicating the key tokenId multiple times.
>
>```solidity
>    function _validateSolvency(
>        address user,
>        TokenId[] calldata positionIdList,
>        uint256 buffer
>    ) internal view returns (uint256 medianData) {
>        // check that the provided positionIdList matches the positions in memory
>>       _validatePositionList(user, positionIdList, 0);
>        ...
>        // Check the user's solvency at the fast tick; revert if not solvent
>        bool solventAtFast = _checkSolvencyAtTick(
>            user,
>            positionIdList,
>            currentTick,
>            fastOracleTick,
>            buffer
>        );
>        if (!solventAtFast) revert Errors.NotEnoughCollateral();
>
>        // If one of the ticks is too stale, we fall back to the more conservative tick, i.e, the user must be solvent at both the fast and slow oracle ticks.
>        if (Math.abs(int256(fastOracleTick) - slowOracleTick) > MAX_SLOW_FAST_DELTA)
>            if (!_checkSolvencyAtTick(user, positionIdList, currentTick, slowOracleTick, buffer))
>                revert Errors.NotEnoughCollateral();
>    }
>
>    function _checkSolvencyAtTick(
>        address account,
>        TokenId[] calldata positionIdList,
>        int24 currentTick,
>        int24 atTick,
>        uint256 buffer
>    ) internal view returns (bool) {
>        (
>            LeftRightSigned portfolioPremium,
>            uint256[2][] memory positionBalanceArray
>        ) = _calculateAccumulatedPremia(
>                account,
>                positionIdList,
>                COMPUTE_ALL_PREMIA,
>                ONLY_AVAILABLE_PREMIUM,
>                currentTick
>            );
>
>        LeftRightUnsigned tokenData0 = s_collateralToken0.getAccountMarginDetails(
>            account,
>            atTick,
>            positionBalanceArray,
>            portfolioPremium.rightSlot()
>        );
>        LeftRightUnsigned tokenData1 = s_collateralToken1.getAccountMarginDetails(
>            account,
>            atTick,
>            positionBalanceArray,
>            portfolioPremium.leftSlot()
>        );
>
>        (uint256 balanceCross, uint256 thresholdCross) = _getSolvencyBalances(
>            tokenData0,
>            tokenData1,
>            Math.getSqrtRatioAtTick(atTick)
>        );
>
>        // compare balance and required tokens, can use unsafe div because denominator is always nonzero
>        unchecked {
>            return balanceCross >= Math.unsafeDivRoundingUp(thresholdCross * buffer, 10_000);
>        }
>    }
>```
>
>CollateralTracker.sol
>
>```solidity
>    function getAccountMarginDetails(
>        address user,
>        int24 currentTick,
>        uint256[2][] memory positionBalanceArray,
>        int128 premiumAllPositions
>    ) public view returns (LeftRightUnsigned tokenData) {
>        tokenData = _getAccountMargin(user, currentTick, positionBalanceArray, premiumAllPositions);
>    }
>
>    function _getAccountMargin(
>        address user,
>        int24 atTick,
>        uint256[2][] memory positionBalanceArray,
>        int128 premiumAllPositions
>    ) internal view returns (LeftRightUnsigned tokenData) {
>        uint256 tokenRequired;
>
>        // if the account has active options, compute the required collateral to keep account in good health
>        if (positionBalanceArray.length > 0) {
>            // get all collateral required for the incoming list of positions
>            tokenRequired = _getTotalRequiredCollateral(atTick, positionBalanceArray);
>
>            // If premium is negative (ie. user has to pay for their purchased options), add this long premium to the token requirement
>            if (premiumAllPositions < 0) {
>                unchecked {
>                    tokenRequired += uint128(-premiumAllPositions);
>                }
>            }
>        }
>
>        // if premium is positive (ie. user will receive funds due to selling options), add this premum to the user's balance
>        uint256 netBalance = convertToAssets(balanceOf[user]);
>        if (premiumAllPositions > 0) {
>            unchecked {
>                netBalance += uint256(uint128(premiumAllPositions));
>            }
>        }
>
>        // store assetBalance and tokens required in tokenData variable
>        tokenData = tokenData.toRightSlot(netBalance.toUint128()).toLeftSlot(
>            tokenRequired.toUint128()
>        );
>        return tokenData;
>    }
>```
>
>Now we have shown how to bypass the _validateSolvency(), we can bypass all related checks. Listing them here:
>
>1. Burn options [#1](https://github.com/code-423n4/2024-04-panoptic/blob/main/contracts/PanopticPool.sol#L577), [#2](https://github.com/code-423n4/2024-04-panoptic/blob/main/contracts/PanopticPool.sol#L600)
>2. Mint options [#1](https://github.com/code-423n4/2024-04-panoptic/blob/main/contracts/PanopticPool.sol#L661)
>3. Force exercise [#1](https://github.com/code-423n4/2024-04-panoptic/blob/main/contracts/PanopticPool.sol#L661), [#2](https://github.com/code-423n4/2024-04-panoptic/blob/main/contracts/PanopticPool.sol#L1275)
>4. SettleLongPremium [#1](https://github.com/code-423n4/2024-04-panoptic/blob/main/contracts/PanopticPool.sol#L1658)
>
>## Proof of Concept
>
>In this report, we will only prove the core issue: duplicate tokenIds are allowed, and won't craft complicated scenarios for relevant impacts.
>
>Add the following test code in `PanopticPool.t.sol`, we can see that passing 257 of the same token ids can still work for `mintOptions()`.
>
>```solidity
>    function test_duplicatePositionHash(
>        uint256 x,
>        uint256[2] memory widthSeeds,
>        int256[2] memory strikeSeeds,
>        uint256[2] memory positionSizeSeeds,
>        uint256 swapSizeSeed
>    ) public {
>        _initPool(x);
>
>        (int24 width, int24 strike) = PositionUtils.getOTMSW(
>            widthSeeds[0],
>            strikeSeeds[0],
>            uint24(tickSpacing),
>            currentTick,
>            0
>        );
>
>        (int24 width2, int24 strike2) = PositionUtils.getOTMSW(
>            widthSeeds[1],
>            strikeSeeds[1],
>            uint24(tickSpacing),
>            currentTick,
>            0
>        );
>        vm.assume(width2 != width || strike2 != strike);
>
>        populatePositionData([width, width2], [strike, strike2], positionSizeSeeds);
>
>        // leg 1
>        TokenId tokenId = TokenId.wrap(0).addPoolId(poolId).addLeg(
>            0, 1, isWETH, 0, 0, 0, strike, width
>        );
>
>        TokenId[] memory posIdList = new TokenId[](257);
>        for (uint i = 0; i < 257; ++i) {
>            posIdList[i] = tokenId;
>        }
>        pp.mintOptions(posIdList, positionSizes[0], 0, 0, 0);
>    }
>```
>
>## Tools Used
>
>Manual review
>
>## Recommended Mitigation Steps
>
>Add a check in `_validatePositionList` that the length is shorter than `MAX_POSITIONS` (32).
>
>## Assessed type
>
>Invalid Validation

## Summary

`PanopticPool` uses a compact 256-bit *fingerprint* to validate a user's list of open positions (`tokenId`s). The top **8 bits** store a position count, and the lower **248 bits** store the XOR of `keccak(tokenId)` values. Because the count is only 8 bits (wraps modulo 256) and XOR cancels duplicate inputs, an attacker can append 256 (or any multiple of 256) duplicates of a `tokenId` to a valid list without changing the stored fingerprint. That forged list is then passed to solvency routines which **sum premiums and compute required collateral from the incoming list**, allowing a malicious user to multiply positive premium contributions while required collateral does not scale proportionally. This enables bypassing solvency checks and abusing flows like `mintOptions`, `burnOptions`, `forceExercise`, and `settleLongPremium`.

## A Better Explanation (With Simplified Example)

### Intended behavior

* The protocol keeps a **single canonical fingerprint** (`s_positionsHash[account]`) representing the *set/multiset* of an account's positions (the actual positions the account holds).
* When a user calls state-changing functions they pass the current `positionIdList`. The contract recomputes a fingerprint from that list and compares it to `s_positionsHash[account]`. If they match, the provided list is accepted as authoritative. This protects downstream code that computes premiums, requirements, and solvency from input tampering.

### What actually happens (bug)

* `PanopticMath.updatePositionsHash` computes:

  * bottom 248 bits = XOR of `uint248(keccak256(abi.encode(tokenId)))` across items
  * top 8 bits = count of items (incremented/decremented on add/remove)
* Two mathematical properties collide maliciously:

  1. XOR has cancellation: `x ^ x = 0`. Repeating the same `keccak(tokenId)` an even number of times cancels out.
  2. The top counter is 8 bits → it wraps modulo 256. Adding 256 to the count leaves the stored 8-bit value unchanged.
* Thus adding **256 duplicates** of any `tokenId` to a valid list leaves both the XOR and the 8-bit count unchanged → fingerprint matches stored value even though the provided list is not the real position list.

### Toy XOR + overflow example (concrete)

Use tiny toy values to make XOR/count behavior obvious.

* Suppose:

  * `hash(A)` = `0x05`
  * `hash(B)` = `0x07`

* Real (legit) list: `[A, B]`

  * count = `2` (top 8 bits)
  * XOR = `5 ^ 7 = 2` (bottom bits)
  * fingerprint = `[count=2 | xor=2]`

* Attacker list: `[A, B] + (256 × A)`

  * logical count = `2 + 256 = 258`, but top 8 bits store `258 mod 256 = 2`
  * XOR = `5 ^ 7 ^ (5 repeated 256 times)` → pairs of `5` cancel; since 256 is even, XOR reduces to `5 ^ 7 = 2`
  * fingerprint = `[count=2 | xor=2]` → identical to legit fingerprint

**Result**: `_validatePositionList()` accepts the forged list as valid.

### How that enables solvency bypass (numeric example with premiums & collateral)

A simplified accounting model used by the pool:

* `balance = collateral_deposited + sum_of_positive_premia_from_incoming_list`
* `required = computed_required_collateral(positionBalanceArray)` — derived from the *unique positions* but the incoming list drives the `positionBalanceArray` and premium calculation.

Exploit scenario:

1. Bob (short) holds one "old" profitable position `P` which has:

   * `required_collateral_for_P = $100`
   * `accumulated_premium_for_P = +$150` (positive premium — the position has earned fees)

2. Without duplication:

   * `balance = collateral + 150`
   * `required = 100`
   * If collateral is small, but `balance >= required`, Bob is nominally solvent.

3. With duplication (attacker supplies the incoming list with 256 copies of `P`):

   * `_calculateAccumulatedPremia()` naively aggregates premia across the provided entries → `portfolioPremium ≈ 150 × 256 = 38,400`
   * `_getTotalRequiredCollateral(...)` either scales differently or is not inflated by duplicated positive-premium entries proportionally (depending on implementation it may dedupe or not apply the same multiplicative factor).
   * Final numbers seen by `_checkSolvencyAtTick()` may be:

     * `balance ≈ collateral + 38,400`
     * `required ≈ 100 × 256 = 25,600` (or if requirement is not scaled by duplicates, even smaller)
   * `balance >= required` → solvency check passes, but in reality Bob only had one real position and far less real collateral/premia.

**Attack surface consequences**:

* An attacker (or a hostile actor controlling/forging `positionIdList`) can:

  * Call `mintOptions` while actually undercollateralized to mint options or extract premium.
  * Call `burnOptions` or other state changes that should be blocked for insolvent accounts.
  * Force settlement/`forceExercise` flows and interfere with liquidation rules.
  * Cause bad debt and break core safety invariants.

## Vulnerable Code Reference

Key spots cited in the report and found in the contract.

### `_validatePositionList` (PanopticPool.sol)

```solidity
function _validatePositionList(
    address account,
    TokenId[] calldata positionIdList,
    uint256 offset
) internal view {
    uint256 pLength;
    uint256 currentHash = s_positionsHash[account];

    unchecked {
        pLength = positionIdList.length - offset;
    }

    uint256 fingerprintIncomingList;

    for (uint256 i = 0; i < pLength; ) {
        fingerprintIncomingList = PanopticMath.updatePositionsHash(
            fingerprintIncomingList,
            positionIdList[i],
            ADD
        );
        unchecked {
            ++i;
        }
    }

    if (fingerprintIncomingList != currentHash) revert Errors.InputListFail();
}
```

***(no duplication check, no explicit length cap)***

### `updatePositionsHash` (PanopticMath.sol)

```solidity
function updatePositionsHash(
    uint256 existingHash,
    TokenId tokenId,
    bool addFlag
) internal pure returns (uint256) {
    unchecked {
        uint248 updatedHash = uint248(existingHash) ^
            (uint248(uint256(keccak256(abi.encode(tokenId)))));
        // increment the top 8 bit if addflag=true, decrement otherwise
        return
            addFlag
                ? uint256(updatedHash) + (((existingHash >> 248) + 1) << 248)
                : uint256(updatedHash) + (((existingHash >> 248) - 1) << 248);
    }
}
```

***(top 8 bits store count → 8-bit overflow risk; bottom bits use XOR → duplicates can cancel)***

## Recommended Mitigation

***Immediate (hotfix) — Minimal & effective***

1. **Reject long lists and duplicates early** in `_validatePositionList`:

   * Enforce `pLength <= MAX_POSITIONS` (e.g., 32).
   * Add an O(n²) duplicate check (cheap because `MAX_POSITIONS` is small).
   * Only after these checks compute the fingerprint.

    Example patch (drop-in):

    ```solidity
    uint256 internal constant MAX_POSITIONS = 32;

    function _validatePositionList(
        address account,
        TokenId[] calldata positionIdList,
        uint256 offset
    ) internal view {
        uint256 pLength;
        uint256 currentHash = s_positionsHash[account];

        unchecked {
            pLength = positionIdList.length - offset;
        }

        if (pLength > MAX_POSITIONS) revert Errors.InputListFail();

        // duplicate check (O(n^2) safe for MAX_POSITIONS small)
        for (uint256 i = 0; i < pLength; ++i) {
            for (uint256 j = i + 1; j < pLength; ++j) {
                if (positionIdList[i] == positionIdList[j]) revert Errors.InputListFail();
            }
        }

        uint256 fingerprintIncomingList;
        for (uint256 i = 0; i < pLength; ) {
            fingerprintIncomingList = PanopticMath.updatePositionsHash(
                fingerprintIncomingList,
                positionIdList[i],
                ADD
            );
            unchecked { ++i; }
        }

        if (fingerprintIncomingList != currentHash) revert Errors.InputListFail();
    }
    ```

    ***Short/medium term (robustness improvements)***

2. **Store exact count separately** (avoid packing count into 8 bits). Replace top-8-bit count with a separate `mapping(address => uint256) s_positionsCount;` updated on mint/burn. This removes wrap-around risk entirely and keeps fingerprint semantics clear.

3. **Use canonical ordering + cryptographic hash**:

    * Require callers to pass `positionIdList` sorted (ascending). Compute `keccak256(abi.encodePacked(positionIdList))` and compare to `s_positionsHash`.
    * Or compute a Merkle root or `keccak` of the canonical concatenation. This prevents reordering or duplication attacks.
    * Sorting + hashing is more expensive gas-wise but robust; use it if lists are short or offload sorting to the caller (require sorted input).

4. **If you keep XOR style fingerprinting, do not pack count in 8-bits** and do not rely on XOR for multisets. XOR is not collision-resistant for multiset membership.

5. **Add observability & limits**:

   * Emit events when `positionsHash` is updated.
   * Reject any `positionIdList` length above a safe cap (e.g., 32) at the ABI boundary to avoid gas DOS or fingerprint abuse.

    **Testing & operational**

6. **Add unit & fuzz tests**:

    * Test that `positionIdList` with 257 duplicates or any multiple of 256 is rejected.
    * Fuzz `positionIdList` inputs for `mintOptions`, `burnOptions`, `forceExercise`, `settleLongPremium` to ensure solvency invariant holds.
    * Reproduce the PoC test (257 identical token IDs) and assert failures on patched code.

7. **Consider a short-term operational pause** on risky entrypoints (if pause switch exists) while hotfix is deployed if exploit seems feasible on mainnet.

## Pattern Recognition Notes

This issue is a classic example of *fingerprint / compact representation* pitfalls combined with small-field counters and XOR aggregation. When auditing or authoring similar systems:

1. **Avoid packing counters into tiny bit fields used for security checks.**

   * Small fixed-width counters can overflow/wrap; do not rely on them for uniqueness semantics.

2. **XOR is not a set/multiset-safe aggregator.**

   * XOR cancels duplicates — it cannot distinguish odd vs even multiplicity. Don't use XOR as the sole integrity check for multisets.

3. **Canonicalization is essential.**

   * Either require canonical order (and hash the ordered list) or maintain a canonical representation server-side (sorted unique list) before hashing.

4. **When memory/storage is optimized for gas, trade off correctness before micro-optimizations.**

   * Compact fingerprints are attractive for gas savings, but they must be collision-resistant (or used with additional safeguards).

5. **Enforce bounds & de-duplication at ABI boundary.**

   * Always enforce sensible list-size limits and check duplicates before any cryptographic or business logic depends on the list.

6. **Test the invariants, not just the happy path.**

   * Add property-based tests: "for any input list that passes validation, the set of unique tokenIds must equal the stored set" — this would catch duplication exploits.

7. **Monitoring: watch for anomalous lists.**

   * Emit events for extremely long `positionIdList` or when `positionsHash` updates in unusual patterns. Have alerts for calls with very large lists.
