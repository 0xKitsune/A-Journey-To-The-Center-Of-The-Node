# Ethereum Chapter 2: Into The Belly Of the Execution Client


While there are many execution clients, we will use Geth for our walkthrough. Please note that this is not a reccomendation to use or not to use Geth. [Client diversity](https://ethereum.org/en/developers/docs/nodes-and-clients/#client-diversity) is very important to keep the network constantly running, so even though we will be using Geth for our walkthrough, consider using other clients as well.

When first looking at the [Geth codebase](https://github.com/0xKitsune/heavy-comment-go-ethereum), it might be a little confusing. Where is the entry point? One of the reasons that the codebase seems so large at first is that there are many different independent, executable binaries that exist under the umbrella of the Geth codebase. Let's take a very brief look at each of them. 

The `abigen` is a source code generator to convert Ethereum contract definitions into easy to use, compile-time type-safe Go packages. It operates on plain Ethereum contract ABIs with expanded functionality if the contract bytecode is also available. However, it also accepts Solidity source files, making development much more streamlined. Please see our Native DApps wiki page for details.

The `bootnode` module is a lightweight application used for the Node Discovery Protocol. Bootnodes do not sync chain data. Using a UDP-based Kademlia-like RPC protocol, Ethereum will connect and interrogate these bootnodes for the location of potential peers.

The `evm` module is a developer utility version of the EVM (Ethereum Virtual Machine) that is capable of running bytecode snippets within a configurable environment and execution mode. Its purpose is to allow isolated, fine-grained debugging of EVM opcodes (e.g. `evm --code 60ff60ff --debug`).

The `puppeth` module is an application binary used to maintain and install various helper tools for managing and deploying your private blockchain. For more info on this you can reference [this article](https://www.sitepoint.com/puppeth-introduction/).

The `rlpdump` module is a developer utility tool to convert binary RLP (Recursive Length Prefix) dumps. RLP data encoding is used when communicating data throughout the Ethereum network (e.g. `rlpdump --hex CE0183FFFFFFC4C304050583616263`).

The `clef` module can be used to sign transactions and data and is meant as a replacement for geth’s account management. This allows DApps not to depend on geth’s account management. When a DApp wants to sign data it can send the data to the signer, the signer will then provide the user with context and asks the user for permission to sign the data. If the users grants the signing request the signer will send the signature back to the DApp.


Finally, the [`geth` module](https://github.com/0xKitsune/heavy-comment-go-ethereum/tree/master/cmd/geth)! This is the entry point into the Ethereum client, which enables everything from node syncing, block validation, querying data, broadcasting transactions to other peers and more. This will be our focus for this part of the series. Now with that out of the way, lets dive into the `geth` module.


## The Geth Module
When the `geth` binary is run, a few things happen. First, variables responsible for initialization logic related to [node flags](https://github.com/0xKitsune/heavy-comment-go-ethereum/blob/master/cmd/geth/main.go#L61), [rpc flags](https://github.com/0xKitsune/heavy-comment-go-ethereum/blob/master/cmd/geth/main.go#L159) and [metric flags](https://github.com/0xKitsune/heavy-comment-go-ethereum/blob/master/cmd/geth/main.go#L189) are created.  Within these newly created variables, there is one variable called [`app`](https://github.com/0xKitsune/heavy-comment-go-ethereum/blob/master/cmd/geth/main.go#L59) which is responsible for keeping track of all the previously mentioned flags, as well as calling the `geth` function which starts the node (we will go more into depth on this in a second). Equally important are  [the `init()` function](https://github.com/0xKitsune/heavy-comment-go-ethereum/blob/master/cmd/geth/main.go#L207) which is run the first time that the `geth` module is initialized as well as the [`main()` function](https://github.com/ethereum/go-ethereum/blob/master/cmd/geth/main.go#L265). The `init()` function is responsible for initalizing the `app` variable defined earlier, binding the command and flag options that are available to `geth` to the application while the `main()` function runs the `app` with `app.Run()`, which starts the node. (If you are unfamiliar with the `init()` or `main()` functions in go, check out [this brief explanation](https://www.geeksforgeeks.org/main-and-init-function-in-golang/)). Taking a closer look at the `init()` function, we can see that first, the `app.Action` is set, which tells the program which function to run when the `app` is started. In our case, the `geth()` function is set as the app.Action. `geth()` is a function that acts as the main entry point into the node client. It creates a default node based on the command line arguments and runs it in blocking mode, waiting for it to be shut down. In the `init` function, the `app.Flags` (ie. the command-line parameters) are grouped together and `app.Before`, which runs before the main `app.Action`, is initialized with an anonymous function telling the program to call `MigrateGlobalFlags` which makes all global flag values available to the app context and aggregates flags that execute the same functionality. Finally, `app.After` is set, which gracefully closes down the node when the app is exited.

```go
func init() {
	// Initialize the CLI app and start Geth
	app.Action = geth
    ...
    ...
    ...
    app.Flags = utils.GroupFlags(
		nodeFlags,
		rpcFlags,
		consoleFlags,
		debug.Flags,
		metricsFlags,
	)

	app.Before = func(ctx *cli.Context) error {
		flags.MigrateGlobalFlags(ctx)
		return debug.Setup(ctx)
	}
	
	app.After = func(ctx *cli.Context) error {
		debug.Exit()
		prompt.Stdin.Close() // Resets terminal mode.
		return nil
	}
}
```

Following the `init()` function, we are met with a simple and straight forward [`main()`](https://github.com/ethereum/go-ethereum/blob/master/cmd/geth/main.go#L265) function which runs the `app`, passing in all the arguments from the command line so that the commands/flags can be properly initialized. `app.Run()` starts the node, but let's take a closer look at the init function to know exactly what is happening when this function is executed.
```go
func main() {
	if err := app.Run(os.Args); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
}
```
 When `app.Run()` is called, the method assigned to `app.Action` is executed, which if you recall, in our case is `geth()`. Let's dive into how this function actually works under the hood.

```go
// geth is the main entry point into the system if no special subcommand is ran.
// It creates a default node based on the command line arguments and runs it in
// blocking mode, waiting for it to be shut down.
func geth(ctx *cli.Context) error {
	if args := ctx.Args().Slice(); len(args) > 0 {
		return fmt.Errorf("invalid command: %q", args[0])
	}

	prepare(ctx)
	stack, backend := makeFullNode(ctx)
	defer stack.Close()

	startNode(ctx, stack, backend, false)
	stack.Wait()
	return nil
}
```

First, the function checks if there are any command line arguments that are left unprocessed (which means that they are unrecognized arguments), and prints an error with an invalid command message. Then, the [`prepare()` function](https://github.com/ethereum/go-ethereum/blob/master/cmd/geth/main.go#L274) checks which network you are running your node for and logs a few statements like `"Starting Geth on Ropsten testnet..."`, `"Starting Geth on Ethereum mainnet..."` or another network. Following this, the prepare function [initializes and runs metric collection](https://github.com/ethereum/go-ethereum/blob/master/cmd/geth/main.go#L334) if you have it enabled (feel free to refer back to [Geth Chapter 2](https://github.com/0xKitsune/Zero_to_100_Demystifying_Nodes/blob/main/Geth_Chapter_2_Configuring_And_Running_A_Node.md#metrics-and-stats-options) for available metrics to monitor).

Next we are met with our first major function called [`makeFullNode()`](https://github.com/ethereum/go-ethereum/blob/3b2a6b34d92b84f6bfab058a46d9fc73c21365c7/cmd/geth/config.go#L158) which initializes a [node](https://github.com/ethereum/go-ethereum/blob/master/node/node.go#L44) as well as a new [node backend](https://github.com/ethereum/go-ethereum/blob/master/internal/ethapi/backend.go#L42). Directly after the `stack` and `backend` variables are initialized, the node is started with the `startNode()` function. The `startNode()` function boots up all registered lifecycles, RPC services as well as p2p networking services, after which it unlocks any requested accounts, starts the RPC/IPC interfaces and starts the miner if mining is enabled. In essence, the `makeFullNode()` function is responsible for creating a new instance of a node and its backend, while the `startNode()` function runs the node which starts processes like node syncing, peer to peer communication, RPC/IPC communication and more. Both the `makeFullNode()` and `startNode()` functions are a rabbit hole of their own with many important details throughout each branch. Through the next chapters in this series, we will begin exploring each function in depth, starting with `makeFullNode()`. 




<br>

<hr>

### Next Chapter:
