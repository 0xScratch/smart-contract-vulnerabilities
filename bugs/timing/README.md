# Timing / Oracle Manipulation

## Major Real-World Incidents Involving Similar Vulnerabilities

### **bZx Protocol (February 2020)**

- **First Attack**: $350,000 stolen through oracle manipulation
- **Second Attack**: $630,000 stolen just 4 days later [[1]](https://cryptobriefing.com/2388-eth-estimated-lost-bzxs-second-exploit/)
- **Method**: Attackers used flash loans to manipulate sUSD prices via Kyber Network oracle, causing bZx to accept inflated collateral values [[2]](https://www.aon.com/en/insights/cyber-labs/flash-loan-attacks-a-case-study) [[3]](https://arxiv.org/html/2411.01230v1)
- **Impact**: Demonstrated how timing assumptions in oracle feeds can be exploited repeatedly

### **Mango Markets (October 2022)**

- **Loss**: Over $100 million stolen [[4]](https://www.theblock.co/post/176445/hacker-steals-over-100-million-from-mango-markets)
- **Method**: Avraham Eisenberg manipulated MNGO token price across multiple exchanges to inflate collateral value [[5]](https://medium.com/coinmonks/how-to-analyze-an-attack-a-case-on-the-mango-markets-hack-f1d4389c009f)
- **Legal outcome**: Eisenberg was prosecuted by DOJ, SEC, and CFTC for market manipulation [[6]](https://www.cftc.gov/PressRoom/PressReleases/8647-23)
- **Connection**: Shows real-world consequences of oracle timing assumptions

### **PancakeBunny (May 2021)**

- **Loss**: $45 million stolen [[7]](https://www.theblock.co/post/105473/bsc-pancakebunny-defi-protocol-exploited-lost-45-million-bunny)
- **Token impact**: BUNNY crashed from $146 to $6 (95% loss)
- **Method**: Flash loan attack exploiting oracle manipulation to mint 7 million BUNNY tokens [[8]](https://coinmarketcap.com/academy/article/the-tragicomedy-of-pancakebunny) [[9]](https://www.merklescience.com/blog/hack-track-pancake-bunny-hack)
- **Recovery**: Partial compensation plan implemented [[10]](https://pancakebunny.medium.com/polybunny-post-mortem-compensation-42b5c35ce957)

### **Warp Finance (December 2020)**

- **Loss**: $7.7 million stolen [[11]](https://cointelegraph.com/news/after-exploit-warp-finance-compensation-plan-takes-promising-strides)
- **Recovery**: 75% of funds recovered ($5.85 million) [[12]](https://www.theblock.co/post/88645/defi-protocol-warp-finance-recovered-funds-flash-loan-attack)
- **Method**: Flash loan attack manipulating Warp's price oracle system [[13]](https://slowmist.medium.com/analysis-of-warp-finance-hacked-incident-cb12a1af74cc)
- **Impact**: Led to increased focus on robust price oracles in DeFi

### Why These Matter

These incidents demonstrate that blockchain parameter assumptions (like block times) and oracle timing vulnerabilities aren't theoretical—they've caused hundreds of millions in losses. The Basin MultiFlowPump vulnerability follows the same pattern: hardcoded timing assumptions that attackers can exploit when network conditions change.
