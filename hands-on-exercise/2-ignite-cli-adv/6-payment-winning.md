---
title: "Handle Wager Payments"
order: 7
description: Token - Wagers go around
tags:
  - guided-coding
  - cosmos-sdk
---

# Handle Wager Payments

<HighlightBox type="prerequisite">

Make sure you have everything you need before proceeding:

* You understand the concepts of [modules](/academy/2-cosmos-concepts/5-modules.md) and [keepers](/academy/2-cosmos-concepts/7-multistore-keepers.md).
* Go is installed.
* You have the checkers blockchain codebase up to the game wager. If not, follow the [previous steps](./5-game-wager.md) or check out [the relevant version](https://github.com/cosmos/b9-checkers-academy-draft/tree/game-wager).

</HighlightBox>

<HighlightBox type="learning">

In this section, you will:

* Work with the bank module.
* Handle money.
* Use mocks.

</HighlightBox>

In the [previous section](./5-game-wager.md), you introduced a wager. On its own, having a `Wager` field is just a piece of information, it does not transfer tokens just by existing.

Transferring tokens is what this section is about.

## Some initial thoughts

When thinking about implementing a wager on games, ask:

* Is there any desirable atomicity of actions?
* At what junctures do you need to handle payments, refunds, and wins?
* Are there errors to report back?
* What event should you emit?

In the case of this example, you can consider that:

* Although a game creator can decide on a wager, it should only be the holder of the tokens that can decide when they are being taken from their balance.
* You might think of adding a new message type, one that indicates that a player puts its wager in escrow. On the other hand, you can leverage the existing messages and consider that when a player makes their first move, this expresses a willingness to participate, and therefore the tokens can be transferred at this juncture.
* For wins and losses, it is easy to imagine that the code handles the payout at the time a game is resolved.

## Code needs

When it comes to your code:

* What Ignite CLI commands, if any, will assist you?
* How do you adjust what Ignite CLI created for you?
* Where do you make your changes?
* How would you unit-test these new elements?
* How would you use Ignite CLI to locally run a one-node blockchain and interact with it via the CLI to see what you get?

Here are some elements of response:

* Your module needs to call the bank to tell it to move tokens.
* Your module needs to be allowed by the bank to keep tokens in escrow.
* How would you test your module when it has such dependencies on the bank?

## What is to be done

A lot is to be done. In order you will:

* Make it possible for your checkers module to call certain functions of the bank to transfer tokens.
* Tell the bank to allow your checkers module to hold tokens in escrow.
* Create helper functions that encapsulate some knowledge about when and how to transfer tokens.
* Use these helper functions at the right places in your code.
* Update your unit tests and make use of mocks for that. You will create the mocks, create helper functions and use all that.

## Declaring expectations

On its own the `Wager` field does not make players pay the wager or receive rewards. You need to add handling actions that ask the `bank` module to perform the required token transfers. For that, your keeper needs to ask for a `bank` instance during setup.

<HighlightBox type="info">

The only way to have access to a capability with the object-capability model of the Cosmos SDK is to be given the reference to an instance which already has this capability.

</HighlightBox>

Payment handling is implemented by having your keeper hold wagers **in escrow** while the game is being played. The `bank` module has functions to transfer tokens from any account to your module and vice-versa.

Alternatively, your keeper could burn tokens instead of keeping them in escrow and mint them again when paying out. However, this makes your blockchain's total supply _falsely_ fluctuate. Additionally, this burning and minting may prove questionable when you later introduce IBC tokens.

<HighlightBox type="best-practice">

Declare an interface that narrowly declares the functions from other modules that you expect for your module. The conventional file for these declarations is `x/checkers/types/expected_keepers.go`.

</HighlightBox>

The `bank` module has many capabilities, but all you need here are two functions. Your module already expects one function of the bank keeper: [`SpendableCoins`](https://github.com/cosmos/b9-checkers-academy-draft/blob/payment-winning/x/checkers/types/expected_keepers.go#L15-L18). Instead of expanding this interface, you add a new one and _redeclare_ the extra functions you need like so:

```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/payment-winning/x/checkers/types/expected_keepers.go#L20-L23]
type BankEscrowKeeper interface {
    SendCoinsFromModuleToAccount(ctx sdk.Context, senderModule string, recipientAddr sdk.AccAddress, amt sdk.Coins) error
    SendCoinsFromAccountToModule(ctx sdk.Context, senderAddr sdk.AccAddress, recipientModule string, amt sdk.Coins) error
}
```

These two functions must exactly match the functions declared in the [`bank`'s keeper.go file](https://github.com/cosmos/cosmos-sdk/blob/v0.45.4/x/bank/keeper/keeper.go#L35-L37). Copy the declarations directly from the `bank`'s file. In Go, any object with these two functions is a `BankEscrowKeeper`.

## Obtaining the capability

With your requirements declared, it is time to make sure your keeper receives a reference to a bank keeper. First add a `BankEscrowKeeper` to your keeper in `x/checkers/keeper/keeper.go`:

```diff-go [https://github.com/cosmos/b9-checkers-academy-draft/blob/payment-winning/x/checkers/keeper/keeper.go#L16]
    type (
        Keeper struct {
+          bank types.BankEscrowKeeper
            ...
        }
    )
```

This `BankEscrowKeeper` is your newly declared narrow interface. Do not forget to adjust the constructor accordingly:

```diff-go [https://github.com/cosmos/b9-checkers-academy-draft/blob/payment-winning/x/checkers/keeper/keeper.go#L25-L38]
    func NewKeeper(
+      bank types.BankEscrowKeeper,
        ...
    ) *Keeper {
        return &Keeper{
+          bank: bank,
            ...
        }
    }
```

Next, update where the constructor is called and pass a proper instance of `BankKeeper`. This happens in `app/app.go`:

```diff-go [https://github.com/cosmos/b9-checkers-academy-draft/blob/payment-winning/app/app.go#L394]
    app.CheckersKeeper = *checkersmodulekeeper.NewKeeper(
+      app.BankKeeper,
        ...
    )
```

This `app.BankKeeper` is a full `bank` keeper that also conforms to your `BankEscrowKeeper` interface.

Finally, inform the app that your checkers module is going to hold balances in escrow by adding it to the **whitelist** of permitted modules:

```diff-go [https://github.com/cosmos/b9-checkers-academy-draft/blob/payment-winning/app/app.go#L173]
    maccPerms = map[string][]string{
        ...
+      checkersmoduletypes.ModuleName: nil,
    }
```

If you compare it to the other `maccperms` lines, the new line does not mention any `authtypes.Minter` or `authtypes.Burner`. Indeed `nil` is what you need to keep in escrow. For your information, the bank creates an _address_ for your module's escrow account. When you have the full `app`, you can access it with:

```go
import(
    "github.com/alice/checkers/x/checkers/types"
)
checkersModuleAddress := app.AccountKeeper.GetModuleAddress(types.ModuleName)
```

## Preparing expected errors

There are several new error situations that you can enumerate with new variables:

```diff-go [https://github.com/cosmos/b9-checkers-academy-draft/blob/payment-winning/x/checkers/types/errors.go#L23-L28]
    var (
        ...
+      ErrBlackCannotPay    = sdkerrors.Register(ModuleName, 1110, "black cannot pay the wager")
+      ErrRedCannotPay      = sdkerrors.Register(ModuleName, 1111, "red cannot pay the wager")
+      ErrNothingToPay      = sdkerrors.Register(ModuleName, 1112, "there is nothing to pay, should not have been called")
+      ErrCannotRefundWager = sdkerrors.Register(ModuleName, 1113, "cannot refund wager to: %s")
+      ErrCannotPayWinnings = sdkerrors.Register(ModuleName, 1114, "cannot pay winnings to winner: %s")
+      ErrNotInRefundState  = sdkerrors.Register(ModuleName, 1115, "game is not in a state to refund, move count: %d")
    )
```

## Money handling steps

With the `bank` now in your keeper, it is time to have your keeper handle the money. Keep this concern in its own file, as the functions are reused on play and forfeit.

Create the new file `x/checkers/keeper/wager_handler.go` and add three functions to collect a wager, refund a wager, and pay winnings:

```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/payment-winning/x/checkers/keeper/wager_handler.go]
func (k *Keeper) CollectWager(ctx sdk.Context, storedGame *types.StoredGame) error
func (k *Keeper) MustPayWinnings(ctx sdk.Context, storedGame *types.StoredGame)
func (k *Keeper) MustRefundWager(ctx sdk.Context, storedGame *types.StoredGame)
```

The `Must` prefix in the function expresses the fact that the transaction either takes place or a `panic` is issued. Indeed, if a player cannot pay the wager, it is a user-side error and the user must be informed of a failed transaction. If the module cannot pay, it means the escrow account has failed. This latter error is much more serious: an invariant may have been violated and the whole application must be terminated.

Now set up collecting a wager, paying winnings, and refunding a wager:

1. **Collecting wagers** happens on a player's first move. Therefore, differentiate between players:

    ```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/payment-winning/x/checkers/keeper/wager_handler.go#L12-L33]
    if storedGame.MoveCount == 0 {
        // Black plays first
    } else if storedGame.MoveCount == 1 {
        // Red plays second
    }
    return nil
    ```

    When there are no moves, get the address for the black player:

    ```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/payment-winning/x/checkers/keeper/wager_handler.go#L14-L17]
    black, err := storedGame.GetBlackAddress()
    if err != nil {
        panic(err.Error())
    }
    ```

    Try to transfer into the escrow:

    ```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/payment-winning/x/checkers/keeper/wager_handler.go#L18-L21]
    err = k.bank.SendCoinsFromAccountToModule(ctx, black, types.ModuleName, sdk.NewCoins(storedGame.GetWagerCoin()))
    if err != nil {
        return sdkerrors.Wrapf(err, types.ErrBlackCannotPay.Error())
    }
    ```

    Then do the same for the [red player](https://github.com/cosmos/b9-checkers-academy-draft/blob/payment-winning/x/checkers/keeper/wager_handler.go#L24-L31) when there is a single move.

2. **Paying winnings** takes place when the game has a declared winner. First get the winner. "No winner" is **not** an acceptable situation in this `MustPayWinnings`. The caller of the function must ensure there is a winner:

    ```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/payment-winning/x/checkers/keeper/wager_handler.go#L37-L43]
    winnerAddress, found, err := storedGame.GetWinnerAddress()
    if err != nil {
        panic(err.Error())
    }
    if !found {
        panic(fmt.Sprintf(types.ErrCannotFindWinnerByColor.Error(), storedGame.Winner))
    }
    ```

    Then calculate the winnings to pay:

    ```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/payment-winning/x/checkers/keeper/wager_handler.go#L44-L49]
    winnings := storedGame.GetWagerCoin()
    if storedGame.MoveCount == 0 {
        panic(types.ErrNothingToPay.Error())
    } else if 1 < storedGame.MoveCount {
        winnings = winnings.Add(winnings)
    }
    ```

    You double the wager only if the red player has also played and therefore both players have paid their wagers.
    
    If you did this wrongly, you could end up in a situation where a game with a single move pays out as if both players had played. This would be a serious bug that an attacker could exploit to drain your module's escrow fund.
    
    Then pay the winner:

    ```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/payment-winning/x/checkers/keeper/wager_handler.go#L50-L53]
    err = k.bank.SendCoinsFromModuleToAccount(ctx, types.ModuleName, winnerAddress, sdk.NewCoins(winnings))
    if err != nil {
        panic(fmt.Sprintf(types.ErrCannotPayWinnings.Error(), err.Error()))
    }
    ```

3. Finally, **refunding wagers** takes place when the game has partially started, i.e. only one party has paid, or when the game ends in a draw. In this narrow case of `MustRefundWager`:

    ```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/payment-winning/x/checkers/keeper/wager_handler.go#L57-L72]
    if storedGame.MoveCount == 1 {
        // Refund
    } else if storedGame.MoveCount == 0 {
        // Do nothing
    } else {
        // TODO Implement a draw mechanism.
        panic(fmt.Sprintf(types.ErrNotInRefundState.Error(), storedGame.MoveCount))
    }
    ```

    Refund the black player when there has been a single move:

    ```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/payment-winning/x/checkers/keeper/wager_handler.go#L59-L66]
    black, err := storedGame.GetBlackAddress()
    if err != nil {
        panic(err.Error())
    }
    err = k.bank.SendCoinsFromModuleToAccount(ctx, types.ModuleName, black, sdk.NewCoins(storedGame.GetWagerCoin()))
    if err != nil {
        panic(fmt.Sprintf(types.ErrCannotRefundWager.Error(), err.Error()))
    }
    ```

    If the module cannot pay, then there is a panic as the escrow has failed.

You will notice that no special case is made when the wager is zero. This is a design choice here, and which way you choose to go is up to you. Not contacting the bank unnecessarily is cheaper in gas. On the other hand, why not outsource the zero check to the bank?

## Insert wager handling

With the desired steps defined in the wager handling functions, it is time to invoke them at the right places in the message handlers.

1. When a player plays for the first time:

    ```diff-go [https://github.com/cosmos/b9-checkers-academy-draft/blob/payment-winning/x/checkers/keeper/msg_server_play_move.go#L47-L50]
        ...
        if !game.TurnIs(player) {
            return nil, sdkerrors.Wrapf(types.ErrNotPlayerTurn, "%s", player)
        }

    +  err = k.Keeper.CollectWager(ctx, &storedGame)
    +  if err != nil {
    +      return nil, err
    +  }
        ...
    ```

2. When a player wins as a result of a move:

    ```diff-go [https://github.com/cosmos/b9-checkers-academy-draft/blob/payment-winning/x/checkers/keeper/msg_server_play_move.go#L79]
        if storedGame.Winner == rules.PieceStrings[rules.NO_PLAYER] {
            ...
        } else {
            ...
            k.Keeper.RemoveFromFifo(ctx, &storedGame, &systemInfo)
    +      k.Keeper.MustPayWinnings(ctx, &storedGame)
        }
    ```

3. When a game expires and there is a forfeit, make sure to only refund or pay full winnings when applicable. The logic needs to be adjusted:

    ```diff-go [https://github.com/cosmos/b9-checkers-academy-draft/blob/payment-winning/x/checkers/keeper/end_block_server_game.go#L45-L56]
        if deadline.Before(ctx.BlockTime()) {
            ...
            if storedGame.MoveCount <= 1 {
                k.RemoveStoredGame(ctx, gameIndex)
    +          if storedGame.MoveCount == 1 {
    +              k.MustRefundWager(ctx, &storedGame)
    +          }
            } else {
                ...
    +          k.MustPayWinnings(ctx, &storedGame)
                storedGame.Board = ""
                ...
            }
        }
    ```

## Unit tests

If you try running your existing tests you get a compilation error on the [test keeper builder](https://github.com/cosmos/b9-checkers-academy-draft/blob/payment-winning/testutil/keeper/checkers.go#L44-L49). Passing `nil` would not get you far with the tests and creating a full-fledged bank keeper would be a lot of work and not a unit test. See the next section on [integration tests](./5-integration-tests.md) for that.

Instead, you create mocks and use them in unit tests, not only to get the existing tests to pass but also to verify that the bank is called as expected.

### Prepare mocks

It is better to create some [mocks](https://en.wikipedia.org/wiki/Mock_object). The Cosmos SDK does not offer mocks of its objects so you have to create your own. For that, the [`gomock`](https://github.com/golang/mock) library is a good resource. Install it:

<CodeGroup>

<CodeGroupItem title="Local" active>

```sh
$ go install github.com/golang/mock/mockgen@v1.6.0
```

</CodeGroupItem>

<CodeGroupItem title="Docker">

```Dockerfile [https://github.com/cosmos/b9-checkers-academy-draft/blob/payment-winning/Dockerfile-ubuntu#L8-L33]
...
ENV NODE_VERSION=18.x
ENV MOCKGEN_VERSION=1.6.0
...
RUN apt-get install -y nodejs

# Install Mockgen
RUN go install github.com/golang/mock/mockgen@v${MOCKGEN_VERSION}
...
```

Rebuild your Docker image.

</CodeGroupItem>

</CodeGroup>

With the library installed, you still need to do a one time creation of the mocks. Run:

<CodeGroup>

<CodeGroupItem title="Local" active>

```sh
$ mockgen -source=x/checkers/types/expected_keepers.go \
    -package testutil \
    -destination=x/checkers/testutil/expected_keepers_mocks.go 
```

</CodeGroupItem>

<CodeGroupItem title="Docker">

```sh
$ docker run --rm -it \
    -v $(pwd):/checkers \
    -w /checkers \
    checkers_i \
    mockgen \
    -source=x/checkers/types/expected_keepers.go \
    -package testutil \
    -destination=x/checkers/testutil/expected_keepers_mocks.go 
```

</CodeGroupItem>

</CodeGroup>

If your expected keepers change, you will have to run this command again. It can be a good idea to save the command for future reference. You may use a `Makefile` for that. Ensure you install the `make` tool for your computer. If you use Docker, add it to the packages and rebuild the image:

```Dockerfile [https://github.com/cosmos/b9-checkers-academy-draft/blob/payment-winning/Dockerfile-ubuntu#L18]
ENV PACKAGES curl gcc jq make
```

Create the `Makefile`:

```lang-makefile [https://github.com/cosmos/b9-checkers-academy-draft/blob/payment-winning/Makefile#L1-L4]
mock-expected-keepers:
    mockgen -source=x/checkers/types/expected_keepers.go \
        -package testutil \
        -destination=x/checkers/testutil/expected_keepers_mocks.go 
```

At any time, you can rebuild the mocks with:

<CodeGroup>

<CodeGroupItem title="Local" active>

```sh
$ make mock-expected-keepers
```

</CodeGroupItem>

<CodeGroupItem title="Docker">

```sh
$ docker run --rm -it \
    -v $(pwd):/checkers \
    -w /checkers \
    checkers_i \
    make mock-expected-keepers
```

</CodeGroupItem>

</CodeGroup>

You are going to set the expectations on this `BankEscrowKeeper` mock many times, including when you do not care about the result. So instead of setting the verbose expectations in every test, it is in your interest to create helper functions that will make setting up the expectations more succinct and therefore readable, which is always a benefit when unit testing. Create a new `bank_escrow_helpers.go` file with:

* A function to _unset_ expectations (ie where anything can go). This is useful when your test does not check things around the escrow:

    ```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/payment-winning/x/checkers/testutil/bank_escrow_helpers.go#L11-L14]
    func (escrow *MockBankEscrowKeeper) ExpectAny(context context.Context) {
        escrow.EXPECT().SendCoinsFromAccountToModule(sdk.UnwrapSDKContext(context), gomock.Any(), gomock.Any(), gomock.Any()).AnyTimes()
        escrow.EXPECT().SendCoinsFromModuleToAccount(sdk.UnwrapSDKContext(context), gomock.Any(), gomock.Any(), gomock.Any()).AnyTimes()
    }
    ```

* A function to quickly create the right type and value of coins to use for expectations:

    ```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/payment-winning/x/checkers/testutil/bank_escrow_helpers.go#L16-L23]
    func coinsOf(amount uint64) sdk.Coins {
        return sdk.Coins{
            sdk.Coin{
                Denom:  sdk.DefaultBondDenom,
                Amount: sdk.NewInt(int64(amount)),
            },
        }
    }
    ```

* A function to define an expectation of payment from a player:

    ```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/payment-winning/x/checkers/testutil/bank_escrow_helpers.go#L25-L31]
    func (escrow *MockBankEscrowKeeper) ExpectPay(context context.Context, who string, amount uint64) *gomock.Call {
        whoAddr, err := sdk.AccAddressFromBech32(who)
        if err != nil {
            panic(err)
        }
        return escrow.EXPECT().SendCoinsFromAccountToModule(sdk.UnwrapSDKContext(context), whoAddr, types.ModuleName, coinsOf(amount))
    }
    ```

* A function to define an expectation of payment from the module:

    ```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/payment-winning/x/checkers/testutil/bank_escrow_helpers.go#L33-L39]
    func (escrow *MockBankEscrowKeeper) ExpectRefund(context context.Context, who string, amount uint64) *gomock.Call {
        whoAddr, err := sdk.AccAddressFromBech32(who)
        if err != nil {
            panic(err)
        }
        return escrow.EXPECT().SendCoinsFromModuleToAccount(sdk.UnwrapSDKContext(context), types.ModuleName, whoAddr, coinsOf(amount))
    }
    ```

Note that the two main expectation functions return `*gomock.Call`. This is so that you can continue adding expectations such as:

* Instructing the number of times the call is expected to be made: for instance `.Times(1)`.
* Instructing the order of the expected calls: for instance `.After(<the previous call>)`.

### Make use of mocks

With the helpers in place, you can add a new function similar to `CheckersKeeper(t testing.TB)` but which uses mocks. Keep the original function, which passes a `nil` for bank:

```diff-go [https://github.com/cosmos/b9-checkers-academy-draft/blob/payment-winning/testutil/keeper/checkers.go#L21-L58]
    func CheckersKeeper(t testing.TB) (*keeper.Keeper, sdk.Context) {
+      return CheckersKeeperWithMocks(t, nil)
+  }

+  func CheckersKeeperWithMocks(t testing.TB, bank *testutil.MockBankEscrowKeeper) (*keeper.Keeper, sdk.Context) {
        ...
        k := keeper.NewKeeper(
+          bank,
            ...
        )
        ...
    }
```

The `CheckersKeeperWithMocks` function takes the mock in its arguments for more versatility. The `CheckersKeeper` remains for the tests that never call the escrow.

Now adjust the small functions that set up the keeper before each test. You do not need to change them for the _create_ tests, because they never call the bank. You have to do it for _play_ and _forfeit_.

For _play_:

```diff-go [https://github.com/cosmos/b9-checkers-academy-draft/blob/payment-winning/x/checkers/keeper/msg_server_play_move_test.go#L17-L32]
-  func setupMsgServerWithOneGameForPlayMove(t testing.TB) (types.MsgServer, keeper.Keeper, context.Context) {
-      k, ctx := keepertest.CheckersKeeper(t)
+  func setupMsgServerWithOneGameForPlayMove(t testing.TB) (types.MsgServer, keeper.Keeper, context.Context,
+      *gomock.Controller, *testutil.MockBankEscrowKeeper) {
+      ctrl := gomock.NewController(t)
+      bankMock := testutil.NewMockBankEscrowKeeper(ctrl)
+      k, ctx := keepertest.CheckersKeeperWithMocks(t, bankMock)
        checkers.InitGenesis(ctx, *k, *types.DefaultGenesis())
        server := keeper.NewMsgServerImpl(*k)
        context := sdk.WrapSDKContext(ctx)
        server.CreateGame(context, &types.MsgCreateGame{
            Creator: alice,
            Black:   bob,
            Red:     carol,
            Wager:   45,
        })
-      return server, *k, context
+      return server, *k, context, ctrl, bankMock
    }
```

This function creates the mock and returns two new objects:

* The mock controller, so that the `.Finish()` method can be called within the test itself. This is the function that will verify the call expectations placed on the mocks.
* The mocked bank escrow. This is the instance on which you place the call expectations.

Both objects will be used from the tests proper.

Do the same for _forfeit_ if their unit tests do not use the above `setupMsgServerWithOneGameForPlayMove`.

### Adjust the unit tests

With these changes, you need to adjust many unit tests for _play_ and _forfeit_. Often you may only want to make the tests pass again without checking any meaningful bank call expectations. There are different situations:

1. The mocked bank is not called. Therefore, you do not add any expectation and still call the controller:

    ```diff-go [https://github.com/cosmos/b9-checkers-academy-draft/blob/payment-winning/x/checkers/keeper/end_block_server_game_test.go#L13-L15]
        func TestForfeitUnplayed(t *testing.T) {
    -      _, keeper, context := setupMsgServerWithOneGameForPlayMove(t)
    +      _, keeper, context, ctrl, _ := setupMsgServerWithOneGameForPlayMove(t)
            ctx := sdk.UnwrapSDKContext(context)
    +      defer ctrl.Finish()
            ...
        }
    ```

    When you expect the mocked bank **not** to be called, it is important that you do not put any expectations on it. This means the test will fail if it is called by mistake.

2. The mocked bank is called, but you do not care about how it is called:

    ```diff-go
    -  msgServer, _, context := setupMsgServerWithOneGameForRejectGame(t)
    +  msgServer, _, context, ctrl, escrow := setupMsgServerWithOneGameForRejectGame(t)
    +  defer ctrl.Finish()
    +  escrow.ExpectAny(context)
    ```

    Here you do not want to be distracted by escrow concerns.

3. The mocked bank is called, and you want to add call expectations:

    ```diff-go [https://github.com/cosmos/b9-checkers-academy-draft/blob/payment-winning/x/checkers/keeper/end_block_server_game_test.go#L139-L143]
        func TestForfeitPlayedOnce(t *testing.T) {
    -      msgServer, keeper, context := setupMsgServerWithOneGameForPlayMove(t)
    +      msgServer, keeper, context, ctrl, escrow := setupMsgServerWithOneGameForPlayMove(t)
            ctx := sdk.UnwrapSDKContext(context)
    +      defer ctrl.Finish()
    +      pay := escrow.ExpectPay(context, bob, 45).Times(1)
    +      escrow.ExpectRefund(context, bob, 45).Times(1).After(pay)
            ...
        }
    ```

Go ahead and make the many necessary changes as you see fit.

### Wager handler unit tests

After these adjustments, it is a good idea to add unit tests directly on the wager handling functions of the keeper. Create a new `wager_handler_test.go` file. In it:

1. Add a setup helper function that does not create any message server:

    ```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/payment-winning/x/checkers/keeper/wager_handler_test.go#L18-L26]
    func setupKeeperForWagerHandler(t testing.TB) (keeper.Keeper, context.Context,
        *gomock.Controller, *testutil.MockBankEscrowKeeper) {
        ctrl := gomock.NewController(t)
        bankMock := testutil.NewMockBankEscrowKeeper(ctrl)
        k, ctx := keepertest.CheckersKeeperWithMocks(t, bankMock)
        checkers.InitGenesis(ctx, *k, *types.DefaultGenesis())
        context := sdk.WrapSDKContext(ctx)
        return *k, context, ctrl, bankMock
    }
    ```

2. Add tests on the `CollectWager` function. For instance, when the game is malformed:

    ```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/payment-winning/x/checkers/keeper/wager_handler_test.go#L28-L40]
    func TestWagerHandlerCollectWrongNoBlack(t *testing.T) {
        keeper, context, ctrl, _ := setupKeeperForWagerHandler(t)
        ctx := sdk.UnwrapSDKContext(context)
        defer ctrl.Finish()
        defer func() {
            r := recover()
            require.NotNil(t, r, "The code did not panic")
            require.Equal(t, "black address is invalid: : empty address string is not allowed", r)
        }()
        keeper.CollectWager(ctx, &types.StoredGame{
            MoveCount: 0,
        })
    }
    ```

    Or when the black player failed to escrow the wager:

    ```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/payment-winning/x/checkers/keeper/wager_handler_test.go#L42-L57]
    func TestWagerHandlerCollectFailedNoMove(t *testing.T) {
        keeper, context, ctrl, escrow := setupKeeperForWagerHandler(t)
        ctx := sdk.UnwrapSDKContext(context)
        defer ctrl.Finish()
        black, _ := sdk.AccAddressFromBech32(alice)
        escrow.EXPECT().
            SendCoinsFromAccountToModule(ctx, black, types.ModuleName, gomock.Any()).
            Return(errors.New("oops"))
        err := keeper.CollectWager(ctx, &types.StoredGame{
            Black:     alice,
            MoveCount: 0,
            Wager:     45,
        })
        require.NotNil(t, err)
        require.EqualError(t, err, "black cannot pay the wager: oops")
    }
    ```

    Or when the collection of a wager works:

    ```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/payment-winning/x/checkers/keeper/wager_handler_test.go#L90-L101]
    func TestWagerHandlerCollectNoMove(t *testing.T) {
        keeper, context, ctrl, escrow := setupKeeperForWagerHandler(t)
        ctx := sdk.UnwrapSDKContext(context)
        defer ctrl.Finish()
        escrow.ExpectPay(context, alice, 45)
        err := keeper.CollectWager(ctx, &types.StoredGame{
            Black:     alice,
            MoveCount: 0,
            Wager:     45,
        })
        require.Nil(t, err)
    }
    ```

3. Add similar tests to the payment of winnings from the escrow. When it fails:

    ```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/payment-winning/x/checkers/keeper/wager_handler_test.go#L163-L184]
    func TestWagerHandlerPayWrongEscrowFailed(t *testing.T) {
        keeper, context, ctrl, escrow := setupKeeperForWagerHandler(t)
        ctx := sdk.UnwrapSDKContext(context)
        defer ctrl.Finish()
        black, _ := sdk.AccAddressFromBech32(alice)
        escrow.EXPECT().
            SendCoinsFromModuleToAccount(ctx, types.ModuleName, black, gomock.Any()).
            Times(1).
            Return(errors.New("oops"))
        defer func() {
            r := recover()
            require.NotNil(t, r, "The code did not panic")
            require.Equal(t, r, "cannot pay winnings to winner: oops")
        }()
        keeper.MustPayWinnings(ctx, &types.StoredGame{
            Black:     alice,
            Red:       bob,
            Winner:    "b",
            MoveCount: 1,
            Wager:     45,
        })
    }
    ```

    Or when it works:

    ```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/payment-winning/x/checkers/keeper/wager_handler_test.go#L200-L212]
    func TestWagerHandlerPayEscrowCalledTwoMoves(t *testing.T) {
        keeper, context, ctrl, escrow := setupKeeperForWagerHandler(t)
        ctx := sdk.UnwrapSDKContext(context)
        defer ctrl.Finish()
        escrow.ExpectRefund(context, alice, 90)
        keeper.MustPayWinnings(ctx, &types.StoredGame{
            Black:     alice,
            Red:       bob,
            Winner:    "b",
            MoveCount: 2,
            Wager:     45,
        })
    }
    ```

4. You will also need a test for [refund](https://github.com/cosmos/b9-checkers-academy-draft/blob/payment-winning/x/checkers/keeper/wager_handler_test.go#L251-L270) situations.

### Add bank escrow unit tests

Now that the wager handling has been convincingly tested, you want to confirm that its functions are called at the right junctures. Add dedicated tests with message servers that confirm how the bank is called. Add them in existing files, for instance:

```go [https://github.com/cosmos/b9-checkers-academy-draft/blob/payment-winning/x/checkers/keeper/msg_server_play_move_winner_test.go#L57-L65]
func TestPlayMoveUpToWinnerCalledBank(t *testing.T) {
    msgServer, _, context, ctrl, escrow := setupMsgServerWithOneGameForPlayMove(t)
    defer ctrl.Finish()
    payBob := escrow.ExpectPay(context, bob, 45).Times(1)
    payCarol := escrow.ExpectPay(context, carol, 45).Times(1).After(payBob)
    escrow.ExpectRefund(context, bob, 90).Times(1).After(payCarol)

    playAllMoves(t, msgServer, context, "1", testutil.Game1Moves)
}
```

After doing all that, confirm that your tests run.

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

You have confirmed that your code behaves correctly given the expectations you have placed on the bank keeper.

## Interact via the CLI

With the tests done, see what happens at the command-line and see if the bank keeper behaves as you expected.

Keep the game expiry at 5 minutes to be able to test a forfeit, as done in a [previous section](./4-game-forfeit.md). Now, you need to check balances after relevant steps to test that wagers are being withheld and paid.

How much do Alice and Bob have to start with?

<CodeGroup>

<CodeGroupItem title="Local" active>

```sh
$ checkersd query bank balances $alice
$ checkersd query bank balances $bob
```

</CodeGroupItem>

<CodeGroupItem title="Docker">

```sh
$ docker exec -it checkers \
    checkersd query bank balances $alice
$ docker exec -it checkers \
    checkersd query bank balances $bob
```

</CodeGroupItem>

</CodeGroup>

This prints:

```txt
balances:
- amount: "100000000"
  denom: stake
- amount: "20000"
  denom: token
  pagination:
  next_key: null
  total: "0"
balances:
- amount: "100000000"
  denom: stake
- amount: "10000"
  denom: token
  pagination:
  next_key: null
  total: "0"
```

### A game that expires

Create a game on which the wager will be refunded because the player playing `red` did not join:

<CodeGroup>

<CodeGroupItem title="Local" active>

```sh
$ checkersd tx checkers create-game $alice $bob 1000000 --from $alice
```

</CodeGroupItem>

<CodeGroupItem title="Docker">

```sh
$ docker exec -it checkers \
    checkersd tx checkers create-game $alice $bob 1000000 --from $alice
```

</CodeGroupItem>

</CodeGroup>

Confirm that the balances of both Alice and Bob are unchanged - as they have not played yet.

<HighlightBox type="note">

In this example, Alice paid no gas fees, other than the transaction costs, to create a game. The gas price is likely `0` here anyway. This is fixed in the [next section](./6-gas-meter.md).

</HighlightBox>

Have Alice play:

<CodeGroup>

<CodeGroupItem title="Local" active>

```sh
$ checkersd tx checkers play-move 1 1 2 2 3 --from $alice
```

</CodeGroupItem>

<CodeGroupItem title="Docker">

```sh
$ docker exec -it checkers \
    checkersd tx checkers play-move 1 1 2 2 3 --from $alice
```

</CodeGroupItem>

</CodeGroup>

Confirm that Alice has paid her wager:

<CodeGroup>

<CodeGroupItem title="Local" active>

```sh
$ checkersd query bank balances $alice
```

</CodeGroupItem>

<CodeGroupItem title="Docker">

```sh
$ docker exec -it checkers \
    checkersd query bank balances $alice
```

</CodeGroupItem>

</CodeGroup>

This prints:

```txt
balances:
- amount: "99000000" # <- 1,000,000 fewer
  denom: stake
- amount: "20000"
  denom: token
pagination:
  next_key: null
  total: "0"
```

Wait 5 minutes for the game to expire and check again:

<CodeGroup>

<CodeGroupItem title="Local" active>

```sh
$ checkersd query bank balances $alice
```

</CodeGroupItem>

<CodeGroupItem title="Docker">

```sh
$ docker exec -it checkers \
    checkersd query bank balances $alice
```

</CodeGroupItem>

</CodeGroup>

This prints:

```txt
balances:
- amount: "100000000" # <- 1,000,000 are back
  denom: stake
- amount: "20000"
  denom: token
pagination:
  next_key: null
  total: "0"
```

### A game played twice

Now create a game in which both players only play once each, i.e. where the player playing `black` forfeits:

<CodeGroup>

<CodeGroupItem title="Local" active>

```sh
$ checkersd tx checkers create-game $alice $bob 1000000 --from $alice
$ checkersd tx checkers play-move 2 1 2 2 3 --from $alice
$ checkersd tx checkers play-move 2 0 5 1 4 --from $bob
```

</CodeGroupItem>

<CodeGroupItem title="Docker">

```sh
$ docker exec -it checkers \
    checkersd tx checkers create-game $alice $bob 1000000 --from $alice
$ docker exec -it checkers \
    checkersd tx checkers play-move 2 1 2 2 3 --from $alice
$ docker exec -it checkers \
    checkersd tx checkers play-move 2 0 5 1 4 --from $bob
```

</CodeGroupItem>

</CodeGroup>

Confirm that both Alice and Bob paid their wagers. Wait 5 minutes for the game to expire and check again:

<CodeGroup>

<CodeGroupItem title="Local" active>

```sh
$ checkersd query bank balances $alice
$ checkersd query bank balances $bob
```

</CodeGroupItem>

<CodeGroupItem title="Docker">

```sh
$ docker exec -it checkers \
    checkersd query bank balances $alice
$ docker exec -it checkers \
    checkersd query bank balances $bob
```

</CodeGroupItem>

</CodeGroup>

This shows:

```txt
balances:
- amount: "99000000" # <- her 1,000,000 are gone for good
  denom: stake
...
balances:
- amount: "101000000" # <- 1,000,000 more than at the beginning
  denom: stake
...
```

This is correct: Bob was the winner by forfeit.

Similarly, you can test that Alice gets her wager back when Alice creates a game, Alice plays, and then Bob lets it expire.

It would be difficult to test by CLI when there is a winner after a full game. That would be better tested with a GUI, or by using integration tests as is done in the [next section](./7-integration-tests.md).

<HighlightBox type="synopsis">

To summarize, this section has explored:

* How to work with the Bank module and handle players making wagers on games, now that the application supports live games playing to completion (with the winner claiming both wagers) or expiring through inactivity (with the inactive player forfeiting their wager as if losing), and no possibility of withheld value being stranded in inactive games.
* How to add handling actions that ask the `bank` module to perform the token transfers required by the wager, and where to invoke them in the message handlers.
* How to create a new wager-handling file with functions to collect a wager, refund a wager, and pay winnings, in which `must` prefixes indicate either a user-side error (leading to a failed transaction) or a failure of the application's escrow account (requiring the whole application be terminated).
* How to interact with the CLI to check account balances to test that wagers are being withheld and paid.

</HighlightBox>

<!--## Next up

You can skip ahead and see how to integrate [foreign tokens](/hands-on-exercise/2-ignite-cli-adv/8-wager-denom.md) via the use of IBC, or take a look at the [next section](./6-gas-meter.md) to prevent spam and reward validators proportional to their effort in your checkers blockchain.-->
