The translation is from ( [https://github.com/ethereum/go-ethereum/pull/1889](https://github.com/ethereum/go-ethereum/pull/1889) )

This PR aggregates a lot of small modifications to core, trie, eth and other packages to collectively implement the eth/63 fast synchronization algorithm. In short, geth --fast.

## Algorithm

The goal of the the fast sync algorithm is to exchange processing power for bandwidth usage. Instead of processing the entire block-chain one link at a time, and replay all transactions that ever happened in history, fast syncing downloads the transaction receipts along the blocks, and pulls an entire recent state database. This allows a fast synced node to still retain its status an an archive node containing all historical data for user queries (and thus not influence the network's health in general), but at the same time to reassemble a recent network state at a fraction of the time it would take full block processing.

An outline of the fast sync algorithm would be:

- Similarly to classical sync, download the block headers and bodies that make up the blockchain
- Similarly to classical sync, verify the header chain's consistency (POW, total difficulty, etc)
- Instead of processing the blocks, download the transaction receipts as defined by the header
- Store the downloaded blockchain, along with the receipt chain, enabling all historical queries
- When the chain reaches a recent enough state (head - 1024 blocks), pause for state sync:
	- Retrieve the entire Merkel Patricia state trie defined by the root hash of the pivot point
	- For every account found in the trie, retrieve it's contract code and internal storage state trie
- Upon successful trie download, mark the pivot point (head - 1024 blocks) as the current head
- Import all remaining blocks (1024) by fully processing them as in the classical sync

## Analysis
By downloading and verifying the entire header chain, we can guarantee with all the security of the classical sync, that the hashes (receipts, state tries, etc) contained within the headers are valid. Based on those hashes, we can confidently download transaction receipts and the entire state trie afterwards. Additionally, by placing the pivoting point (where fast sync switches to block processing) a bit below the current head (1024 blocks), we can ensure that even larger chain reorganizations can be handled without the need of a new sync (as we have all the state going that many blocks back).

## Caveats
The historical block-processing based synchronization mechanism has two (approximately similarly costing) bottlenecks: transaction processing and PoW verification. The baseline fast sync algorithm successfully circumvents the transaction processing, skipping the need to iterate over every single state the system ever was in. However, verifying the proof of work associated with each header is still a notably CPU intensive operation.

However, we can notice an interesting phenomenon during header verification. With a negligible probability of error, we can still guarantee the validity of the chain, only by verifying every K-th header, instead of each and every one. By selecting a single header at random out of every K headers to verify, we guarantee the validity of an N-length chain with the probability of (1/K)^(N/K) (i.e. we have 1/K chance to spot a forgery in K blocks, a verification that's repeated N/K times).

Let's define the negligible probability Pn as the probability of obtaining a 256 bit SHA3 collision (i.e. the hash Ethereum is built upon): 1/2^128. To honor the Ethereum security requirements, we need to choose the minimum chain length N (below which we veriy every header) and maximum K verification batch size such as (1/K)^(N/K) <= Pn holds. Calculating this for various {N, K} pairs is pretty straighforward, a simple and lenient solution being http://play.golang.org/p/B-8sX_6Dq0.


| N    | K   | N    | K   | N    | K   | N    | K   |
| ---- | --- | ---- | --- | ---- | --- | ---- | --- |
| 1024 | 43  | 1792 | 91  | 2560 | 143 | 3328 | 198 |
| 1152 | 51  | 1920 | 99  | 2688 | 152 | 3456 | 207 |
| 1280 | 58  | 2048 | 108 | 2816 | 161 | 3584 | 217 |
| 1408 | 66  | 2176 | 116 | 2944 | 170 | 3712 | 226 |
| 1536 | 74  | 2304 | 128 | 3072 | 179 | 3840 | 236 |
| 1664 | 82  | 2432 | 134 | 3200 | 189 | 3968 | 246 |


The above table should be interpreted in such a way, that if we verify every K-th header, after N headers the probability of a forgery is smaller than the probability of an attacker producing a SHA3 collision. It also means, that if a forgery is indeed detected, the last N headers should be discarded as not safe enough. Any {N, K} pair may be chosen from the above table, and to keep the numbers reasonably looking, we chose N=2048, K=100. This will be fine tuned later after being able to observe network bandwidth/latency effects and possibly behavior on more CPU limited devices.

Using this caveat however would mean, that the pivot point can be considered secure only after N headers have been imported after the pivot itself. To prove the pivot safe faster, we stop the "gapped verificatios" X headers before the pivot point, and verify every single header onward, including an additioanl X headers post-pivot before accepting the pivot's state. Given the above N and K numbers, we chose X=24 as a safe number.

With this caveat calculated, the fast sync should be modified so that up to the pivoting point - X, only every K=100-th header should be verified (at random), after which all headers up to pivot point + X should be fully verified before starting state database downloading. Note: if a sync fails due to header verification the last N headers must be discarded as they cannot be trusted enough.


## Weakness
Blockchain protocols in general (i.e. Bitcoin, Ethereum, and the others) are susceptible to Sybil attacks, where an attacker tries to completely isolate a node from the rest of the network, making it believe a false truth as to what the state of the real network is. This permits the attacker to spend certain funds in both the real ne