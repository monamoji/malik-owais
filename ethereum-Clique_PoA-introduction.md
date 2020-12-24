https://github.com/ethereum/EIPs/issues/225

In Clique mode, users can't get Ethereum because they can't mine, so if you need Ethereum, you need to get it through special channels.

You can get ether from this website.

https://faucet.rinkeby.io/

You need to have a google+ account, facebook or twitter account to get it. For detailed information, please refer to the above website.

Clique is a Power of authority implementation of Ethereum and is now primarily used on the Rinkeby test network.

## Background

The first official test network of Ethereum was Morden. From July 2015 to November 2016, due to the accumulated garbage between Geth and Parity and some testnet consensus issues, it was decided to stop restarting testnet.

Ropsten was born, cleaned up all the rubbish, starting with a clean slate. This operation continued until the end of February 2017, when malicious actors decided to abuse Pow and gradually increased GasLimit from the normal 4.7 million to 9 billion, at which time a huge transaction was sent to damage the entire network. Even before this, the attacker tried several very long blockchain reorganizations, resulting in network splits between different clients, even different versions.

The root cause of these attacks is that the security of the PoW network is as secure as the computing power behind it. Restarting a new test network from scratch does not solve any problems because the attacker can install the same attack over and over again. The Parity team decided to take an urgent solution, roll back a large number of blocks, and develop a soft cross that does not allow GasLimit to exceed a certain threshold.

Although this solution may work in the short term:

This is not elegant: Ethereum should have dynamic restrictions. This is not portable: other customers need to implement new fork logic on their own. It is not compatible with synchronous mode: fast sync and light client are both bad luck. This only extends the attack. Time: Garbage can still steadily advance in the endless situation Parity's 