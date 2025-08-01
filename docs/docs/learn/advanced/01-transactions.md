---
sidebar_position: 1
---

# Transactions

:::note Synopsis
`Transactions` are objects created by end-users to trigger state changes in the application.
:::

:::note Pre-requisite Readings

* [Anatomy of a Cosmos SDK Application](../beginner/00-app-anatomy.md)

:::

## Transactions

Transactions are comprised of metadata held in [contexts](./02-context.md) and [`sdk.Msg`s](../../build/building-modules/02-messages-and-queries.md) that trigger state changes within a module through the module's Protobuf [`Msg` service](../../build/building-modules/03-msg-services.md).

When users want to interact with an application and make state changes (e.g. sending coins), they create transactions. Each of a transaction's `sdk.Msg` must be signed using the private key associated with the appropriate account(s), before the transaction is broadcasted to the network. A transaction must then be included in a block, validated, and approved by the network through the consensus process. To read more about the lifecycle of a transaction, click [here](../beginner/01-tx-lifecycle.md).

## Type Definition

Transaction objects are Cosmos SDK types that implement the `Tx` interface

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/types/tx_msg.go#L53-L58
```

It contains the following methods:

* **GetMsgs:** unwraps the transaction and returns a list of contained `sdk.Msg`s - one transaction may have one or multiple messages, which are defined by module developers.

As a developer, you should rarely manipulate `Tx` directly, as `Tx` is an intermediate type used for transaction generation. Instead, developers should prefer the `TxBuilder` interface, which you can learn more about [below](#transaction-generation).

### Signing Transactions

Every message in a transaction must be signed by the addresses specified by its `GetSigners`. The Cosmos SDK currently allows signing transactions in two different ways.

#### `SIGN_MODE_DIRECT` (preferred)

The most used implementation of the `Tx` interface is the Protobuf `Tx` message, which is used in `SIGN_MODE_DIRECT`:

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/proto/cosmos/tx/v1beta1/tx.proto#L15-L28
```

Because Protobuf serialization is not deterministic, the Cosmos SDK uses an additional `TxRaw` type to denote the pinned bytes over which a transaction is signed. Any user can generate a valid `body` and `auth_info` for a transaction, and serialize these two messages using Protobuf. `TxRaw` then pins the user's exact binary representation of `body` and `auth_info`, called respectively `body_bytes` and `auth_info_bytes`. The document that is signed by all signers of the transaction is `SignDoc` (deterministically serialized using [ADR-027](../../build/architecture/adr-027-deterministic-protobuf-serialization.md)):

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/proto/cosmos/tx/v1beta1/tx.proto#L50-L67
```

Once signed by all signers, the `body_bytes`, `auth_info_bytes` and `signatures` are gathered into `TxRaw`, whose serialized bytes are broadcasted over the network.

#### `SIGN_MODE_LEGACY_AMINO_JSON`

The legacy implementation of the `Tx` interface is the `StdTx` struct from `x/auth`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/x/auth/migrations/legacytx/stdtx.go#L82-L89
```

The document signed by all signers is `StdSignDoc`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/x/auth/migrations/legacytx/stdsign.go#L30-L43
```

which is encoded into bytes using Amino JSON. Once all signatures are gathered into `StdTx`, `StdTx` is serialized using Amino JSON, and these bytes are broadcasted over the network.

#### Other Sign Modes

The Cosmos SDK also provides a couple of other sign modes for particular use cases.

#### `SIGN_MODE_DIRECT_AUX`

`SIGN_MODE_DIRECT_AUX` is a sign mode released in the Cosmos SDK v0.46 which targets transactions with multiple signers. Whereas `SIGN_MODE_DIRECT` expects each signer to sign over both `TxBody` and `AuthInfo` (which includes all other signers' signer infos, i.e. their account sequence, public key and mode info), `SIGN_MODE_DIRECT_AUX` allows N-1 signers to only sign over `TxBody` and _their own_ signer info. Moreover, each auxiliary signer (i.e. a signer using `SIGN_MODE_DIRECT_AUX`) doesn't
need to sign over the fees:

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/proto/cosmos/tx/v1beta1/tx.proto#L68-L93
```

The use case is a multi-signer transaction, where one of the signers is appointed to gather all signatures, broadcast the signature and pay for fees, and the others only care about the transaction body. This generally allows for a better multi-signing UX. If Alice, Bob and Charlie are part of a 3-signer transaction, then Alice and Bob can both use `SIGN_MODE_DIRECT_AUX` to sign over the `TxBody` and their own signer info (no need an additional step to gather other signers' ones, like in `SIGN_MODE_DIRECT`), without specifying a fee in their SignDoc. Charlie can then gather both signatures from Alice and Bob, and
create the final transaction by appending a fee. Note that the fee payer of the transaction (in our case Charlie) must sign over the fees, so must use `SIGN_MODE_DIRECT` or `SIGN_MODE_LEGACY_AMINO_JSON`.


#### `SIGN_MODE_TEXTUAL`

`SIGN_MODE_TEXTUAL` is a new sign mode for delivering a better signing experience on hardware wallets and it is included in the v0.50 release. In this mode, the signer signs over the human-readable string representation of the transaction (CBOR) and makes all data being displayed easier to read. The data is formatted as screens, and each screen is meant to be displayed in its entirety even on small devices like the Ledger Nano.

There are also _expert_ screens, which will only be displayed if the user has chosen that option in its hardware device. These screens contain things like account number, account sequence and the sign data hash.

Data is formatted using a set of `ValueRenderer` which the SDK provides defaults for all the known messages and value types. Chain developers can also opt to implement their own `ValueRenderer` for a type/message if they'd like to display information differently.

If you wish to learn more, please refer to [ADR-050](../../build/architecture/adr-050-sign-mode-textual.md).

#### Custom Sign modes

There is the opportunity to add your own custom sign mode to the Cosmos-SDK.  While we can not accept the implementation of the sign mode to the repository, we can accept a pull request to add the custom signmode to the SignMode enum located [here](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/proto/cosmos/tx/signing/v1beta1/signing.proto#L17)

## Transaction Process

The process of an end-user sending a transaction is:

* decide on the messages to put into the transaction,
* generate the transaction using the Cosmos SDK's `TxBuilder`,
* broadcast the transaction using one of the available interfaces.

The next paragraphs will describe each of these components, in this order.

### Messages

:::tip
Module `sdk.Msg`s are not to be confused with [ABCI Messages](https://docs.cometbft.com/v0.37/spec/abci/) which define interactions between the CometBFT and application layers.
:::

**Messages** (or `sdk.Msg`s) are module-specific objects that trigger state transitions within the scope of the module they belong to. Module developers define the messages for their module by adding methods to the Protobuf [`Msg` service](../../build/building-modules/03-msg-services.md), and also implement the corresponding `MsgServer`.

Each `sdk.Msg`s is related to exactly one Protobuf [`Msg` service](../../build/building-modules/03-msg-services.md) RPC, defined inside each module's `tx.proto` file. A SDK app router automatically maps every `sdk.Msg` to a corresponding RPC. Protobuf generates a `MsgServer` interface for each module `Msg` service, and the module developer needs to implement this interface.
This design puts more responsibility on module developers, allowing application developers to reuse common functionalities without having to implement state transition logic repetitively.

To learn more about Protobuf `Msg` services and how to implement `MsgServer`, click [here](../../build/building-modules/03-msg-services.md).

While messages contain the information for state transition logic, a transaction's other metadata and relevant information are stored in the `TxBuilder` and `Context`.

### Transaction Generation

The `TxBuilder` interface contains data closely related with the generation of transactions, which an end-user can set to generate the desired transaction:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/client/tx_config.go#L39-L57
```

* `Msg`s, the array of [messages](#messages) included in the transaction.
* `GasLimit`, option chosen by the users for how to calculate how much gas they will need to pay.
* `Memo`, a note or comment to send with the transaction.
* `FeeAmount`, the maximum amount the user is willing to pay in fees.
* `TimeoutHeight`, block height until which the transaction is valid.
* `Unordered`, an option indicating this transaction may be executed in any order (requires Sequence to be unset.)
* `TimeoutTimestamp`, the timeout timestamp (unordered nonce) of the transaction (required to be used with Unordered).
* `Signatures`, the array of signatures from all signers of the transaction.

As there are currently two sign modes for signing transactions, there are also two implementations of `TxBuilder`:

* [wrapper](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/x/auth/tx/builder.go#L27-L44) for creating transactions for `SIGN_MODE_DIRECT`,
* [StdTxBuilder](https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/x/auth/migrations/legacytx/stdtx_builder.go#L14-L17) for `SIGN_MODE_LEGACY_AMINO_JSON`.

However, the two implementations of `TxBuilder` should be hidden away from end-users, as they should prefer using the overarching `TxConfig` interface:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/client/tx_config.go#L27-L37
```

`TxConfig` is an app-wide configuration for managing transactions. Most importantly, it holds the information about whether to sign each transaction with `SIGN_MODE_DIRECT` or `SIGN_MODE_LEGACY_AMINO_JSON`. By calling `txBuilder := txConfig.NewTxBuilder()`, a new `TxBuilder` will be created with the appropriate sign mode.

Once `TxBuilder` is correctly populated with the setters exposed above, `TxConfig` will also take care of correctly encoding the bytes (again, either using `SIGN_MODE_DIRECT` or `SIGN_MODE_LEGACY_AMINO_JSON`). Here's a pseudo-code snippet of how to generate and encode a transaction, using the `TxEncoder()` method:

```go
txBuilder := txConfig.NewTxBuilder()
txBuilder.SetMsgs(...) // and other setters on txBuilder

bz, err := txConfig.TxEncoder()(txBuilder.GetTx())
// bz are bytes to be broadcasted over the network
```

### Broadcasting the Transaction

Once the transaction bytes are generated, there are currently three ways of broadcasting it.

#### CLI

Application developers create entry points to the application by creating a [command-line interface](./07-cli.md), [gRPC and/or REST interface](./06-grpc_rest.md), typically found in the application's `./cmd` folder. These interfaces allow users to interact with the application through command-line.

For the [command-line interface](../../build/building-modules/09-module-interfaces.md#cli), module developers create subcommands to add as children to the application top-level transaction command `TxCmd`. CLI commands actually bundle all the steps of transaction processing into one simple command: creating messages, generating transactions and broadcasting. For concrete examples, see the [Interacting with a Node](../../user/run-node/02-interact-node.md) section. An example transaction made using CLI looks like:

```bash
simd tx send $MY_VALIDATOR_ADDRESS $RECIPIENT 1000stake
```

#### gRPC

[gRPC](https://grpc.io) is the main component for the Cosmos SDK's RPC layer. Its principal usage is in the context of modules' [`Query` services](../../build/building-modules/04-query-services.md). However, the Cosmos SDK also exposes a few other module-agnostic gRPC services, one of them being the `Tx` service:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.53.0/proto/cosmos/tx/v1beta1/service.proto
```

The `Tx` service exposes a handful of utility functions, such as simulating a transaction or querying a transaction, and also one method to broadcast transactions.

Examples of broadcasting and simulating a transaction are shown [here](../../user/run-node/03-txs.md#programmatically-with-go).

#### REST

Each gRPC method has its corresponding REST endpoint, generated using [gRPC-gateway](https://github.com/grpc-ecosystem/grpc-gateway). Therefore, instead of using gRPC, you can also use HTTP to broadcast the same transaction, on the `POST /cosmos/tx/v1beta1/txs` endpoint.

An example can be seen [here](../../user/run-node/03-txs.md#using-rest)

#### CometBFT RPC

The three methods presented above are actually higher abstractions over the CometBFT RPC `/broadcast_tx_{async,sync,commit}` endpoints, documented [here](https://docs.cometbft.com/v0.37/core/rpc). This means that you can use the CometBFT RPC endpoints directly to broadcast the transaction, if you wish so.

### Unordered Transactions

:::tip

Looking to enable unordered transactions on your chain? 
Check out the [v0.53.0 Upgrade Guide](https://docs.cosmos.network/v0.53/build/migrations/upgrade-guide#enable-unordered-transactions-optional)

:::

:::warning

Unordered transactions MUST leave sequence values unset. When a transaction is both unordered and contains a non-zero sequence value,
the transaction will be rejected. Services that operate on prior assumptions about transaction sequence values should be updated to handle unordered transactions.
Services should be aware that when the transaction is unordered, the transaction sequence will always be zero.

:::

Beginning with Cosmos SDK v0.53.0, chains may enable unordered transaction support. 
Unordered transactions work by using a timestamp as the transaction's nonce value. The sequence value must NOT be set in the signature(s) of the transaction.
The timestamp must be greater than the current block time and not exceed the chain's configured max unordered timeout timestamp duration.
Senders must use a unique timestamp for each distinct transaction. The difference may be as small as a nanosecond, however.

These unique timestamps serve as a one-shot nonce, and their lifespan in state is short-lived.
Upon transaction inclusion, an entry consisting of timeout timestamp and account address will be recorded to state. 
Once the block time is passed the timeout timestamp value, the entry will be removed. This ensures that unordered nonces do not indefinitely fill up the chain's storage.
