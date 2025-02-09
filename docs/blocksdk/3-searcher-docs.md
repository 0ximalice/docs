---
description: Searcher Documentation
title: 🥷 For Searchers
sidebar_position: 3
---

:::caution Searcher Simluation

Bundles must only pass basic `CheckTx` validation (e.g. nonce, account balance, etc.) in order to be accepted by the auction. This means that bundles that are submitted to the auction may not be entirely valid on-chain since `runMsgs` is not executed. Searchers are encouraged to simulate their bundles before submitting them to the auction.
:::

### ➡️ How do searchers submit bundles to chains that use the Block SDK?

:::info Definitions
💡 `AuctionTx` (auction bid transaction) = `sdk.Tx` that includes a single `MsgAuctionBid`

:::

Searchers submit bundles by broadcasting a `AuctionTx` in the same way they broadcast any other transaction. A few important things to note:

- When a `MsgAuctionBid` message is included in a transaction, no other `sdk.Msg` can be present.
- Interfacing with the auction _may be different across_ `Block SDK` chains. Bidding may involve interacting with a dedicated `AuctionHouse` smart contract instead of including this special message type. In the future, we will link a chain directory here to check on which interface you need, when there are different implementations.

#### Default Auction Bid Message

```go
// MsgAuctionBid defines a request type for sending bids to the x/auction
// module.
type MsgAuctionBid struct {
    // bidder is the address of the account that is submitting a bid to the
    // auction.
    Bidder string `protobuf:"bytes,1,opt,name=bidder,proto3" json:"bidder,omitempty"`
    // bid is the amount of coins that the bidder is bidding to participate in the
    // auction.
    Bid types.Coin `protobuf:"bytes,3,opt,name=bid,proto3" json:"bid"`
    // transactions are the bytes of the transactions that the bidder wants to
    // bundle together.
    Transactions [][]byte `protobuf:"bytes,4,rep,name=transactions,proto3" json:"transactions,omitempty"`
}
```

There are three things searchers must specify to participate in an auction:

1. The **`Transactions`** they want executed at the top of block.
   - Transactions will be executed in the order they are specified in the message.
   - Each transactions included must be the raw bytes of the transaction.
2. The **`Bidder`** who is bidding for top of block execution.
   - This must be the same account that signs the transaction.
3. The **`Bid`** they want to send alongside the bundle.
   - This should be in the denom configured by the auction parameters (see below)
4. The **`Timeout`** height i.e. until what height the bid is valid for.
   - This must be added to the transaction when it is being constructed i.e. `txBuilder.SetTimeoutHeight(timeoutHeight)`

#### Nonce Checking

In general, all bundles must respect nonce ordering of accounts. If a bundle is submitted with an invalid nonce, it will be rejected.
The execution of the bundle will always look like the following:

- Auction Transaction (which extracts the bid)
- All transactions in the bundle (in order)

For example, assume the following:

1. Searcher has account `A` with nonce `n`
2. Searcher wants to submit a bundle with 3 transactions from account `A`

The searcher must first sign the `AuctionTx` with nonce `n + 1`. Then, the searcher must sign the first transaction in the bundle with nonce `n + 2`, the second transaction with nonce `n + 3`, and the third transaction with nonce `n + 4`.

#### Skipper Bot

User’s can bootstrap their searching bots by utilizing Skip’s own open source [Skipper Bot](https://github.com/skip-mev/skipper).

#### Creating an `AuctionTx`

```go
// createBidTx creates an auction bid tx with the given parameters
func createBidTx(
    privateKey *secp256k1.PrivKey,
    bidder string,
    bid sdk.Coin,
    bundle [][]byte,
    height uint64,
    sequenceOffset uint64,
) (sdk.Tx, error) {
    // bid transaction can only include a single MsgAuctionBid
    msgs := []sdk.Msg{
        &buildertypes.MsgAuctionBid{
            Bidder:       bidder,
            Bid:          bid,
            Transactions: bundle,
        },
    }

    // Retrieve the expected account and sequence number
    sequenceNum, accountNum := getAccountInfo(privateKey)

    txConfig := authtx.NewTxConfig(
        codec.NewProtoCodec(codectypes.NewInterfaceRegistry()),
        authTx.DefaultSignModes,
    )
    txBuilder := txConfig.NewTxBuilder()

    // Set the messages along with other fee information
    txBuilder.SetMsgs(msgs...)
    ...
    txBuilder.SetGasLimit(5000000)

    // ******************************************************
    // SET A TIMEOUT HEIGHT FOR HOW LONG THE BID IS VALID FOR
    // ******************************************************
    txBuilder.SetTimeoutHeight(height)

    // Sign the auction transaction with nonce n + 1
    // where n is the number of transactions in the bundle
    // that are signed by the searcher
    offsetNonce := sequenceNum + sequenceOffset
    signerData := auth.SignerData{
        ChainID:       CHAIN_ID,
        AccountNumber: accountNumber,
        Sequence:      offsetNonce,
    }

    sigV2, err = clientTx.SignWithPrivKey(
        txConfig.SignModeHandler().DefaultMode(),
        signerData,
        txBuilder,
        privateKey,
        txConfig,
        offsetNonce,
    )
    if err != nil {
        return nil, err
    }

    if err = txBuilder.SetSignatures(sigV2); err != nil {
        return nil, error
    }

    return txBuilder.GetTx(), nil
}
```

### ⚙️ Auction fees

:::info Auction Configuration
All auction parameters are accessible though the `/block-sdk/x/auction/v1/params` HTTP path on a standard node or gRPC service defined by `x/auction`.
:::

In order to participate in an auction, searchers must pay a fee. This fee is paid in the native token of the chain. The fee is determined by the auction parameters, which are set by the chain. The auction parameters are:

1. **`MaxBundleSize`**: specifies the maximum number of transactions that can be included in a bundle (bundle = an ordered list of transactions). Bundles must be ≤ this number.
2. **`ReserveFee`**: specifies the bid floor to participate in the auction. Bids that are lower than the reserve fee are ignored.
3. **`MinBidIncrement`**: specifies how much greater each subsequent bid must be (as seen by an individual node) in order to be considered. If the bid is lower than the `highest current bid + min bid increment`, the bid is ignored.
4. **`FrontRunningProtection`**: determines whether front-running and sandwich protection is enabled.

:::info Front-running and sandwich protection
**If this is set to true, your bundle must follow these guidelines:**

- You must put your signed transactions **after** transactions you didn’t sign
- You can only have **at most two** unique signers in the bundle

:::

Bundle Examples:

1. **Valid**: [tx1, tx2, tx3] where tx1 is signed by the signer 1 and tx2 and tx3 are signed by the bidder.
2. **Valid**: [tx1, tx2, tx3, tx4] where tx1 - tx4 are signed by the bidder.
3. **Invalid**: [tx1, tx2, tx3] where tx1 and tx3 are signed by the bidder and tx2 is signed by some other signer. (possible sandwich attack)
4. **Invalid**: [tx1, tx2, tx3] where tx1 is signed by the bidder, and tx2, tx3 are signed by some other signer. (possible front-running attack)

#### Querying auction parameters

```go
func getAuctionParams() (*auctiontypes.Params, error) {
    # Replace this URL with the gRPC url of a node
    url := "localhost:9090"

    # Establish a gRPC connection to query auction parameters
    grpcConn, err := grpc.Dial(
        url,
        grpc.WithTransportCredentials(insecure.NewCredentials()),
    )
    if err != nil {
        return nil, err
    }

    client := auctiontypes.NewQueryClient(grpcConn)

    req := &auctiontypes.QueryParamsRequest{}
    resp, err := client.Params(context.Background(), req)

    return resp.Params
}
```

### 🚨 Chains currently using the MEV-Lane

#### Mainnets

| Chain Name  | Chain-ID        | Block-SDK Version |
| ----------- | --------------- | ----------------- |
| Osmosis     | `osmosis-1`     | `v1.4.2`          |
| Neutron     | `neutron-1`     | `v1.4.0`          |
| Juno        | `juno-1`        | `v1.0.4`          |
| Persistence | `persistence-1` | `v1.0.5`          |
| Pryzm       | `NA`            | `v1.0.2`          |

#### Testnets

| Chain Name | Chain-ID       | Block-SDK Version |
| ---------- | -------------- | ----------------- |
| Initia     | `initiation-1` | `v2.1.2`          |
| Juno       | `uni-6`        | `v1.0.4`          |
