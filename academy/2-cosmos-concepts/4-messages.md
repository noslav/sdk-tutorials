---
title: "Messages"
order: 5
description: Introduction to MsgService and the flow of messages
tags: 
  - concepts
  - cosmos-sdk
---

# Messages

<HighlightBox type="prerequisite">

It is recommended to take a look at the following previous sections to better understand messages:

* [A Blockchain App Architecture](./1-architecture.md)
* [Accounts](./2-accounts.md)
* [Transactions](./3-transactions.md)

</HighlightBox>

<HighlightBox type="learning">

In this section, you will take a closer look at messages, `Msg`. At the end of the section, you can find a code example that illustrates message creation and the inclusion of messages in transactions for your checkers blockchain.
<br/><br/>
Understanding `Msg` will help you prepare for the next section, on [modules in the Cosmos SDK](./5-modules.md), as messages are a primary object handled by modules.

</HighlightBox>

Messages are one of two primary objects handled by a module in the Cosmos SDK. The other primary object handled by modules is queries. While messages inform the state and have the potential to alter it, queries inspect the module state and are always read-only.

In the Cosmos SDK, a **transaction** contains **one or more messages**. The module processes the messages after the transaction is included in a block by the consensus layer.

<ExpansionPanel title="Signing a message">

Remember from the [previous section on transactions](./3-transactions.md) that transactions must be signed before a validator includes them in a block. Every message in a transaction must be signed by the addresses as specified by `GetSigners`.
<br/><br/>
The Cosmos SDK currently allows signing transactions with either `SIGN_MODE_DIRECT` or `SIGN_MODE_LEGACY_AMINO_JSON` methods.
<br/><br/>
When an account signs a message it signs an array of bytes. This array of bytes is the outcome of serializing the message. For the signature to be verifiable at a later date, this conversion needs to be deterministic. For this reason, you define a canonical bytes-representation of the message, typically with the parameters ordered alphabetically.

</ExpansionPanel>

## Messages and the transaction lifecycle

Transactions containing one or more valid messages are serialized and confirmed by CometBFT. As you might recall, CometBFT is agnostic to the transaction interpretation and has absolute finality. When a transaction is included in a block, it is confirmed and finalized with no possibility of chain re-organization or cancellation.

The confirmed transaction is relayed to the Cosmos SDK application for interpretation. Each message is routed to the appropriate module via `BaseApp` using `MsgServiceRouter`. `BaseApp` decodes each message contained in the transaction. Each module has its own `MsgService` that processes each received message.

## `MsgService`

Although it is technically feasible to proceed to create a novel `MsgService`, the recommended approach is to define a Protobuf `Msg` service. Each module has exactly one Protobuf `Msg` service defined in `tx.proto` and there is an RPC service method for each message type in the module. The Protobuf message service implicitly defines the interface layer of the state, mutating processes contained within the module.

How does all of this translate into code? Here is an example `MsgService` from the [`bank` module](https://docs.cosmos.network/v0.46/modules/bank/):

```protobuf
// Msg defines the bank Msg service.
service Msg {
  // Send defines a method for sending coins from one account to another account.
  rpc Send(MsgSend) returns (MsgSendResponse);

  // MultiSend defines a method for sending coins from some accounts to other accounts.
  rpc MultiSend(MsgMultiSend) returns (MsgMultiSendResponse);
}
```

In this example:

* Each `Msg` service method has exactly **one argument**, such as `MsgSend`, which must implement the `sdk.Msg` interface and a Protobuf response.
* The **standard naming convention** is to call the RPC argument `Msg<service-rpc-name>` and the RPC response `Msg<service-rpc-name>Response`.

## Client and server code generation

The Cosmos SDK uses Protobuf definitions to generate client and server code:

* The `MsgServer` interface defines the server API for the `Msg` service. Its implementation is described in the [`Msg` services documentation](https://docs.cosmos.network/main/building-modules/msg-services.html).
* Structures are generated for all RPC requests and response types.

<HighlightBox type="docs">

If you want to dive deeper when it comes to messages, the `Msg` service, and modules, see:

* The Cosmos SDK documentation on [`Msg` service](https://docs.cosmos.network/main/building-modules/msg-services.html).
* The Cosmos SDK documentation on messages and queries, addressing how to define messages using `Msg` services - [Amino `LegacyMsg`](https://docs.cosmos.network/main/building-modules/messages-and-queries.html#legacy-amino-legacymsgs).

</HighlightBox>

## Code example

<ExpansionPanel title="Show me some code for my checkers blockchain - Including messages">

In the [previous](./3-transactions.md) design exercise's code examples, the ABCI application was aware of a single transaction type: that of a checkers move with four `int` values. With multiple games, this is no longer sufficient. Additionally, you need to conform to the SDK's way of handling `Tx`, which means **creating messages that are then included in a transaction**.
<br/>

If you want the guided coding exercise instead of design and implementation considerations, see the links at the bottom of the page.

<br/><br/>
**What you need**

Begin by describing the messages you need for your checkers application to have a solid starting point before diving into the code:

1. In the former _Play_ transaction, your four integers need to move from the transaction to an `sdk.Msg`, wrapped in said transaction. Four flat `int` values are no longer sufficient, as you need to follow the `sdk.Msg` interface, identify the game for which a move is meant, and distinguish a move message from other message types.
2. You need to add a message type for creating a new game. When this is done, a player can create a new game which mentions other players. A generated (possibly) ID identifies this newly created game and is returned to the message creator.
3. It would be a good feature for the other person to be able to reject the challenge. This would have the added benefit of clearing the state of stale, unstarted games.

**How to proceed**

Focus on the messages around the **game creation**. There is no single true way of deciding what goes into your messages. The following is one reasonable example.

1. The message itself is structured like this:

    ```go
    type MsgCreateGame struct {
        Creator string
        Black   string
        Red     string
    }
    ```

    Note that `Creator` contains the address of the message signer.

2. The corresponding response message would then be:

    ```go
    type MsgCreateGameResponse struct {
        GameIndex string
    }
    ```

    The idea here is that when the creator does not know the ID of the game that will be created, the ID needs to be returned.

With the messages defined, you need to declare how the message should be handled. This involves:

1. Describing how the messages are serialized.
2. Writing the code that handles the message and places the new game in the storage.
3. Putting hooks and callbacks at the right places in the general message handling.

Thinking from design to implementation, Ignite CLI can help you create these elements, plus the `MsgCreateGame` and `MsgCreateGameResponse` objects, with this command:

```sh
$ ignite scaffold message createGame black red \
    --module checkers \
    --response gameIndex
```

<HighlightBox type="info">

Ignite CLI creates a variety of other files. See [Run Your Own Cosmos Chain](/hands-on-exercise/1-ignite-cli/index.md) for details, and to make additions to existing files.

</HighlightBox>

_**A sample of things Ignite CLI did for you**_

Ignite CLI significantly reduces the amount of work a developer has to do to build an application with the Cosmos SDK. Among others, it assists with:

1. Getting the signer, the `Creator`, of your message:

    ```go
    func (msg *MsgCreateGame) GetSigners() []sdk.AccAddress {
        creator, err := sdk.AccAddressFromBech32(msg.Creator)
        if err != nil {
           panic(err)
       }
       return []sdk.AccAddress{creator}
    }
    ```

    Where `GetSigners` is [a requirement of `sdk.Msg`](https://github.com/cosmos/cosmos-sdk/blob/1dba673/types/tx_msg.go#L21).

2. Making sure the message's bytes to sign are deterministic:

    ```go
    func (msg *MsgCreateGame) GetSignBytes() []byte {
        bz := ModuleCdc.MustMarshalJSON(msg)
        return sdk.MustSortJSON(bz)
    }
    ```

3. Adding a callback for your new message type in your module's message handler `x/checkers/handler.go`:

    ```go
    ...
    switch msg := msg.(type) {
        ...
        case *types.MsgCreateGame:
            res, err := msgServer.CreateGame(sdk.WrapSDKContext(ctx), msg)
            return sdk.WrapServiceResult(ctx, res, err)
        ...
    }
    ```

4. Creating an empty shell of a file (`x/checkers/keeper/msg_server_create_game.go`) for you to include your code, and the response message:

    ```go
    func (k msgServer) CreateGame(goCtx context.Context, msg *types.MsgCreateGame) (*types.MsgCreateGameResponse, error) {
        ctx := sdk.UnwrapSDKContext(goCtx)

        // TODO: Handling the message
        _ = ctx

        return &types.MsgCreateGameResponse{}, nil
    }
    ```

    Ignite CLI is opinionated in terms of which files it creates to separate which concerns. If you are not using it, you are free to create the files you want.

**What is left to do?**

Your work is mostly done. You want to create the specific game creation code to replace `// TODO: Handling the message`. For this, you need to:

1. Decide how to create a new and unique game ID: `newIndex`.

    <HighlightBox type="info">

    For more details, and to avoid diving too deep in this section, see:
    
    * [Run Your Own Cosmos Chain](/hands-on-exercise/1-ignite-cli/index.md) to start the guided coding exercise from scratch,
    * [Create and Save a Game Properly](/hands-on-exercise/1-ignite-cli/5-create-handling.md) to jump straight where you handle the new message.
      
    </HighlightBox>

2. Extract and verify addresses, such as:

    ```go
    black, err := sdk.AccAddressFromBech32(msg.Black)
    if err != nil {
        return nil, errors.New("invalid address for black")
    }
    ```

3. Create a game object with all required parameters - see the [modules section](./5-modules.md) for the declaration of this object:

    ```go
    storedGame := {
        Creator:   creator,
        Index:     newIndex,
        Game:      rules.New().String(),
        Black:     black,
        Red:       red,
    }
    ```

4. Send the game object to storage - see the [modules section](./5-modules.md) for the declaration of this function:

    ```go
    k.Keeper.SetStoredGame(ctx, storedGame)
    ```

5. And finally, return the expected message:

    ```go
    return &types.MsgCreateGameResponse{
        GameIndex: newIndex,
    }, nil
    ```

<HighlightBox type="remember">

Remember, as a part of good design practice:

* If you encounter an internal error (one that denotes an error in logic or catastrophic failure), you should `panic("This situation should not happen")`.
* If you encounter a user or _regular_ error (like a user not having enough funds), you should return a regular `error`.

</HighlightBox>

**The other messages**

You can also implement other messages:

1. The **play message**, which means implicitly accepting the challenge when playing for the first time. If you create it with Ignite CLI, use:

    ```sh
    $ ignite scaffold message playMove gameIndex fromX:uint fromY:uint toX:uint toY:uint --module checkers --response capturedX:int,capturedY:int,winner
    ```

    This generates, among others, the object files, callbacks, and a new file for you to write your code:

    ```go
    func (k msgServer) PlayMove(goCtx context.Context, msg *types.MsgPlayMove) (*types.MsgPlayMoveResponse, error) {
        ctx := sdk.UnwrapSDKContext(goCtx)

        // TODO: Handling the message
        _ = ctx

        return &types.MsgPlayMoveResponse{}, nil
    }
    ```

    To jump straight to the corresponding part of the coding exercise, head to [Add a Way to Make a Move](/hands-on-exercise/1-ignite-cli/6-play-game.md).

2. The **reject message**, which should be valid only if the player never played any moves in this game.

    ```sh
    $ ignite scaffold message rejectGame gameIndex --module checkers
    ```

    This generates, among others:

    ```go
    func (k msgServer) RejectGame(goCtx context.Context, msg *types.MsgRejectGame) (*types.MsgRejectGameResponse, error) {
        ctx := sdk.UnwrapSDKContext(goCtx)

        // TODO: Handling the message
        _ = ctx

        return &types.MsgRejectGameResponse{}, nil
    }
    ```

    This message is not implemented in the coding exercise as it is not necessary if you implement an expiration.

**Other considerations**

What would happen if one of the two players has accepted the game by playing, but the other player has neither accepted nor rejected the game? You can address this scenario by:

* Having a timeout after which the game is canceled.
* Keeping an index as a First-In-First-Out (FIFO) list, or a list of unstarted games ordered by their cancellation time, so that this automatic trigger does not consume too many resources.

What would happen if a player stops taking turns? To ensure functionality for your checkers application, you can consider:

* Having a timeout after which the game is forfeited. You could also automatically charge the forgetful player, if and when you implement a wager system. For the guided coding exercise on this part, head straight to [Keep an Up-To-Date Game Deadline](/hands-on-exercise/2-ignite-cli-adv/1-game-deadline.md).
* Keeping an index of games that could be forfeited. If both timeouts are the same, you can keep a single FIFO list of games so you can clear them from the top of the list as necessary. For the guided coding exercise on this part, head straight to [Put Your Games in Order](/hands-on-exercise/2-ignite-cli-adv/3-game-fifo.md).
* Handling the cancelation in ABCI's `EndBlock` (or rather its equivalent in the Cosmos SDK) without any of the players having to trigger the cancelation. For the guided coding exercise on this part, head straight to [Auto-Expiring Games](/hands-on-exercise/2-ignite-cli-adv/4-game-forfeit.md).

In general terms, you could add `timeout: Timestamp` to your `StoredGame` and update it every time something changes in the game. You can decide on a maximum delay, for example _one day_.

<HighlightBox type="info">

There are no _open_ challenges, meaning a player cannot create a game where the second player is unknown until someone steps in. Therefore, player matching is left outside of the blockchain. The enterprising student can incorporate it inside the blockchain by changing the necessary models.

</HighlightBox>

</ExpansionPanel>

<HighlightBox type="tip">

If you would like to get started on building your own checkers game, you can go straight to the main exercise in [Run Your Own Cosmos Chain](/hands-on-exercise/1-ignite-cli/index.md) to start from scratch.

More specifically, you can jump to:

* [Create Custom Messages](/hands-on-exercise/1-ignite-cli/4-create-message.md) to see how to simply create the `MsgCreateGame`,
* [Create and Save a Game Properly](/hands-on-exercise/1-ignite-cli/5-create-handling.md) to see how to handle `MsgCreateGame`,
* [Add a Way to Make a Move](/hands-on-exercise/1-ignite-cli/6-play-game.md) for the same but with `MsgPlayMove`.

</HighlightBox>

<HighlightBox type="synopsis">

To summarize, this section has explored:

* Messages, one of two primary objects handled by a module in the Cosmos SDK, which inform the state and have the potential to alter it.
* How one or more messages form a transaction in the Cosmos SDK, and messages are only processed after a transaction is signed by a validator and included in a block by the consensus layer.
* An example of more complex message handling capabilities related to the checkers game blockchain. 

</HighlightBox>

<!--## Next up

Look at the above code example to get a better sense of how theory translates into development. If you feel ready to dive into the next main concept of the Cosmos SDK, you can go directly to the [next section](./5-modules.md) to learn more about modules.-->
