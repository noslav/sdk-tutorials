---
title: "Incentivize Players"
order: 9
description: Gas - reward validators proportional to their effort
tags: 
  - guided-coding
  - cosmos-sdk
---

# Incentivize Players

<HighlightBox type="prerequisite">

Make sure you have everything you need before proceeding:

* You understand the concept of gas.
* Go is installed.
* You have the checkers blockchain codebase with the integration tests. If not, follow the [previous steps](./7-integration-tests.md) or check out the [relevant version](https://github.com/cosmos/b9-checkers-academy-draft/tree/integration-tests).

</HighlightBox>

<HighlightBox type="learning">

In this section, you will:

* Add transaction fees.
* Set fees and add metering.
* Do integration tests.

</HighlightBox>

Players can start playing checkers with your Cosmos blockchain. Transaction fees are paid by the players themselves, at least the fee related to transporting the serialized bytes and the other gas-metered parts like `bank`.

Your blockchain is taking shape, but you need to take care of peripheral concerns. For instance, how do you make sure that participants pay their fair share of the costs they impose on the network?

Next, you should add your own gas metering to reflect the costs that different transactions impose, or you can add costs to discourage spam.

## Some initial thoughts

To continue developing your checkers blockchain:

* At what junctures can you charge gas?
* At what junctures can you **not** charge gas, and what do you do about it?
* Are there new errors to report back?
* What event should you emit?

## Code needs

Before diving into the specifics, ask yourself:

* What Ignite CLI commands, if any, will assist you?
* How do you adjust what Ignite CLI created for you?
* Where do you make your changes?
* How would you unit-test these new elements?
* How would you use Ignite CLI to locally run a one-node blockchain and interact with it via the CLI to see what you get?

## New data

These values provide examples but you can, and should, set your own. To get a rule-of-thumb idea of how much gas is already consumed without your additions, look back at your previous transactions. Save your pick of the values as new constants:

```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/gas-meter/x/checkers/types/keys.go#L65-L68]
const (
    CreateGameGas = 15000
    PlayMoveGas   = 1000
)
```

Here are the debatable rationales for these values:

1. Creating a game imposes a large cost because it creates a brand new entry in storage, which contains many fields. This new storage entry is stored on all nodes.
2. Playing a game imposes a smaller cost because it makes changes to an existing storage entry, which was already paid for. On the other hand it costs some computation and pushes back the time by when the game expires.

When looking at all the actions a user can take, you can also consider a gas refund – in particular if the action is beneficial to the network. For instance, removing an item from storage entirely can be considered beneficial. In a situation like this, you can consider a gas refund (i.e. a negative gas cost). This gas refund will offset part or all of the gas cost already expended as part of the message.

As the checkers blockchain creator, your goal may be to have as many on-going games as possible. Adding costs sounds counter to this goal. However, here the goal is to optimize potential congestion at the margin. If there is little network activity then the gas price will go down, and these additional costs will be trivial for players anyway. Conversely, if there is a lot of network activity the gas price will go up, and whether you have put additional costs or not players will still be less likely to participate.

## Add handling

Add a line that consumes or refunds the designated amount of gas in each relevant handler:

1. When handling a game creation:

    ```diff-go [https://github.com/cosmos/b9-checkers-academy-draft/blob/gas-meter/x/checkers/keeper/msg_server_create_game.go#L46]
        ...
        k.Keeper.SetSystemInfo(ctx, systemInfo)
    +  ctx.GasMeter().ConsumeGas(types.CreateGameGas, "Create game")
        ...
    ```

2. When handling a move:

    ```diff-go [https://github.com/cosmos/b9-checkers-academy-draft/blob/gas-meter/x/checkers/keeper/msg_server_play_move.go#L90]
        ...
        k.Keeper.SetSystemInfo(ctx, systemInfo)
    +  ctx.GasMeter().ConsumeGas(types.PlayMoveGas, "Play a move")
        ...
    ```

In this example, when handling a move the gas is measured towards the end of the function's body. This is a design decision, and you may arrive at a different conclusion.

<HighlightBox type="note">

In the current arrangement, if a player hits an error (such as a failure to pay the wager, or making an illegal move) then the code will return an error _before_ the gas metering line. This means the player making the error will only pay the gas already metered by the rest of the Cosmos SDK.

If you decide to move the gas metering line closer to the beginning of the function's body, then you will charge the player extra gas for any move that contains errors _despite the fact that the requested move will not be accepted_.

This alternate approach would certainly dis-incentivize players from submitting erroneous moves, but it also risks alienating them through financial punishment for accidental mistakes. Better is to implement a feature that allows players to check a move is valid _before_ they pay the cost of handling. This is explored in the [next section](/hands-on-exercise/2-ignite-cli-adv/7-can-play.md).

</HighlightBox>

You are free to consider charging different gas costs for when a player _plays a regular legal move_ and for when a player _plays a winning move_. After a win the board is cleared, and you therefore may want to exact a smaller cost (or even a gas refund) in this situation.

<HighlightBox type="tip">

You do not meter gas in your `EndBlock` handler because it is **not** called by a player sending a transaction. Instead, this is a service rendered by the network.

</HighlightBox>

<HighlightBox type="tip">

If you want to account for the gas cost of game expiration, you have to devise a way to pre-collect it from players as part of other messages. You could even imagine a reputation system: the next time a forfeiter plays on a game, the _past_ cost of their forfeit is taken as part of the current _play move_ transaction.

</HighlightBox>

<HighlightBox type="best-practice">

As part of your code optimization, avoid calling `ConsumeGas` with a fixed gas cost (for instance `k`) from within a loop. Each pass of the loop uses computation resources (`c`) on each node. If you know the number of times your code loops (`n`), you know that running the full loop will use `n*c` computation resources.
<br/><br/>
Now consider the case of a user who sent a transaction without enough gas. The transaction will fail anyway, but at what point will it fail?

1. If you call `ConsumeGas(k)` _within_ the loop, the transaction will fail during one of the passes (the `m`th pass). This means that the node has already used `m*c` computation resources.
2. If you call `ConsumeGas(n*k)` once _before_ the loop, the transaction will fail immediately, and the node will have used `0` computation resources.

Choosing option 2 improves the effectiveness of your blockchain, and potentially protects it from spam and denial-of-service attacks.

Additionally, making only a single call to `ConsumeGas` slightly saves computation resources of the node.

</HighlightBox>

## Unit tests

Now you must add tests that confirm the gas consumption. However, it is not possible to differentiate the gas cost that BaseApp is incurring on your messages from the gas cost your module imposes on top of it. Also, you cannot distinguish via the descriptor [unless it panics](https://github.com/cosmos/cosmos-sdk/blob/v0.45.4/store/types/gas.go#L90-L101). Nevertheless, you can add a lame test like:

```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/gas-meter/x/checkers/keeper/msg_server_create_game_test.go#L119-L131]
func TestCreate1GameConsumedGas(t *testing.T) {
    msgSrvr, _, context := setupMsgServerCreateGame(t)
    ctx := sdk.UnwrapSDKContext(context)
    before := ctx.GasMeter().GasConsumed()
    msgSrvr.CreateGame(context, &types.MsgCreateGame{
        Creator: alice,
        Black:   bob,
        Red:     carol,
        Wager:   45,
    })
    after := ctx.GasMeter().GasConsumed()
    require.GreaterOrEqual(t, after, before+25_000)
}
```

Now add another test for [play](https://github.com/cosmos/b9-checkers-academy-draft/blob/gas-meter/x/checkers/keeper/msg_server_play_move_test.go#L176-L192).

These new tests are lame, because their `5_000` or `25_000` values cannot be predicted but have to be found by trial and error.

## Interact via the CLI

Here, you want to confirm that gas is consumed by different actions. The difficulty is that Alice's and Bob's balances in `stake` tokens change not only because of the gas used but also depending on the gas price. An easy measurement is to use `--dry-run`:

<CodeGroup>

<CodeGroupItem title="Local" active>

```sh
$ checkersd tx checkers create-game \
    $alice $bob 1000000 \
    --from $alice \
    --dry-run
```

</CodeGroupItem>

<CodeGroupItem title="Docker">

```sh
$ docker exec -it checkers \
    checkersd tx checkers create-game \
    $alice $bob 1000000 \
    --from $alice \
    --dry-run
```

</CodeGroupItem>

</CodeGroup>

Say this returns `69422`, which is the estimated gas used. Now comment out the `.ConsumeGas` line in `msg_server_create_game.go`, save it, wait a few minutes for Ignite CLI to rebuild, and try again:

<CodeGroup>

<CodeGroupItem title="Local" active>

```sh
$ checkersd tx checkers create-game \
    $alice $bob 1000000 \
    --from $alice \
    --dry-run
```

</CodeGroupItem>

<CodeGroupItem title="Docker">

```sh
$ docker exec -it checkers \
    checkersd tx checkers create-game \
    $alice $bob 1000000 \
    --from $alice \
    --dry-run
```

</CodeGroupItem>

</CodeGroup>

Say, this time you get `54422`. This is good: the `15000` gas is no longer part of the estimation, as expected. Uncomment the `.ConsumeGas` line. You can try `--dry-run` on play too.

Estimating with `--dry-run` is a good start. Now have Alice create a game and check the gas used in the transaction:

<CodeGroup>

<CodeGroupItem title="Local" active>

```sh
$ checkersd tx checkers create-game \
    $alice $bob 1000000 \
    --from $alice
```

</CodeGroupItem>

<CodeGroupItem title="Docker">

```sh
$ docker exec -it checkers \
    checkersd tx checkers create-game \
    $alice $bob 1000000 \
    --from $alice
```

</CodeGroupItem>

</CodeGroup>

This mentions:

```txt
...
gas_used: "69422"
...
```

You could impose a `--gas-prices` and then check balances, but this would obfuscate the gas consumption which is what you want to confirm.

As before, comment the `.ConsumeGas` line `msg_server_create_game.go` and wait for Ignite CLI to rebuild. Then try again:

<CodeGroup>

<CodeGroupItem title="Local" active>

```sh
$ checkersd tx checkers create-game \
    $alice $bob 1000000 \
    --from $alice
```

</CodeGroupItem>

<CodeGroupItem title="Docker">

```sh
$ docker exec -it checkers \
    checkersd tx checkers create-game \
    $alice $bob 1000000 \
    --from $alice
```

</CodeGroupItem>

</CodeGroup>

This mentions:

```txt
...
gas_used: "65540"
...
```

There is only a difference of `4000`. The rest of the system likely had some under-the-hood initializations, such as Merkle tree creations, which may _falsify_ the early results. Create 10 more games without `.Consume`ing gas and only look at the `gas_used`. It should stabilize at a certain value:

<CodeGroup>

<CodeGroupItem title="Local" active>

```sh
$ checkersd tx checkers create-game \
    $alice $bob 1000000 \
    --from $alice -y | 
        grep gas_used
```

</CodeGroupItem>

<CodeGroupItem title="Docker">

```sh
$ docker exec -it checkers \
    bash -c "checkersd tx checkers create-game $alice $bob 1000000 --from $alice -y | grep gas_used"
```

</CodeGroupItem>

</CodeGroup>

This mentions:

```txt
gas_used: "65507"
```

Put back the `.ConsumeGas` line and rebuild. Then try again:

<CodeGroup>

<CodeGroupItem title="Local" active>

```sh
$ checkersd tx checkers create-game \
    $alice $bob 1000000 \
    --from $alice -y | \
        grep gas_used
```

</CodeGroupItem>

<CodeGroupItem title="Docker">

```sh
$ docker exec -it checkers \
    bash -c "checkersd tx checkers create-game $alice $bob 1000000 --from $alice -y | grep gas_used"
```

</CodeGroupItem>

</CodeGroup>

It now consistently mentions a difference of `15000`:

```txt
gas_used: "80507"
```

That is sufficient confirmation.

Do not worry if you do not get the same values. At least try multiple times to see if the values look like each other on your system.

<HighlightBox type="synopsis">

To summarize, this section has explored:

* How to add gas metering to your application so participants contribute toward the cost of the work being demanded of the network by gameplay, and add costs to discourage spam.
* What new data constants need to be added, such as fees for creating games or playing moves, and gas consumption lines for handlers relating to these gameplay aspects.
* Best practices for gas metering, including where not to call fixed gas costs and the implications of a user sending transactions without enough gas to process them.
* What texts to add that confirm gas consumption, acknowledging the limitations on precision that the use of BaseApp and your module also imposes on understanding how much gas is used by various transactions.
* How to interact via the CLI to confirm that gas is being consumed by different actions, acknowledging the additional complications arising from variable account balances and gas price.

</HighlightBox>

<!--## Next up

Make your checkers blockchain more user-friendly by helping players avoid bad transactions via a query that tests a move. Just follow the exercise in the [next section](./7-can-play.md).-->
