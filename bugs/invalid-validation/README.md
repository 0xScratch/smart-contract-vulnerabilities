# Invalid Validation

## Major Real-World Incidents Involving Similar Vulnerabilities

### **XMON Unlock Strategy Sandwich Attack (2023)**

- **Loss**: Trader lost effectively 100% of their \$10,000 swap, receiving only ~\$3.90 worth of XMON, then automatically locking it and receiving a nearly worthless airdrop [[1]](https://www.theblock.co/post/216487/trader-makes-error-in-xmon-unlock-strategy-loses-100-of-trade-in-slippage)
- **Method**: Trader's custom contract set `amountOutMin = 0` when swapping WETH for XMON; an MEV bot frontrunner manipulated the price, causing the swap to execute at the worst possible rate and capturing the slippage profit
- **Legal outcome**: No known legal action; incident highlighted the critical need for slippage checks in custom swap contracts
- **Connection**: Demonstrates how missing minimum-output validation can enable sandwich attacks that drain user funds without theft of assets from the contract itself

### **Sushiswap Swap Path Manipulation Hack (2023)**

- **Loss**: Over \$3.3 million drained from Sushiswap liquidity pools due to malformed swap path input [[2]](https://www.halborn.com/blog/post/explained-the-sushi-swap-hack-march-2023)
- **Method**: Attackers exploited improper validation of the `path[]` parameter in swap functions, injecting malicious token addresses that diverted user swaps to attacker-controlled contracts [[3]](https://hacken.io/discover/sushi-hack-explained/)
- **Legal outcome**: No public prosecutions; vulnerability spurred widespread adoption of rigorous input sanitization for swap router parameters
- **Connection**: Classic input-validation failure where unverified user-supplied arrays allowed routing of funds to unintended destinations

### **AffineDeFi Upgrade Exploit via Input Validation (2024)**

- **Loss**: Attacker stole ~33 aEthwstETH (~\$88 000) from Affine protocol by abusing upgrade parameters [[4]](https://blog.verichains.io/p/lack-of-validation-in-input-data)
- **Method**: Flash-loan-funded calls to `receiveFlashLoan` bypassed checks for `newStrategy` and `tx.origin`, allowing malicious upgrade and collateral transfer [[5]](https://medium.com/neptune-mutual/how-was-affine-protocol-exploited-f4933c4035b4)
- **Legal outcome**: No reported legal proceedings; incident emphasized need for strict validation of callback parameters in flash-loan callbacks
- **Connection**: Highlights risk when user-controlled inputs directly drive critical logic (upgrade and collateral transfer) without content validation
