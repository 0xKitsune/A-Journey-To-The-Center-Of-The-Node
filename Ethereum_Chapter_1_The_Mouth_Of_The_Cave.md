# Ethereum Chapter 1: The Mouth Of The Cave


Welcome to the mouth of the cave. As you may have learned from the "how blockchain works" link in the preface of this series, a blockchain is like a distributed database where any computer can run a program that keeps track of the database's state, validates changes to the state and can send transactions to update the state. The software required to keep track of the shared state is called a node client (often referred to as a "client"). When it comes to present day Ethereum, there are two different major moving parts that make up "the node". There is the consensus client (Lighthouse, Prysm, Nimbus, ect.) and the execution client (Geth, Erigon, Akula, ect.). In Ethereum, the execution client (also known as the Execution Engine, EL client or formerly the Eth1 client) listens to new transactions broadcasted in the network, executes them in EVM, and holds the latest state and database of all current Ethereum data. The consensus client (also known as the Beacon Node, CL client or formerly the Eth2 client) implements the proof-of-stake consensus algorithm, which enables the network to achieve agreement based on validated data from the execution client.

Node clients can be created by anyone and there is no restriction as to which client you can use. As long as your client follows the specifications in the Ethereum yellow paper for validating/updating the shared state, your client will work just as well as any other. This opens up the door for developers who are very familiar with node architecture to hyper tune their clients to fit their specific needs. 

Any computer that is running the necessary parts of a node client is often referred to as a "node". As mentioned previously, nodes are responsible for keeping track of data on the blockchain and sending transactions to the network. For example if you wanted to check how much Ether was in a specific account at the most recent block, you can send a request to the node which will then send you a response with the data that you requested. Similarly, if you wanted to send some Ether to another account you would create a transaction, sign it and then send it to the node to be broadcasted to the network. You can either run the node client on your local computer, or use a hosted node. This series will focus on locally hosted nodes, but if you would like to learn more about hosted nodes [you can do so here](https://ethereum.org/en/developers/docs/nodes-and-clients/nodes-as-a-service/).


There are a few different ways to sync an execution client that dictate what "type" of node you are running. There are full nodes, light nodes, and archive nodes. Much of the following content is from the [Go Ethereum Docs](https://geth.ethereum.org/docs/dapp/tracing), as well as [The Ethereum Docs](https://ethereum.org/en/developers/docs/nodes-and-clients/), which I have synthesized below.



<br>

## Full Node

Full nodes store the current block's state as well as the state of the previous 128 blocks and participate in validating newly added blocks. Full nodes can process transactions as well as query blockchain data. They have some access to historical data through "traces" (we will touch on traces in a second) but a full node is not always the best choice for some types of long range historical data. To run a full node on Ethereum mainnet, you will need the following hardware (this is only for Ethereum and full node requirements will vary greatly depending on the network):
- A fast Cpu with 4+ cores.
- 16 GB+ of RAM.
- A fast SSD drive with at least 500GB+ of space.
- 25 MBit/s+ bandwidth.



<br>


## Light Node
Instead of downloading every block, light nodes only download block headers. When any other information that is requested, the light node fowards the request to a full node. The light node can then independently verify the data they receive against the state roots in the block headers. Light nodes enable users to participate in the Ethereum network without the powerful hardware or high bandwidth required to run full nodes. There are also potential routes to providing light client data over the gossip network also known as the P2P (peer to peer) network. This is advantageous because the gossip network could support a network of light nodes without requiring full nodes to serve requests.

A light node for Ethereum mainnet only requires < 400MB of storage and can run on just about any machine including laptops, a Raspberry Pi, or even a toaster (probably).


<br>


## Archive Node

An archive node is very similar to a full node, with the only difference being that an archive node retains all historical data back to genesis. It can therefore trace arbitrary transactions at any point in the history of the chain. Tracing a single transaction requires re-executing all preceding transactions in the same block.

To run an archive node on Ethereum mainnet, you will need the following hardware (note that the blockchain is always growing and the necessary SSD drive space necessary for an archive node will increase over time as well.):
- A fast Cpu with 4+ cores.
- 16 GB+ of RAM.
- A fast SSD drive with at least ~12TB+ of space.
- 25 MBit/s+ bandwidth.



<br>


## Traces

While most people familiar with the EVM know what transactions are, not everyone knows what traces are. Traces are integral to how a node queries data.

Transactions executed in the block trigger smaller [atomic actions](https://news.ycombinator.com/item?id=14851610) that modify the blockchain's state, called "execution traces", or just "traces". In its simplest form, tracing a transaction entails sending a request to a node to re-execute the desired transaction and have it return the traces. Re-executing a transaction has a few prerequisites that need to be met. All historical state accessed by the transaction that you are trying to trace must be available including the account balance, the nonce, the bytecode of the "to" address in the transaction, the storage (ie. state) of the "to" address, as well as the bytecode and storage of all the internally invoked contracts. The node will also need access to block metadata (block timestamp, block difficulty, ect.) referenced during execution of the transaction. Lastly, the node needs access to the traces generated by all the preceding transactions executed in the same block before the transaction you are requesting. Some block explorers refer to traces as [internal transactions](https://etherscan.io/tx/0xf796d58b691c8ee75bb83c06e0e9a53ed052353523b532046fa3d606cc47344f#internal). There are four EVM opcodes that result in a trace, which are "CALL", "CREATE", "CREATE2" and "SELFDESTRUCT". Don't worry about EVM opcodes just yet, we will go into depth on this in a later chapter when we explore traces further.

This means there are limits on the transactions that can be traced imposed by the synchronization and pruning configuration (how many blocks your node keeps in cache). A full synced node retains the most recent 128 blocks in memory, so transactions in that range are always accessible. Full nodes also store occasional checkpoints back to genesis that can be used to rebuild the state at any point on-the-fly. This means older transactions can be traced but if there is a large distance between the requested transaction and the most recent checkpoint rebuilding the state can take a long time. Tracing a single transaction requires re-executing all preceding transactions in the same block and all preceding blocks until the previous stored snapshot.

A snap synced node holds the most recent 128 blocks in memory, so transactions in that range are always accessible. However, snap-sync only starts processing from a relatively recent block (as opposed to genesis for a full node). Between the initial sync block and the 128 most recent blocks, the node stores occasional checkpoints that can be used to rebuild the state on-the-fly. This means transactions can be traced back as far as the block that was used for the initial sync. Tracing a single transaction requires re-executing all preceding transactions in the same block, and all preceding blocks until the previous stored snapshot.

An archive node retains all historical data back to genesis. It can therefore trace arbitrary transactions at any point in the history of the chain. Tracing a single transaction requires re-executing all preceding transactions in the same block.

A light synced node retrieving data on demand can in theory trace transactions for which all required historical state is readily available in the network. This is because the data required to generate the trace is requested from an les-serving full node. In practice, data availability cannot be reasonably assumed.


<br>


## Pruning And Traces 
A "pivot block" is classified as the oldest block where state is immediately available, but where one block older is not immediately available (ie. 128 blocks ago for a full node). Let's walk through a quick example to demonstrate how tracing works for each syncing method, depending on if the block has been pruned or not. 

For this example, let's say someone requests tracing on a transaction in block `B`. We will denote the pivot block as the variable `P`.



### Full Node
If block `P` is older than block `B`, than the full node has immediate access to the state and can replay the transactions to derive the traces for the transaction in block `B`. However, if `B` is older than `P`, then the state cannot be regenerated without replaying the chain from checkpoints. Geth looks for the closes checkpoint `C`, before `B`, and then computes the state up to `B`. The default pivot block for a full node is 128 blocks ago. The same behavior applies to a full node that has been snap synced, with the difference being that the default pivot block is 64 blocks ago. With this being said, you can customize the depth of the pivot block by modifying the `TriesInMemory` variable in `core/blockchain.go` to have a customized number of cached states (but more on this in a later chapter).




### Archive Node
An archive node is quite simply a fully synced node without any pruning, meaning that the entire state history is always retained. This is happens when the `--gcmode=archive` flag is used. With an archive node, there is no pivot block since state is never deleted, which allows it to immediately fetch state/traces for at block depth.

The time taken to regenerate a specific state increases with the distance between P and B. If the distance between P and B is large, the regeneration time can be substantial.



**The ELI5 for the default behavior is:** A full node can access everything within 128 blocks quickly, but has to use checkpoints to regenerate state when accessing anything further, an archive node has immediate access to everything and a light client has access to block headers only and is reliant on a full node or archive node for all other data. The time taken to regenerate a specific state increases as the distance between `P` and `B` increases. If the distance between `P` and `B` is large, the regeneration time can be substantial.


<br>

Now that you know a little about how a node syncs and the different types of nodes, [feel free to sync a local node for yourself](https://ethereum.org/en/developers/docs/nodes-and-clients/run-a-node/)! If you do decide to sync a node, keep in mind that we will go over the exectuion and consensus client's configuration settings in detail, so don't feel like you have to understand it all at first glance. If you are ready to move on, head to the next chapter where we will dive into the execution client to learn more about what happens when you first start up the node.  

<hr>

### Next Chapter:

[Ethereum Chapter 2: Into The Belly Of the Execution Client]()