# Front Running / MEV (Miner Extractable Value) / Auction Manipulation / ERC-777 DoS (Denial of Service) / Block Stuffing

## Major Real-World Incidents Involving Similar Vulnerabilities

### Fomo3D Block-Stuffing Attack (August 2018)

- **Loss**: Around 10,469 ETH (~$3 million at the time) [[1]](https://hackernoon.com/the-anatomy-of-a-block-stuffing-attack-a488698732ae)
- **Method**: Attacker deployed a gas-burning contract to fill 17 consecutive Ethereum blocks (~175 s), preventing any other "key" purchases and ensuring they were the last player when the game's 24 h timer expired.
- **Connection**: Demonstrates how block stuffing can censor competing transactions in a time-boxed auction/game, identical to manipulating ECG auctions by preventing bids until the most profitable window.

### MakerDAO "Black Thursday" Zero-Bid Liquidations (March 2020)

- **Loss**: ≈$4.5 million in unbacked DAI; vault owners lost all collateral [[2]](https://cryptoslate.com/defi-posterchild-makerdao-reflects-on-4-million-black-thursday-eth-losses/)
- **Method**: Sudden ETH crash + network congestion spiked gas fees; keepers' bids failed to confirm, enabling a single liquidator to win auctions with zero-DAI bids, acquiring ETH collateral effectively for free. [[3]](https://insights.glassnode.com/what-really-happened-to-makerdao/)
- **Connection**: An unintentionally short auction window with high on-chain congestion acted like a block-stuffing vector, allowing a lone actor to capture collateral at zero cost—mirrors ECG's risk when others cannot bid before midpoint.

### Uniswap v1 imBTC ERC-777 Reentrancy Exploit (April 2020)

- **Loss**: $1.1 million worth of imBTC drained from Uniswap; $25 million from Lendf.Me [[4]](https://securityboulevard.com/2020/04/a-hackers-dream-payday-ledf-me-and-uniswap-lose-25-million-worth-of-cryptocurrency/)
- **Method**: imBTC's ERC-777 tokensToSend hook enabled reentrancy in transferFrom, letting attackers repeatedly withdraw ETH before balances updated, draining liquidity pools in a single transaction. [[5]](https://blog.blockmagnates.com/detailed-explanation-of-uniswaps-erc777-re-entry-risk-8fa5b3738e08)
- **Connection**: Highlights dangers of push-pattern transfers to ERC-777 tokens—a core element in ECG's vulnerability where malicious collateral hooks DoS auction bids until midpoint.
