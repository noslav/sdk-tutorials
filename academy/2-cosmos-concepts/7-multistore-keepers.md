---
title: "Multistore and Keepers"
order: 8
description: Store types, the AnteHandler, and keepers
tags: 
  - concepts
  - cosmos-sdk
---

# Multistore and Keepers

<HighlightBox type="prerequisite">

Look at the following sections before you begin:

* [Transactions](./3-transactions.md)
* [Messages](./4-messages.md)
* [Modules](./5-modules.md)
* [Protobuf](./6-protobuf.md)

</HighlightBox>

<HighlightBox type="learning">

Keepers are responsible for managing access to states defined by modules. Because states are accessed through keepers, they are an ideal place to ensure that invariants are enforced and security principles are always applied.
<br/><br/>
You can find a code example for your checkers blockchain at the end of the section that explores dealing with storage elements, message handling, and gas costs.

</HighlightBox>

A Cosmos SDK application on an application-specific blockchain usually consists of several modules attending to separate concerns. Each module has a state that is a subset of the entire application state.

The Cosmos SDK adopts an object-capabilities-based approach to help developers better protect their application from unwanted inter-module interactions. Keepers are at the core of this approach.

<HighlightBox type="info">

A keeper is a Cosmos SDK abstraction that manages access to the subset of the state defined by a module. All access to the module's data must go through the module's keeper.

</HighlightBox>

A keeper can be thought of as the literal gatekeeper of a module's stores. Each store defined within a module (typically an IAVL store) comes with a `storeKey` which grants unlimited access. The module's keeper holds this `storeKey`, which should otherwise remain unexposed, and defines methods for reading and writing to any store.

When a module needs to interact with the state defined in another module, it does so by interacting with the methods of the other module’s keeper. Developers control the interactions their module can have with other modules by defining methods and controlling access.

![Keepers in the Cosmos SDK](/academy/2-cosmos-concepts/images/keeper.png)

## Format

Keepers are generally defined in a `/keeper/keeper.go` file located in the module’s folder. The type keeper of a module is named simply `keeper.go` by convention. It usually follows the following structure:

```go
type Keeper struct {
    // Expected external keepers, if any

    // Store key(s)

    // codec
}
```

### Parameters

The following parameters are of importance concerning the type definitions of keepers in modules:

* An expected `keeper` is a keeper external to a module that is required by the internal keeper of said module. External keepers are listed in the internal keeper's type definition as interfaces. These interfaces are themselves defined in an `expected_keepers.go` file in the root of the module's folder. Interfaces are used to reduce the number of dependencies and to facilitate the maintenance of the module in this context.
* `storeKeys` grant access to any stores of the multistore managed by the module. They should always remain unexposed to external modules.
* `cdc` is the codec used to marshal and unmarshal structs to and from []byte. The `cdc` can be `codec.BinaryCodec`, `codec.JSONCodec`, or `codec.Codec` based on your requirements. Note that `code.Codec` includes the other interfaces. It can be either a proto or amino codec, as long as they implement these interfaces.

Each keeper has its own constructor function, which is called from the application's constructor function. This is where keepers are instantiated and where developers make sure to pass correct instances of the module's keepers to other modules that require them.

### Scope and best practices

Keepers primarily expose getter and setter methods for stores managed by their module. Methods should remain simple and strictly limited to getting or setting the requested value. Validity checks should already have been done with the `ValidateBasic()` method of the message and the `Msg` server before the keeper's methods are called.

The getter method will typically have the following signature:

```go
func (k Keeper) Get(ctx sdk.Context, key string) (value returnType, found bool)
```

The setter method will typically have the following signature:

```go
func (k Keeper) Set(ctx sdk.Context, key string, value valueType)
```

Keepers also should implement an iterator method with the following signature when appropriate:

```go
func (k Keeper) GetAll(ctx sdk.Context) (list []returnType)
```

<!-- Keepers also implement an iterator method: TODO: Iterator  method -->

## Store types

The Cosmos SDK offers different store types to work with. It is important to gain a good overview of the different store types available for development.

### `KVStore` and `Multistore` in the Interchain

Each Cosmos SDK application contains a state at its root, the `Multistore`. It is subdivided into separate compartments managed by each module in the application. The `Multistore` is a store of `KVStore`s that follows the [`Multistore interface`](https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc6/store/types/store.go#L104-L133).

The base `KVStore` and `Multistore` implementations are wrapped in extensions that offer specialized behavior. A [`CommitMultistore`](https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc6/store/types/store.go#L141-L184) is a `Multistore` with a committer. This is the main type of multistore used in the Cosmos SDK. The underlying `KVStore` is used primarily to restrict access to the committer.

The [`rootMulti.Store`](https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc6/store/rootmulti/store.go#L43-L61) is the go-to implementation of the `CommitMultiStore` interface. It is a base-layer multistore built around a database on top of which multiple `KVStore`s can be mounted. It is the default multistore store used in `BaseApp`.

### `CacheMultistore`

A [`cachemulti.Store`](https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc6/store/cachemulti/store.go#L17-L28) is used whenever the `rootMulti.Store` needs to be branched. `cachemulti.Store` branches all substores, creates a virtual store for each substore in its constructor, and holds them in `Store.stores`. This is used primarily to create an isolated store, typically when it is necessary to mutate the state but it might be reverted later.

`CacheMultistore` caches all read queries. `Store.GetKVStore()` returns the store from `Store.stores`, and `Store.Write()` recursively calls `CacheWrap.Write()` on all substores.

### Transient store

As the name suggests, `Transient.Store` is a `KVStore` that is discarded automatically at the end of each block. `Transient.Store` is a `dbadapter.Store` with a `dbm.NewMemDB()`. All `KVStore` methods are reused. A new `dbadapter.Store` is assigned when `Store.Commit()` is called, discarding the previous reference. Garbage collection is attended to automatically.

<HighlightBox type="docs">

Take a closer look at the [IAVL spec](https://github.com/cosmos/iavl/blob/v0.15.0-rc5/docs/overview.md) for when working with the IAVL store.

</HighlightBox>

<HighlightBox type="info">

The default implementation of `KVStore` and `CommitKVStore` is the `IAVL.Store`. The `IAVL.Store` is a self-balancing binary search tree that ensures get and set operations are `O(log n)`, where `n` is the number of elements in the tree.

</HighlightBox>

## Additional KVStore wrappers

In addition to the previous store types, there are a few others with more specific usage.

### GasKv store

Cosmos SDK applications use gas to track resource usage and prevent spam. The `GasKv.Store` is a `KVStore` wrapper that enables automatic gas consumption each time a read or write to the store is made. It is the solution of choice to track storage usage in Cosmos SDK applications.

`GasKv.Store` automatically consumes the appropriate amount of gas depending on the `Store.gasConfig` when methods of the parent `KVStore` are called. All `KVStores` are wrapped in `GasKv.Stores` by default when retrieved. This is done in the `KVStore()` method of the context. The default gas configuration is used in this case.

### TraceKv store

`tracekv.Store` is a `KVStore` wrapper which provides operation tracing functionalities over the underlying `KVStore`. It is applied automatically by the Cosmos SDK on all `KVStore`s if tracing is enabled on the parent `MultiStore`.
When each of the `KVStore` methods are called, `tracekv.Store` automatically logs `traceOperation` to the `Store.writer`. `traceOperation.Metadata` is filled with `Store.context` when it is not nil. `TraceContext` is a `map[string]interface{}`.

### Prefix store

`prefix.Store` is a `KVStore` wrapper which provides automatic key-prefixing functionalities over the underlying `KVStore`:

* When `Store.{Get, Set}()` is called, the store forwards the call to its parent with the key prefixed with the `Store.prefix`.
* When `Store.Iterator()` is called, it does not simply prefix the `Store.prefix` since it does not work as intended. Some of the elements are traversed even when they are not starting with the prefix in this case.

## `AnteHandler`

The `AnteHandler` is a special handler that implements the `AnteHandler` interface. It is used to authenticate a transaction before the transaction's internal messages are processed.

The `AnteHandler` is theoretically optional but still a very important component of public blockchain networks. It serves three primary purposes:

* It is a first line of defense against spam, and the second line of defense (after the mempool) against transaction replay with fees deduction and sequence checking.
* It performs preliminary stateful validity checks, like ensuring signatures are valid, or that a sender has enough funds to pay for fees.
* It plays a role in the incentivization of stakeholders via the collection of transaction fees.

`BaseApp` holds an `AnteHandler` as a parameter that is initialized in the application's constructor. The most widely used `AnteHandler` is the auth module.

<HighlightBox type="docs">

For more information on the subject, see the following resources:

* [Cosmos SDK documentation: Gas and Fees](https://github.com/cosmos/cosmos-sdk/blob/master/docs/docs/basics/04-gas-fees.md)
* [Cosmos SDK documentation: AnteHandler](https://github.com/cosmos/cosmos-sdk/blob/master/docs/docs/basics/04-gas-fees.md#antehandler)

</HighlightBox>

## Code example

<ExpansionPanel title="Show me some code for my checkers blockchain">

In the [Accounts section](./2-accounts.md), you were shown the elements of the stored game but not where this game is stored. This will now be explained.
<br/><br/>
**Game object in storage**

You need to decide under what structure you want to store a game in the storage. The Cosmos SDK partitions the global storage per module, with `checkers` being its own module. You need to take care of how to store games in the checkers module's corner of the key/value pair storage.
<br/><br/>
The first idea is to attribute a unique ID to a game, and to store the game value at that ID. For the sake of clarity, and to differentiate between other stored elements in the future, you add a prefix to each ID. The storage structure looks like this:

```go
// Pseudo-code
var globalStore sdk.KVStore
checkersStore := globalStore.getAtPrefix("checkers-")
gamesStore := checkersStore.getAtPrefix("games-")
storedGame := gamesStore.getAtPrefix(gameId)
// Or again in a different way
storedGame := globalStore.getAtPrefix("checkers-games-" + gameId)
```

This is pseudo-code because:

* Prefixes have to be `byte[]` instead of `string`. This is easily handled with a cast such as `[]byte("games-")`.
* The Cosmos SDK prevents you from directly accessing any random module's store, such that `globalStore.getAtPrefix("checkers-")` is not allowed and instead has to be accessed via a secret key.

Define the ID of the `StoredGame`. To return a single object, include `Index` in the object's value:

```go
type StoredGame struct {
    ...
    Index string
}
```

The Cosmos SDK provides much support; you only need to define the required prefixes in your corner of the storage:

```go
package types

const (
    StoredGameKeyPrefix = "StoredGame/value/"
)
```

This assists you with accessing a game:

```go
package keeper

import (
    "github.com/cosmos/cosmos-sdk/store/prefix"
    sdk "github.com/cosmos/cosmos-sdk/store/types"
    "github.com/alice/checkers/x/checkers/types"
)

func (k Keeper) GetStoredGame(ctx sdk.Context, gameId string) (storedGame types.StoredGame, found bool) {
    checkersStore := ctx.KVStore(k.storeKey)
    gamesStore := prefix.NewStore(checkersStore, []byte(types.StoredGameKeyPrefix))
    gameBytes := gamesStore.Get([]byte(gameId))
    if gameBytes == nil {
        return storedGame, false
    }
    k.cdc.MustUnmarshalBinaryBare(gameBytes, &storedGame)
    return storedGame, true
}
```

If you want to save a game:

```go
func (k Keeper) SetStoredGame(ctx sdk.Context, storedGame types.StoredGame) {
    checkersStore := ctx.KVStore(k.storeKey)
    gamesStore := prefix.NewStore(checkersStore, []byte(types.StoredGameKeyPrefix))
    gameBytes := k.cdc.MustMarshalBinaryBare(&storedGame)
    gamesStore.Set(byte[](storedGame.Index), gameBytes)
}
```

If you want to delete a stored game, you call `gamesStore.Delete(byte[](storedGame.Index))`.
<br/><br/>
The `KVStore` also allows you to obtain an iterator on a given prefix. You can list all stored games because they share the same prefix, which you do with:

```go
func (k Keeper) GetAllStoredGame(ctx sdk.Context) (list []types.StoredGame) {
    checkersStore := ctx.KVStore(k.storeKey)
    gamesStore := prefix.NewStore(checkersStore, []byte(types.StoredGameKeyPrefix))
    iterator := sdk.KVStorePrefixIterator(gamesStore, []byte{}) // []byte{} is an empty array

    defer iterator.Close() // Think of it as: try { everything below } finally { iterator.Close() }

    for ; iterator.Valid(); iterator.Next() {
        var storedGame types.StoredGame
        k.cdc.MustUnmarshalBinaryBare(iterator.Value(), &storedGame)
        list = append(list, storedGame)
    }

    return
}
```

Note the `MustMarshalBinaryBare` and `MustUnmarshalBinaryBare` functions in the previous `codec`. They need to be instructed on how to proceed with the marshaling. Protobuf handled this for you.

<HighlightBox type="tip">

See the [previous section on Protobuf](./6-protobuf.md) to explore how Protobuf deals with the marshaling.

</HighlightBox>

**Boilerplate, boilerplate everywhere!**

Note how the `Set`, `Get`, `Remove`, and `GetAll` functions shown previously look like boilerplate too. Do you have to redo these functions for every type? *No* - it was all created with this Ignite CLI command:

```sh
$ ignite scaffold map storedGame \
        board turn black red wager:uint \
        --module checkers \
        --no-message
```

`map` is the command that tells Ignite CLI to add an `Index` and store all elements under a map-like structure.

**Other storage elements**

How do you pick a value for the `Index` in `storedGame`? A viable idea is to keep a counter in storage for the next game. Unlike `StoredGame`, which is saved as a map, this `SystemInfo` object has to be at a unique location in the storage.
<br/><br/>
First define the object:

```go
package types

type SystemInfo struct {
    NextId uint64
}
```

Then define the key where it resides:

```go
const (
    SystemInfoKey = "SystemInfo-value-"
)
```

Then define the functions to get and set:

```go
func (k Keeper) SetSystemInfo(ctx sdk.Context, systemInfo types.SystemInfo) {
    systemInfoStore := prefix.NewStore(ctx.KVStore(k.storeKey), []byte(types.SystemInfoKey))
    nextBytes := k.cdc.MustMarshalBinaryBare(&systemInfo)
    systemInfoStore.Set([]byte{0}, nextBytes)
}
```

Remember that `SystemInfo` needs an initial value, which is the role of the genesis block definition:

```go
type GenesisState struct {
    SystemInfo SystemInfo
    StoredGameList []StoredGame
}
```

Now initialize:

```go
func DefaultGenesis() *GenesisState {
    return &GenesisState{
        SystemInfo: SystemInfo{
            NextId:  uint64(1), // Starts at 1
        },
        StoredGameList: []StoredGame{}, // Starts empty
    }
}
```

**What about message handling**

You go from the message to the game in storage with `MsgCreateGame`, which was defined in an earlier [section on messages](./4-messages.md). That is also the role of the keeper.
<br/><br/>
Define a handling function such as:

```go
func (k Keeper) CreateGame(goCtx context.Context, msg *types.MsgCreateGame) (*types.MsgCreateGameResponse, error) {
    ctx := sdk.UnwrapSDKContext(goCtx)

    // TODO: Handle the message

    return &types.MsgCreateGameResponse{}, nil
}
```

You now have all the pieces necessary to replace the `TODO`.

Get the next game ID:

```go
    systemInfo, found := k.GetSystemInfo(ctx)
    if !found {
        // Panic because this should never happen.
        panic("SystemInfo not found")
    }
    newIndex := strconv.FormatUint(systemInfo.NextId, 10)
```

Extract and sanitize the message elements:

```go
    creator, err := sdk.AccAddressFromBech32(msg.Creator)
    if err != nil {
        return nil, errors.New("creator address invalid")
    }
    black, err := sdk.AccAddressFromBech32(msg.Black)
    if err != nil {
        // Standard error because users can make mistakes.
        return nil, errors.New("black address invalid")
    }
    red, err := sdk.AccAddressFromBech32(msg.Red)
    if err != nil {
        return nil, errors.New("red address invalid")
    }
```

Create the stored game object:

```go
    storedGame := types.StoredGame{
        Creator: msg.Creator,
        Index:   newIndex,
        Board:   rules.New().String(),
        Turn:    "b",
        Black:   msg.Black,
        Red:     msg.Red,
        Wager:   msg.Wager,
    }
```

Save the stored game object in storage:

```go
    k.SetStoredGame(ctx, storedGame)
```

Prepare for the next created game:

```go
    systemInfo.NextId++
    k.SetSystemInfo(ctx, systemInfo)
```

Return the game ID for reference:

```go
    return &types.MsgCreateGameResponse{
        GameIndex: newIndex,
    }, nil
```

You would also do the same for `MsgPlayMoveResponse` and `MsgRejectGame`. Why not try it out as an exercise? See the bottom of the page for relevant links.
<br/><br/>
**More on game theory**

Some players may drop out from games, especially if they know they are about to lose. As the blockchain designer, you need to protect your blockchain, and in particular to avoid bloat, or locked tokens.

A good first point is to introduce a game deadline. This demonstrates how you would add a small feature to your existing blockchain:

```go
const (
    MaxTurnDuration = time.Duration(24 * 3_600 * 1000_000_000) // 1 day
    DeadlineLayout  = "2006-01-02 15:04:05.999999999 +0000 UTC"
)
type StoredGame struct {
    ...
    Deadline string
}
```

Set its initial value on creation:

```go
storedGame := types.StoredGame{
    ...
    Deadline: ctx.BlockTime().Add(types.MaxTurnDuration).UTC().Format(types.DeadlineLayout),
}
```

Update its value after a move:

```go
storedGame.Deadline = ctx.BlockTime().Add(types.MaxTurnDuration).UTC().Format(types.DeadlineLayout)
```

Extract and verify its value when necessary:

```go
deadline, err = time.Parse(DeadlineLayout, storedGame.Deadline)
if err != nil {
    panic(err)
}
if deadline.Before(ctx.BlockTime()) {
    // TODO
}
```

**How to expire games**

How can you know what games should be removed? Should you load _all_ games and filter for those that have expired? That would be extremely expensive, O(n) of the number of games in fact. This means the more successful your blockchain becomes, the slower it would run.

Better is to use a First-In-First-Out (FIFO) strategy, where fresh games are pushed back to the tail so that the head contains the next games to expire.
<br/><br/>
In the context of the Cosmos SDK, you need to keep track of where the FIFO starts and stops by saving the corresponding game IDs:

```go
const (
    NoFifoIndex = "-1"
)
type SystemInfo struct {
    ...
    FifoHeadIndex string
    FifoTailIndex string
}
```

Each game must know its relative position and the number of moves done, to assist the refunding logic on games with zero, one, or more than two moves:

```go
type StoredGame struct {
    ...
    MoveCount   uint64
    BeforeIndex string
    AfterIndex  string
}
```

Next, you need to code a regular FIFO, whereby:

* Games are sent to the back when created or played on.
* Games are removed from the FIFO when they are finished or time out.

**When to expire games**

See the next section about the [BaseApp](/academy/2-cosmos-concepts/8-base-app.md) for a possible solution.

</ExpansionPanel>

<HighlightBox type="tip">

If you want to go beyond out-of-context code samples like the above and see in more detail how to define these features, go to [Run Your Own Cosmos Chain](/hands-on-exercise/1-ignite-cli/index.md).
<br/><br/>
More precisely, you can jump to:

* [Store Object - Make a Checkers Blockchain](/hands-on-exercise/1-ignite-cli/3-stored-game.md) for general detail of how you handle your game in storage.
* [Create and Save a Game Properly](/hands-on-exercise/1-ignite-cli/5-create-handling.md) for how you would handle the game when it is being created.
* [Keep an Up-To-Date Game Deadline](/hands-on-exercise/2-ignite-cli-adv/1-game-deadline.md), where you add a small feature to your chain.
* [Put Your Games in Order](/hands-on-exercise/2-ignite-cli-adv/3-game-fifo.md) to see the implementation of the FIFO within the constraints of the store.
* [Store leaderboard candidates](/hands-on-exercise/4-run-in-prod/3-add-leaderboard.md) in the context's transient store.

</HighlightBox>

<HighlightBox type="synopsis">

To summarize, this section has explored:

* How each keeper manages access to the subset of the blockchain state that is a given module's state, which is at the core of the Cosmos SDK's object-capabilities-based approach to protecting applications from unwanted inter-module interactions.
* How each keeper holds a `storeKey` granting unlimited access to its module's data and defines how to read and write to any store, so when one module needs to interact with another it must follow the methods of the other module's keeper.
* How all Cosmos SDK applications contain a `Multistore` root state that is subdivided into compartments managed by each module and which stores all the `KVStore`s of the application's modules.
* How inclusion of the `AnteHandler` component is recommended to authenticate transactions before their internal messages are processed. It defends against spam and other wasteful transaction events, performs preliminary stateful validity checks, and is involved in collecting transaction fees.

</HighlightBox>

<!--## Next up

In the [next section](./8-base-app.md) you can find more information on `BaseApp` and its role in the Cosmos SDK.-->
