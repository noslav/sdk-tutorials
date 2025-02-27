---
title: "Create and Save a Game Properly"
order: 6
description: Create a proper game
tags: 
  - guided-coding
  - cosmos-sdk
---

# Create and Save a Game Properly

<HighlightBox type="prerequisite">

Make sure you have everything you need before proceeding:

* You have Go installed.
* You have the checkers blockchain codebase with `MsgCreateGame` created by Ignite CLI. If not, follow the [previous steps](./4-create-message.md) and check out [the relevant version](https://github.com/cosmos/b9-checkers-academy-draft/tree/create-game-msg).

</HighlightBox>

<HighlightBox type="learning">

In this section, you will:

* Make use of the rules of checkers.
* Update the message handler to create a game and return its ID.

</HighlightBox>

In the [previous section](./4-create-message.md), you added the message to create a game along with its serialization and dedicated gRPC function with the help of Ignite CLI.

However, it does not create a game yet because you have not implemented the message handling. How would you do this?

## Some initial thoughts

Dwell on the following questions to guide you in the exercise:

* How do you sanitize your inputs?
* How do you avoid conflicts with past and future games?
* How do you use your files that implement the checkers rules?

## Code needs

* No Ignite CLI is involved here, it is just Go.
* Of course, you need to know where to put your code - look for `TODO`.
* How would you unit-test this message handling?
* How would you use Ignite CLI to locally run a one-node blockchain and interact with it via the CLI to see what you get?

For now, do not bother with niceties like gas metering or event emission.

You must add code that:

* Verifies input sanity.
* Creates a brand new game.
* Saves it in storage.
* Returns the ID of the new game.

For input sanity, your code can only accept or reject a message. You cannot _fix_ a message, as that would change its content and break the signature. However, remember that your application is called via [ABCI's `CheckTx`](/academy/2-cosmos-concepts/1-architecture.md#checktx) for each transaction that it receives. It is at this point that your application can statelessly _sanitize_ inputs. For each message type, Ignite CLI isolates this concern into a `ValidateBasic` function:

```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/create-game-msg/x/checkers/types/message_create_game.go#L41-L47]
func (msg *MsgCreateGame) ValidateBasic() error {
    _, err := sdk.AccAddressFromBech32(msg.Creator)
    if err != nil {
        return sdkerrors.Wrapf(sdkerrors.ErrInvalidAddress, "invalid creator address (%s)", err)
    }
    return nil
}
```

It is in here that you can add further stateless checks on the message.

Ignite CLI isolated the _create a new game_ concern into a separate file, `x/checkers/keeper/msg_server_create_game.go`, for you to edit:

```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/create-game-msg/x/checkers/keeper/msg_server_create_game.go#L10-L17]
func (k msgServer) CreateGame(goCtx context.Context, msg *types.MsgCreateGame) (*types.MsgCreateGameResponse, error) {
    ctx := sdk.UnwrapSDKContext(goCtx)
    // TODO: Handling the message
    _ = ctx
    return &types.MsgCreateGameResponse{}, nil
}
```

Ignite CLI has conveniently created all the message processing code for you. You are only required to code the key features.

## Message verification coding steps

What is a well-formatted `MsgCreateGame`? Eventually, you want the black and red players to be able to play moves. They will send and sign transactions for that. So, at the very least, you can check that the addresses passed are valid:

```diff-go [https://github.com/cosmos/b9-checkers-academy-draft/blob/create-game-handler/x/checkers/types/message_create_game.go#L46-L55]
    func (msg *MsgCreateGame) ValidateBasic() error {
        _, err := sdk.AccAddressFromBech32(msg.Creator)
        if err != nil {
            return sdkerrors.Wrapf(sdkerrors.ErrInvalidAddress, "invalid creator address (%s)", err)
        }
+      _, err = sdk.AccAddressFromBech32(msg.Black)
+      if err != nil {
+          return sdkerrors.Wrapf(sdkerrors.ErrInvalidAddress, "invalid black address (%s)", err)
+      }
+      _, err = sdk.AccAddressFromBech32(msg.Red)
+      if err != nil {
+          return sdkerrors.Wrapf(sdkerrors.ErrInvalidAddress, "invalid red address (%s)", err)
+      }
        return nil
    }
```

You should not try to check whether they have enough tokens to play as that would be a stateful check. Stateful checks are handled as part of the message handling behind ACBI's [`DeliverTx`](/academy/2-cosmos-concepts/1-architecture.md#delivertx).

## Message handling coding steps

Given that you have already done a lot of preparatory work, what coding is involved? How do you replace `// TODO: Handling the message`?

1. First, `rules` represents the ready-made file with the imported rules of the game:

    ```diff-go [https://github.com/cosmos/b9-checkers-academy-draft/blob/create-game-handler/x/checkers/keeper/msg_server_create_game.go#L7]
        import (
            ...
    +      "github.com/alice/checkers/x/checkers/rules"
            ...
        )
    ```

2. Get the new game's ID with the [`Keeper.GetSystemInfo`](https://github.com/cosmos/b9-checkers-academy-draft/blob/create-game-handler/x/checkers/keeper/system_info.go#L17) function created by the `ignite scaffold single systemInfo...` command:

    ```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/create-game-handler/x/checkers/keeper/msg_server_create_game.go#L15-L19]
    systemInfo, found := k.Keeper.GetSystemInfo(ctx)
    if !found {
        panic("SystemInfo not found")
    }
    newIndex := strconv.FormatUint(systemInfo.NextId, 10)
    ```

    <HighlightBox type="info">

    You panic if you cannot find the `SystemInfo` object because there is no way to continue if it is not there. It is not like a user error, which would warrant returning an error.

    </HighlightBox>

3. Create the object to be stored:

    ```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/create-game-handler/x/checkers/keeper/msg_server_create_game.go#L21-L28]
    newGame := rules.New()
    storedGame := types.StoredGame{
        Index: newIndex,
        Board: newGame.String(),
        Turn:  rules.PieceStrings[newGame.Turn],
        Black: msg.Black,
        Red:   msg.Red,
    }
    ```

    <HighlightBox type="note">

    Note the use of:

    * The [`rules.New()`](https://github.com/cosmos/b9-checkers-academy-draft/blob/create-game-handler/x/checkers/rules/checkers.go#L122) command, which is part of the checkers rules file you imported earlier.
    * The string content of the `msg *types.MsgCreateGame`, namely `.Black` and `.Red`.

    Also note that you lose the information about the creator. If your design is different, you may want to keep this information.

    </HighlightBox>

4. Confirm that the values in the object are correct by checking the validity of the players' addresses:

    ```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/create-game-handler/x/checkers/keeper/msg_server_create_game.go#L30-L33]
    err := storedGame.Validate()
    if err != nil {
        return nil, err
    }
    ```

    `.Red`, and `.Black` need to be checked because they were copied as **strings**. You do not need to check `.Creator` because at this stage the message's signatures have been verified, and the creator is the signer.

    <HighlightBox type="note">

    Note that by returning an error, instead of calling `panic`, players cannot stall your blockchain. They can still spam but at a cost, because they will still pay the gas fee up to this point.

    </HighlightBox>

5. Save the `StoredGame` object using the [`Keeper.SetStoredGame`](https://github.com/cosmos/b9-checkers-academy-draft/blob/create-game-handler/x/checkers/keeper/stored_game.go#L10) function created by the `ignite scaffold map storedGame...` command:

    ```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/create-game-handler/x/checkers/keeper/msg_server_create_game.go#L35]
    k.Keeper.SetStoredGame(ctx, storedGame)
    ```

6. Prepare the ground for the next game using the [`Keeper.SetSystemInfo`](https://github.com/cosmos/b9-checkers-academy-draft/blob/create-game-handler/x/checkers/keeper/system_info.go#L10) function created by Ignite CLI:

    ```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/create-game-handler/x/checkers/keeper/msg_server_create_game.go#L36-L37]
    systemInfo.NextId++
    k.Keeper.SetSystemInfo(ctx, systemInfo)
    ```

7. Return the newly created ID for reference:

    ```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/create-game-handler/x/checkers/keeper/msg_server_create_game.go#L39-L41]
    return &types.MsgCreateGameResponse{
        GameIndex: newIndex,
    }, nil
    ```

You just handled the _create game_ message by actually creating the game.

## Unit tests

To test your additions to the message's `ValidateBasic`, you can simply add cases to the existing [`message_create_game_test.go`](https://github.com/cosmos/b9-checkers-academy-draft/blob/create-game-msg/x/checkers/types/message_create_game_test.go#L17-L28). You can verify that your additions have made the existing test fail:

<CodeGroup>

<CodeGroupItem title="Local" active>

```sh
$ go test github.com/alice/checkers/x/checkers/types
```

</CodeGroupItem>

<CodeGroupItem title="Docker">

```sh
$ docker run --rm -it \
    -v $(pwd):/checkers \
    -w /checkers \
    checkers_i \
    go test github.com/alice/checkers/x/checkers/types
```

</CodeGroupItem>

</CodeGroup>

This should return:

```txt
--- FAIL: TestMsgCreateGame_ValidateBasic (0.00s)
    --- FAIL: TestMsgCreateGame_ValidateBasic/valid_address (0.00s)
        message_create_game_test.go:37: 
                Error Trace:    /Users/alice/checkers/x/checkers/types/message_create_game_test.go:37
                Error:          Received unexpected error:
                            
                                github.com/alice/checkers/x/checkers/types.(*MsgCreateGame).ValidateBasic
                                        /Users/alice/checkers/x/checkers/types/message_create_game.go:50
                                github.com/alice/checkers/x/checkers/types.TestMsgCreateGame_ValidateBasic.func1
                                        /Users/alice/checkers/x/checkers/types/message_create_game_test.go:32
                                invalid black address (empty address string is not allowed): invalid address
                Test:           TestMsgCreateGame_ValidateBasic/valid_address
```

First, change the file's package to `types_test` for consistency:


```diff-go [https://github.com/cosmos/b9-checkers-academy-draft/blob/create-game-handler/x/checkers/types/message_create_game_test.go#L1-L7]
-  package types
+  package types_test

    import(
+      "github.com/b9lab/checkers/x/checkers/types"
    )
```

Then adjust the existing test cases and add to them:

```diff-go [https://github.com/cosmos/b9-checkers-academy-draft/blob/create-game-handler/x/checkers/types/message_create_game_test.go#L18-L52]
    {
-      name: "invalid address",
-      msg: MsgCreateGame{
+      name: "invalid creator address",
+      msg: types.MsgCreateGame{
            Creator: "invalid_address",
+          Black:   sample.AccAddress(),
+          Red:     sample.AccAddress(),
        },
        err: sdkerrors.ErrInvalidAddress,
    },
+  {
+      name: "invalid black address",
+      msg: types.MsgCreateGame{
+          Creator: sample.AccAddress(),
+          Black:   "invalid_address",
+          Red:     sample.AccAddress(),
+      },
+      err: sdkerrors.ErrInvalidAddress,
+  },
+  {
+      name: "invalid red address",
+      msg: types.MsgCreateGame{
+          Creator: sample.AccAddress(),
+          Black:   sample.AccAddress(),
+          Red:     "invalid_address",
+      },
+      err: sdkerrors.ErrInvalidAddress,
+  },
    {
-      name: "valid address",
-      msg: MsgCreateGame{
+      name: "valid addresses",
+      msg: types.MsgCreateGame{
            Creator: sample.AccAddress(),
+          Black:   sample.AccAddress(),
+          Red:     sample.AccAddress(),
        },
    },
```

Your tests on `/types` should now pass.

Moving on to the keeper, try the unit test you prepared in the previous section again:

<CodeGroup>

<CodeGroupItem title="Local" active>

```sh
$ go test github.com/alice/checkers/x/checkers/keeper
```

</CodeGroupItem>

<CodeGroupItem title="Docker">

```sh
$ docker run --rm -it \
    -v $(pwd):/checkers \
    -w /checkers \
    checkers_i \
    go test github.com/alice/checkers/x/checkers/keeper
```

</CodeGroupItem>

</CodeGroup>

This should fail with:

```txt
panic: SystemInfo not found [recovered]
        panic: SystemInfo not found
...
```

Your keeper was initialized with an empty genesis. You must fix that one way or another.

You can fix this by always initializing the keeper with the default genesis. However such a default initialization may not always be desirable. So it is better to keep this default initialization closest to the tests. Copy the `setupMsgServer` from [`msg_server_test.go`](https://github.com/cosmos/b9-checkers-academy-draft/blob/create-game-handler/x/checkers/keeper/msg_server_test.go#L13-L16) into your `msg_server_create_game_test.go`. Modify it to also return the keeper:

```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/create-game-handler/x/checkers/keeper/msg_server_create_game_test.go#L15-L19]
func setupMsgServerCreateGame(t testing.TB) (types.MsgServer, keeper.Keeper, context.Context) {
    k, ctx := keepertest.CheckersKeeper(t)
    checkers.InitGenesis(ctx, *k, *types.DefaultGenesis())
    return keeper.NewMsgServerImpl(*k), *k, sdk.WrapSDKContext(ctx)
}
```

<HighlightBox type="note">

Note the new import:

```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/create-game-handler/x/checkers/keeper/msg_server_create_game_test.go#L8]
import (
    "github.com/alice/checkers/x/checkers"
)
```

</HighlightBox>

Do not forget to replace `setupMsgServer(t)` with this new function everywhere in the file. For instance:

```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/create-game-handler/x/checkers/keeper/msg_server_create_game_test.go#L22]
msgServer, _, context := setupMsgServerCreateGame(t)
```

Run the tests again with the same command as before:

<CodeGroup>

<CodeGroupItem title="Local" active>

```sh
$ go test github.com/alice/checkers/x/checkers/keeper
```

</CodeGroupItem>

<CodeGroupItem title="Docker">

```sh
$ docker run --rm -it \
    -v $(pwd):/checkers \
    -w /checkers \
    checkers_i \
    go test github.com/alice/checkers/x/checkers/keeper
```

</CodeGroupItem>

</CodeGroup>

The error has changed to `Not equal`, and you need to adjust the expected value as per the default genesis:

```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/create-game-handler/x/checkers/keeper/msg_server_create_game_test.go#L29-L31]
require.EqualValues(t, types.MsgCreateGameResponse{
    GameIndex: "1",
}, *createResponse)
```

One unit test is good, but you can add more, in particular testing whether the values in storage are as expected when you create a single game:

```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/create-game-handler/x/checkers/keeper/msg_server_create_game_test.go#L34-L56]
func TestCreate1GameHasSaved(t *testing.T) {
    msgSrvr, keeper, context := setupMsgServerCreateGame(t)
    msgSrvr.CreateGame(context, &types.MsgCreateGame{
        Creator: alice,
        Black:   bob,
        Red:     carol,
    })
    systemInfo, found := keeper.GetSystemInfo(sdk.UnwrapSDKContext(context))
    require.True(t, found)
    require.EqualValues(t, types.SystemInfo{
        NextId: 2,
    }, systemInfo)
    game1, found1 := keeper.GetStoredGame(sdk.UnwrapSDKContext(context), "1")
    require.True(t, found1)
    require.EqualValues(t, types.StoredGame{
        Index: "1",
        Board: "*b*b*b*b|b*b*b*b*|*b*b*b*b|********|********|r*r*r*r*|*r*r*r*r|r*r*r*r*",
        Turn:  "b",
        Black: bob,
        Red:   carol,
    }, game1)
}
```

Or when you [create 3](https://github.com/cosmos/b9-checkers-academy-draft/blob/create-game-handler/x/checkers/keeper/msg_server_create_game_test.go#L102-L127) games. Other tests could include whether the _get all_ functionality works as expected after you have created [1 game](https://github.com/cosmos/b9-checkers-academy-draft/blob/create-game-handler/x/checkers/keeper/msg_server_create_game_test.go#L58-L74), or [3](https://github.com/cosmos/b9-checkers-academy-draft/blob/create-game-handler/x/checkers/keeper/msg_server_create_game_test.go#L181-L221), or if you create a game in a hypothetical [far future](https://github.com/cosmos/b9-checkers-academy-draft/blob/create-game-handler/x/checkers/keeper/msg_server_create_game_test.go#L223-L253). Also add games with [badly formatted](https://github.com/cosmos/b9-checkers-academy-draft/blob/create-game-handler/x/checkers/keeper/msg_server_create_game_test.go#L76-L87) or [missing input](https://github.com/cosmos/b9-checkers-academy-draft/blob/create-game-handler/x/checkers/keeper/msg_server_create_game_test.go#L89-L100).

## Interact via the CLI

Now you can also confirm that the transaction creates a game via the CLI. Start with:

<CodeGroup>

<CodeGroupItem title="Local" active>

```sh
$ ignite chain serve
```

</CodeGroupItem>

<CodeGroupItem title="Docker">

```sh
$ docker run --rm -it \
    --name checkers \
    -v $(pwd):/checkers \
    -w /checkers \
    checkers_i \
    ignite chain serve
```

</CodeGroupItem>

</CodeGroup>

Send your transaction as you did in the [previous section under "Interact via the CLI"](./4-create-message.md):

<CodeGroup>

<CodeGroupItem title="Local" active>

```sh
$ checkersd tx checkers create-game $alice $bob --from $alice --gas auto
```

</CodeGroupItem>

<CodeGroupItem title="Docker">

```sh
$ docker exec -it checkers \
    checkersd tx checkers create-game $alice $bob --from $alice --gas auto
```

</CodeGroupItem>

</CodeGroup>

A first good sign is that the output `gas_used` is slightly higher than it was before (`gas_used: "52498"`). After the transaction has been validated, confirm the current state.

Show the system info:

<CodeGroup>

<CodeGroupItem title="Local" active>

```sh
$ checkersd query checkers show-system-info
```

</CodeGroupItem>

<CodeGroupItem title="Docker">

```sh
$ docker exec -it checkers \
    checkersd query checkers show-system-info
```

</CodeGroupItem>

</CodeGroup>

This returns:

```txt
SystemInfo:
  nextId: "2"
```

List all stored games:

<CodeGroup>

<CodeGroupItem title="Local" active>

```sh
$ checkersd query checkers list-stored-game
```

</CodeGroupItem>

<CodeGroupItem title="Docker">

```sh
$ docker exec -it checkers \
    checkersd query checkers list-stored-game
```

</CodeGroupItem>

</CodeGroup>

This returns a game at index `1` as expected:

```txt
pagination:
  next_key: null
  total: "0"
storedGame:
- black: cosmos169mc8qqd6tlued00z23fs75tyecfcazpuwapc4
  board: '*b*b*b*b|b*b*b*b*|*b*b*b*b|********|********|r*r*r*r*|*r*r*r*r|r*r*r*r*'
  index: "1"
  red: cosmos10mqyvj55hm4wunsd62wprwfv9ehcerkfghcjfl
  turn: b
```

Show the new game alone:

<CodeGroup>

<CodeGroupItem title="Local" active>

```sh
$ checkersd query checkers show-stored-game 1
```

</CodeGroupItem>

<CodeGroupItem title="Docker">

```sh
$ docker exec -it checkers \
    checkersd query checkers show-stored-game 1
```

</CodeGroupItem>

</CodeGroup>

This returns:

```txt
storedGame:
  black: cosmos169mc8qqd6tlued00z23fs75tyecfcazpuwapc4
  board: '*b*b*b*b|b*b*b*b*|*b*b*b*b|********|********|r*r*r*r*|*r*r*r*r|r*r*r*r*'
  index: "1"
  red: cosmos10mqyvj55hm4wunsd62wprwfv9ehcerkfghcjfl
  turn: b
```

Now your game is in the blockchain's storage. Notice how `alice` was given the black pieces and it is already her turn to play. As a note for the next sections, this is how to understand the board:

```txt
*b*b*b*b|b*b*b*b*|*b*b*b*b|********|********|r*r*r*r*|*r*r*r*r|r*r*r*r*
                   ^X:1,Y:2                              ^X:3,Y:6
```

Or if placed in a square:

```txt
X 01234567
  *b*b*b*b 0
  b*b*b*b* 1
  *b*b*b*b 2
  ******** 3
  ******** 4
  r*r*r*r* 5
  *r*r*r*r 6
  r*r*r*r* 7
           Y
```

You can also get this in a one-liner:

<CodeGroup>

<CodeGroupItem title="Linux" active>

```sh
$ checkersd query checkers show-stored-game 1 --output json | jq ".storedGame.board" | sed 's/"//g' | sed 's/|/\n/g'
```

</CodeGroupItem>

<CodeGroupItem title="Docker">

```sh
$ docker exec -it checkers \
    bash -c "checkersd query checkers show-stored-game 1 --output json | jq \".storedGame.board\" | sed 's/\"//g' | sed 's/|/\n/g'"
```

</CodeGroupItem>

<CodeGroupItem title="Mac">

```sh
$ checkersd query checkers show-stored-game 1 --output json | jq ".storedGame.board" | sed 's/"//g' | sed 's/|/\'$'\n/g'
```

</CodeGroupItem>

</CodeGroup>

When you are done with this exercise you can stop Ignite's `chain serve.`

<HighlightBox type="synopsis">

To summarize, this section has explored:

* How to add stateless checks on your message.
* How to implement a Message Handler that will create a new game, save it in storage, and return its ID on receiving the appropriate prompt message.
* How to create unit tests to demonstrate the validity of your code.
* How to interact via the CLI to confirm that sending the appropriate transaction will successfully create a game.

</HighlightBox>

## Overview of upcoming content

You will learn how to modify this handling in later sections by:

* Adding [new fields](/hands-on-exercise/2-ignite-cli-adv/3-game-fifo.md) to the stored information.
* Adding [an event](./7-events.md).
* Consuming [some gas](/hands-on-exercise/2-ignite-cli-adv/8-gas-meter.md).
* Facilitating the eventual [deadline enforcement](/hands-on-exercise/2-ignite-cli-adv/4-game-forfeit.md).
* Adding [_money_](/hands-on-exercise/2-ignite-cli-adv/5-game-wager.md) handling, including [foreign tokens](/hands-on-exercise/2-ignite-cli-adv/10-wager-denom.md).

<!--Now that a game is created, it is time to play it by adding moves. That is the subject of the [next section](./6-play-game.md).-->
