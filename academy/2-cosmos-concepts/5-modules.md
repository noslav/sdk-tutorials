---
title: "Modules"
order: 6
description: Core Cosmos SDK modules and their components
tags: 
  - concepts
  - cosmos-sdk
---

# Modules

<HighlightBox type="prerequisite">

Review the following sections to better understand modules in the Cosmos SDK:

* [Transactions](./3-transactions.md)
* [Messages](./4-messages.md)
* [Queries](./9-queries.md)

</HighlightBox>

<HighlightBox type="learning">

Modules are functional components that address application-level concerns such as token management or governance. The Cosmos SDK includes several ready-made modules so that application developers can focus on the truly unique aspects of their application.
<br/><br/>
A code example that illustrates module creation and an introduction to your checkers blockchain can be found at the end of this section.

</HighlightBox>

Each Cosmos chain is a purpose-built blockchain. Cosmos SDK modules define the unique properties of each chain. Modules can be considered state machines within the larger state machine. They contain the storage layout, or state, and the state transition functions, which are the message methods.

Modules define most of the logic of Cosmos SDK applications.

![Transaction message flow to modules](/academy/2-cosmos-concepts/images/message_processing.png)

When a transaction is relayed from the underlying CometBFT consensus engine, `BaseApp` decomposes the `Messages` contained within the transaction and routes messages to the appropriate module for processing. Interpretation and execution occur when the appropriate module message handler receives the message.

Developers compose together modules using the Cosmos SDK to build custom application-specific blockchains.

## Module scope

Modules include **core** functionality that every blockchain node needs:

* A boilerplate implementation of the Application Blockchain Interface (ABCI) that communicates with CometBFT.
* A general-purpose data store that persists the module state called `multistore`.
* A server and interfaces to facilitate interactions with the node.

Modules implement the majority of the application logic while the **core** attends to wiring and infrastructure concerns, and enables modules to be composed into higher-order modules.

A module defines a subset of the overall state, using:

* One or more keys or value stores, known as `KVStore`.
* A subset of message types that are needed by the application and do not exist yet.

Modules also define interactions with other modules that already exist.

<HighlightBox type="info">

Most of the work for developers involved in building a Cosmos SDK application consists of building custom modules required by their application that do not exist yet, and integrating them with modules that already exist into one coherent application. Existing modules can come either from the Cosmos SDK itself or from **third-party developers**. You can download these from an online module repository.

</HighlightBox>

## Module components

It is best practice to define a module in the `x/moduleName` folder. For example, the module called `Checkers` would go in `x/checkers`. If you look at the Cosmos SDK's base code, it also [defines its modules](https://github.com/cosmos/cosmos-sdk/tree/main/x) in an `x/` folder.

Modules implement several elements:

<Accordion :items="
    [
        {
            title: 'Interfaces',
            description: 'Interfaces facilitate communication between modules and the composition of multiple modules into coherent applications.'
        },
        {
            title: 'Protobuf',
            description: 'Protobuf provides one `Msg` service to handle messages and one gRPC `Query` service to handle queries.'
        },
        {
            title: 'Keeper',
            description: 'A Keeper is a controller that defines the state and presents methods for updating and inspecting the state.'
        }
    ]
"/>

### Interfaces

A module must implement **three application module interfaces** to be integrated with the rest of the application:

* **`AppModuleBasic`:** implements non-dependent elements of the module.
* **`AppModule`:** interdependent, specialized elements of the module that are unique to the application.
* **`AppModuleGenesis`:** interdependent, genesis/initialization elements of the module that establish the initial state of the blockchain at inception.

You define `AppModule` and `AppModuleBasic`, and their functions, in your module's `x/moduleName/module.go` file.

### Protobuf services

Each module defines two Protobuf services:

* **`Msg`:** a set of RPC methods related one-to-one to Protobuf request types, to handle messages.
* **`Query`:** gRPC query service, to handle queries.

<HighlightBox type="tip">

If this topic is new to you, read an introduction to [Protocol Buffers](https://www.ionos.com/digitalguide/websites/web-development/protocol-buffers-explained/).

</HighlightBox>

### `Msg` service

Regarding the `Msg` service, keep in mind:

* Best practice is to define the `Msg` Protobuf service in the `tx.proto` file.
* Each module should implement the `RegisterServices` method as part of the `AppModule` interface. This lets the application know which messages and queries the module can handle.
* Service methods should use a _keeper_, which encapsulates knowledge about the storage layout and presents methods for updating the state.

### gRPC `Query` service

For the gRPC `Query` service, keep in mind:

* Best practice is to define the `Query` Protobuf service in the `query.proto` file.
* It allows users to query the state using gRPC.
* Each gRPC endpoint corresponds to a service method, named with the `rpc` prefix inside the gRPC `Query` service.
* It can be configured under the `grpc.enable` and `grpc.address` fields in `app.toml`.

Protobuf generates a `QueryServer` interface containing all the service methods for each module. Modules implement this `QueryServer` interface by providing the concrete implementation of each service method in separate files. These implementation methods are the handlers of the corresponding gRPC query endpoints. This division of concerns across different files makes the setup safe from a re-generation of files by Protobuf.

<HighlightBox type="info">

[gRPC](https://grpc.io/) is a modern, open-source, high-performance framework that supports multiple languages. It is the recommended standard for external clients such as wallets, browsers, and backend services to interact with a node.

</HighlightBox>

gRPC-Gateway REST endpoints support external clients that may not wish to use gRPC. The Cosmos SDK provides a gRPC-gateway REST endpoint for each gRPC service.

<HighlightBox type="docs">

See the [gRPC-Gateway documentation](https://grpc-ecosystem.github.io/grpc-gateway/) for more on the gRPC-Gateway plugin.

</HighLightBox>

### Command-line commands

Each module defines commands for a command-line interface (CLI). Commands related to a module are defined in a folder called `client/cli`. The CLI divides commands into two categories: transactions and queries. These are the same as those which you defined in `tx.go` and `query.go` respectively.

### Keeper

Keepers are the gatekeepers to any stores in the module. It is mandatory to go through a module’s keeper to access a store. A keeper encapsulates the knowledge about the layout of the storage within the store and contains methods to update and inspect it. If you come from a module-view-controller (MVC) world, then it helps to think of the keeper as the controller.

![Keeper in a node](/academy/2-cosmos-concepts/images/keeper.png)

Other modules may need access to a store, but other modules are also potentially malicious or poorly written. For this reason, developers need to consider who and what should have access to their module stores. To prevent a module from randomly accessing another module at runtime, a module needs to declare its intent to use another module at construction. At this point, such a module is granted a runtime key that lets it access the other module. Only modules that hold this key to a store can access the store. This is part of what is called an **object-capability model**.

Keepers are defined in `keeper.go`. A keeper's type definition generally consists of keys to the module's own store in the `multistore`, references to the keepers of other modules, and a reference to the application's codec.

## Core modules

The Cosmos SDK includes a set of core modules that address common concerns with well-solved, standardized implementations. Core modules address application needs such as tokens, staking, and governance.

Core modules offer several advantages over ad-hoc solutions:

* Standardization is established early, which helps ensure good interoperability with wallets, analytics, other modules, and other Cosmos SDK applications.
* Duplication of effort is significantly reduced because application developers focus on what is unique about their application.
* Core modules are working examples of Cosmos SDK modules that provide strong hints about suggested structure, style, and best practices.

Developers create coherent applications by selecting and composing core modules first and then implementing the custom logic.

<HighlightBox type="tip">

Why not explore the [list of core modules and the application concerns they address](https://github.com/cosmos/cosmos-sdk/tree/main/x)?

</HighlightBox>

## Recommended folder structure

<HighlightBox type="info">

The following ideas are meant to be applied as suggestions. Application developers are encouraged to improve and contribute to the module structure and development design.

</HighlightBox>

### Structure

A typical Cosmos SDK module can be structured as follows:

1. The serializable data types and Protobuf interfaces:

```shell
proto
└── {project_name}
    └── {module_name}
        └── {proto_version}
            ├── {module_name}.proto
            ├── event.proto
            ├── genesis.proto
            ├── query.proto
            └── tx.proto
```

* `{module_name}.proto`: the module's common message type definitions.
* `event.proto`: the module's message type definitions related to events.
* `genesis.proto`: the module's message type definitions related to the genesis state.
* `query.proto`: the module's `Query` service and related message type definitions.
* `tx.proto`: the module's `Msg` service and related message type definitions.

2. Then the rest of the code elements:

```shell
x/{module_name}
├── client
│   ├── cli
│   │   ├── query.go
│   │   └── tx.go
│   └── testutil
│       ├── cli_test.go
│       └── suite.go
├── exported
│   └── exported.go
├── keeper
│   ├── genesis.go
│   ├── grpc_query.go
│   ├── hooks.go
│   ├── invariants.go
│   ├── keeper.go
│   ├── keys.go
│   ├── msg_server.go
│   └── querier.go
├── module
│   └── module.go
├── simulation
│   ├── decoder.go
│   ├── genesis.go
│   ├── operations.go
│   └── params.go
├── spec
│   ├── 01_concepts.md
│   ├── 02_state.md
│   ├── 03_messages.md
│   └── 04_events.md
├── {module_name}.pb.go
├── abci.go
├── codec.go
├── errors.go
├── events.go
├── events.pb.go
├── expected_keepers.go
├── genesis.go
├── genesis.pb.go
├── keys.go
├── msgs.go
├── params.go
├── query.pb.go
└── tx.pb.go
```

* `client/`: the module's CLI client functionality implementation and the module's integration testing suite.
* `exported/`: the module's exported types - typically interface types (see also the following note).
* `keeper/`: the module's `Keeper` and `MsgServer` implementations.
* `module/`: the module's `AppModule` and `AppModuleBasic` implementations.
* `simulation/`: the module's simulation package defines functions used by the blockchain simulator application (`simapp`).
* `spec/`: the module's specification documents outlining important concepts, state storage structure, and message and event type definitions.
* The root directory includes type definitions for messages, events, and genesis state, including the type definitions generated by Protocol Buffers:
    * `abci.go`: the module's `BeginBlocker` and `EndBlocker` implementations. This file is only required if `BeginBlocker` and/or `EndBlocker` need to be defined.
    * `codec.go`: the module's registry methods for interface types.
    * `errors.go`: the module's sentinel errors.
    * `events.go`: the module's event types and constructors.
    * `expected_keepers.go`: the module's expected other keeper interfaces.
    * `genesis.go`: the module's genesis state methods and helper functions.
    * `keys.go`: the module's store keys and associated helper functions.
    * `msgs.go`: the module's message type definitions and associated methods.
    * `params.go`: the module's parameter type definitions and associated methods.
    * `*.pb.go`: the module's type definitions generated by Protocol Buffers as defined in the respective `*.proto` files.

<HighlightBox type="info">

If a module relies on keepers from another module, the `exported/` code element expects to receive the keepers as interface contracts to avoid a direct dependency on the module implementing the keepers. However, these interface contracts can define methods that operate on (or return types that are specific to) the module that is implementing the keepers.
<br/><br/>
The interface types defined in `exported/` use canonical types that allow for the module to receive the interface contracts through the `expected_keepers.go` file. This pattern allows for code to remain [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) and also alleviates import cycle chaos.

</HighlightBox>

## Errors

Modules are encouraged to define and register their own errors to provide better context for failed messages or handler executions. Errors should be common or general errors, which can be further wrapped to provide additional specific execution context.

<HighlightBox type="docs">

For more details see the [Cosmos SDK documentation on errors when building modules](https://docs.cosmos.network/main/building-modules/errors.html).

</HighlightBox>

### Registration

Modules should define and register their custom errors in `x/{module}/errors.go`. Registration of errors is handled via the `types/errors` package.

Each custom module error must provide the codespace, which is typically the module name (for example, "distribution") and is unique per module, and a `uint32` code. The codespace and code together provide a globally unique Cosmos SDK error.

The only restrictions on error codes are the following:

* They must be greater than one, as a code value of one is reserved for internal errors.
* They must be unique within the module.

<HighlightBox type="info">

The Cosmos SDK provides a core set of common errors. These errors are defined in [`types/errors/errors.go`](https://github.com/cosmos/cosmos-sdk/blob/master/types/errors/errors.go).

</HighlightBox>

### Wrapping

The custom module errors can be returned as their concrete type, as they already fulfill the error interface. Module errors can be wrapped to provide further context and meaning to failed executions.

Regardless of whether an error is wrapped or not, the Cosmos SDK's errors package provides an API to determine if an error is of a particular kind via `Is`.

### ABCI

If a module error is registered, the Cosmos SDK errors package allows ABCI information to be extracted through the `ABCIInfo` API. The package also provides `ResponseCheckTx` and `ResponseDeliverTx` as auxiliary APIs to automatically get `CheckTx` and `DeliverTx` responses from an error.

## Code example

<ExpansionPanel title="Show me some code for my checkers blockchain">

Now your application is starting to take shape.
<br/><br/>
**The `checkers` module**

When you create your checkers blockchain application, you ought to include a majority of the standard modules like `auth`, `bank`, and so on. With the Cosmos SDK boilerplate in place, the _checkers part_ of your checkers application will most likely reside in a single `checkers` module. This is the module that you author.
<br/><br/>
**Game wager**

Earlier the goal was to let players play with _money_. With the introduction of modules like `bank` you can start handling that.
<br/><br/>
The initial ideas are:

* The wager amount is declared when creating a game.
* Each player is billed the amount when making their first move, which is interpreted as "challenge accepted". The amount should not be deducted on the game creation. If the game times out, the first player gets refunded.
* Subsequent moves by a player do not cost anything.
* If a game ends in a win or times out on a forfeit, the winning player gets the total wager amount.
* If a game ends in a draw, then both players get back their amount.

How would this look in terms of code? You need to add the wager to:

* The game:

    ```go
    type StoredGame struct {
        ...
        Wager uint64
    }
    ```

* The message to create a game:

    ```go
    type MsgCreateGame struct {
        ...
        Wager uint64
    }
    ```

**Wager payment**

Now you must decide how the tokens are moved. When a player accepts a challenge, the amount is deducted from that player's balance. But where does it go? You could burn the tokens and re-mint them at a later date, but this would make the total supply fluctuate wildly for no apparent benefit.
<br/><br/>
It is possible to transfer from a player to a module. The module acts as the escrow account for all games. So when playing for the first time, a player would:

```go
import (
    sdk "github.com/cosmos/cosmos-sdk/types"
    bankKeeper "github.com/cosmos/cosmos-sdk/x/bank/keeper"
)

wager := sdk.NewCoin("stake", sdk.NewInt(storedGame.Wager))
payment := sdk.NewCoins(wager)
var bank bankKeeper.Keeper
var ctx sdk.Context
var playerAddress sdk.AccAddress
err := bank.SendCoinsFromAccountToModule(ctx, playerAddress, "checkers", payment)
if err != nil {
    return errors.New("player cannot pay the wager")
}
```

`"stake"` identifies the likely name of the base token of your application, the token that is used with the consensus. Of course, you can choose to have the wager paid in any denomination, even one decided by players when creating the game. Conversely, when paying a winner you would have:

```go
amount := sdk.NewInt(storedGame.Wager).Mul(sdk.NewInt(2))
winnings := sdk.NewCoin("stake", amount)
payment := sdk.NewCoins(winnings)
var winnerAddress sdk.AccAddress
err := bank.SendCoinsFromModuleToAccount(ctx, "checkers", winnerAddress, payment)
if err != nil {
    panic("Cannot pay the winnings to winner")
}
```

<HighlightBox type="info">

Note that:

* It is a _standard_ error when the player cannot pay, which is _easily_ fixed by the player.
* It is a panic (an internal error) when the escrow account cannot pay, because if the escrow cannot pay it means there is a logic problem somewhere.

</HighlightBox>

</ExpansionPanel>

<HighlightBox type="tip">

If you want to go beyond the code samples in the expandable above and instead see in more detail how to define all this, go to [Run Your Own Cosmos Chain](/hands-on-exercise/1-ignite-cli/index.md).

More specifically, you can jump to:

* [Ignite CLI](/hands-on-exercise/1-ignite-cli/1-ignitecli.md) to create a new blockchain with your checkers module.
* The advanced [Handle Wager Payments](/hands-on-exercise/2-ignite-cli-adv/6-payment-winning.md) to have your checkers module access and use the bank module.
* [Add a leaderboard module](/hands-on-exercise/4-run-in-prod/3-add-leaderboard.md) and loosely couple it with another module with the use of the hooks pattern. It also leverages the module's `Params` construct.
* The IBC-advanced [Extend the Checkers Game With a Leaderboard](/hands-on-exercise/5-ibc-adv/6-ibc-app-checkers.md) to add a second custom IBC module to your checkers blockchain.
* The IBC-advanced [Create a Leaderboard Chain](/hands-on-exercise/5-ibc-adv/7-ibc-app-leaderboard.md) to create a new blockchain with a module, once again.

</HighlightBox>

<HighlightBox type="synopsis">

To summarize, this section has explored:

* How Cosmos SDK modules can be viewed as purpose-specific state machines that define the unique properties of the larger state machine that is each blockchain.
* How messages are decomposed from the incoming transaction containing them and routed to the appropriate module for processing.
* How all modules comprise three core functionalities: an implementation of ABCI to communicate with CometBFT; a general-purpose data store which persists the module state; and the server and interfaces which facilitate interactions with the node.
* How the majority of work for developers is in building custom modules that satisfy their unique needs, which are then integrated into a coherent application alongside existing modules from the Cosmos SDK of third-party developers.
* How the Cosmos SDK's set of core modules address common applications needs (such as tokens, staking, and governance) while providing useful benefits like standardization across the Ecosystem, less duplication of effort, and practical examples of effective structure, style, and best practices.
* How modules should ideally define and register their own set of errors (in addition to the Cosmos SDK's set of common errors), allowing developers to add context and meaning to failed executions.             

</HighlightBox>

<!--## Next up

Look at the above code example to see modules in practice, or go straight to the [next section](../2-cosmos-concepts/6-protobuf.md) for an introduction to Protobuf.-->
