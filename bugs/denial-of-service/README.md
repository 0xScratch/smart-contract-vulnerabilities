# Denial Of Service

## Major Real-World Incidents Involving Griefing Attacks

### **Parity Wallet Library Self-Destruct (November 2017)**

- **Loss**: Approximately 500,000 ETH frozen (no direct theft but funds irretrievably locked) [[1]](https://medium.com/paritytech/a-postmortem-on-the-parity-multi-sig-library-self-destruct-63daca3a4cf7)
- **Method**: Attacker calling the unguarded `selfdestruct` in the shared library contract behind all Parity multisig wallets, deleting code and rendering all dependent wallet stubs unusable [[2]](https://blog.openzeppelin.com/parity-wallet-hack-reloaded)
- **Legal outcome**: No criminal prosecution; Parity Tech disabled the library and warned users but funds remain locked without on-chain remedy
- **Connection**: A classic Denial of Service against wallet functionality—attackers didn't steal funds but prevented any withdrawals, indefinitely freezing user assets

### **KingOfEther Denial of Service (2016)**

- **Loss**: No ether stolen, but contract permanently locked—no one else could claim "king" status thereafter [[3]](https://medium.com/@Knownsec_Blockchain_Lab/in-depth-understanding-of-denial-of-service-vulnerabilities-dd437b1d7a1c)
- **Method**: Attacker contract becomes the "king" in the `claimThrone()` pattern and omits a payable fallback, causing refunds to fail and blocking all subsequent claims
- **Legal outcome**: No legal action; demonstrated vulnerability informed best practices for refund patterns
- **Connection**: Illustrates gas-based DoS where failing external calls prevent core functionality (new bids/refunds) without asset theft

### **Bitcoin Testnet Griefing Attack (April 2025)**

- **Loss**: No monetary loss but disrupted normal testnet operations, delaying development and sync processes [[4]](https://www.theblock.co/post/291519/bitcoin-testnet-griefing-attack-generates-three-years-worth-of-blocks-in-one-week-frustrating-developers)
- **Method**: Jameson Lopp spammed testnet with low-fee transactions to force heavy block generation (3 years' worth in one week), overwhelming nodes and clogging the mempool
- **Legal outcome**: No legal action; sparked debate on permissionless testnets and need for rate limiting
- **Connection**: Pure griefing—attacker gained nothing except disruption, exemplifying DoS at the network layer
