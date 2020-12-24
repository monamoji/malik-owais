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

This is not elegant: Ethereum should have dynamic restrictions. This is not portable: other customers need to implement new fork logic on their own. It is not compatible with synchronous mode: fast sync and light client are both bad luck. This only extends the attack. Time: Garbage can still steadily advance in the endless situation Parity's solution is not perfect, but still feasible. I want to come up with a longer-term alternative solution that involves more, but should be simple enough to be launched in a reasonable amount of time.

## Standardized PoA

As mentioned above, in a network with no value, Pow cannot work safely. Ethereum has a long-term PoS goal based on Casper, but this is a tedious study, so we can't rely on it to solve today's problems. However, a solution is easy to implement and is sufficiently efficient to properly repair the test network, the proof-of-authority scheme.

Note that Parity does have a PoA implementation, although it looks more complicated than needed, there is not much protocol documentation, but it's hard to see it can be played with other customers. I welcome them to give me more feedback on this proposal based on their experience.

The main design goal of the PoA protocol described here is to implement and embed any existing Ethereum client should be very simple, while allowing the use of existing synchronization technologies (fast, easy, and distorted) without the need for client developers. Add custom logic to critical software.

## PoA101

For those who don't realize how PoA works, this is a very simple agreement, rather than the miners competing to solve a difficult problem, and the signatory can decide whether to create a new block at any time.

The challenge revolves around how to control the mining frequency, how to distribute the load (and opportunity) between different signers, and how to dynamically adjust the signer list. The next section defines a recommended protocol for handling all of these scenarios.

## Rinkeby proof-of-authority

In general, there are two ways to synchronize blockchains:

- The traditional approach is to start all blocks and tighten the transactions one by one. This approach has been tried and has proven to be very computationally intensive in a complex network such as Ethereum. The other is to download only the blockchains and verify their validity, after which you can download an arbitrary recent state from the network and check the nearest header.
- The PoA solution is based on the idea that blocks can only be completed by trusted signers. Therefore, each block (or header) seen by the client can match the list of trusted signers. The challenge here is how to maintain a list of authorized signers that can be changed in time? The obvious answer (stored in the Ethereum contract) is also the wrong answer: it is inaccessible during fast synchronization.

**The protocol that maintains the list of authorized signers must be fully contained in the block header.**

The next obvious idea is to change the structure of the block header so that you can abandon the concept of PoW and introduce new fields to cater to the voting mechanism. This is also the wrong answer: Changing such a core data structure in multiple implementations will be a nightmare for development, maintenance, and security.

**The protocol that maintains the list of authorized signers must be fully adapted to the current data model.**

So, according to the above, we can't use the EVM to vote, but have to resort to the block header. And we can't change the block header field and have to resort to the currently available fields. There are not many choices.

### Use some other fields of the block header to implement voting and signature

The most obvious field currently used only as interesting metadata is the 32-byte ExtraData portion of the block header. Miners usually put their clients and versions there, but some people fill them with additional "information." The protocol will extend this field to add 65 bytes to store the miner's KEC signature. This will allow anyone who gets a block to verify it based on the list of authorized signers. At the same time it also invalidates the field of the miner address in the block header.

Note that changing the length of the block header is a non-intrusive operation because all code (such as RLP encoding, hashing) is agnostic, so the client does not need custom logic.

The above is enough to verify a chain, but how do we update a dynamic list of signers. The answer is that we can re-use the newly obsolete miner field beneficiary and the PoA obsolete nonce field to create a voting protocol:

- In a regular block, both fields will be set to zero.
- If the signer wishes to make changes to the list of authorized signers, it will:
  - Set the miner field **beneficiary** to the signer who wishes to vote
  - Set the **nonce** to 0 or 0xff ... f to vote for adding or kicking out

Clients of any synchronization chain can "count" votes during block processing and maintain a dynamic list of authorized signers by ordinary voting. The initial set of signers is provided by the parameters of the genesis block (to avoid the complexity of deploying the "initial voter list" contract in the initial state).

To avoid having an infinite window to count votes and to allow regular elimination of stale proposals, we can reuse ethash's concept epoch, and each epoch conversion will refresh all pending votes. In addition, these epoch conversions can also be used as stateless checkpoints that contain a list of currently authorized signers within the header's extra data. This allows the client to synchronize based only on the checkpoint hash without having to replay all the votes made on the chain. It also allows the complete definition of the blockchain with the genesis block that contains the initial signer.

### Attack vector: malicious signer

It may happen that a malicious user is added to the list of signers or the signer's key/machine is compromised. In this case, the agreement needs to be able to withstand restructuring and spam. The proposed solution is that given a list of N authorized signers, any signer may only fill 1 block per K. This ensures that damage is limited and the remaining miners can cast malicious users.

## Attack vector: review signer

Another interesting attack vector is if a signer (or a group of signers) tries to check out the blocks that removed them from the authorization list. To solve this problem, we limit the minimum frequency allowed by the signer to N / 2. This ensures that the malicious signer needs to control at least 51% of the signed account, in which case the game will not be able to proceed anyway.

## Attack vector: spammer signer

The final small attack vector is that malicious signers inject new voting suggestions into each block. Since the node needs to count all votes to create an actual list of authorized signers, they need to track all votes by time. There is no limit to the voting window, which may grow slowly, but it is unlimited. The solution is to place a W block's moving w