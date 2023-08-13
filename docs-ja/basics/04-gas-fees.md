---
sidebar_position: 1
---

# Gas and Fees

:::note Synopsis
This document describes the default strategies to handle gas and fees within a Cosmos SDK application.
:::

:::note Pre-requisite Readings

* [Anatomy of a Cosmos SDK Application](./00-app-anatomy.md)

:::

## Introduction to `Gas` and `Fees`

Cosmos SDKにおいて、`gas`は実行中のリソース消費を追跡するために使用される特別な単位です。通常、`gas`はストアへの読み書きが行われる際に消費されますが、高コストの計算が必要な場合にも消費されることがあります。`gas`には次の2つの主な目的があります：

* ブロックがあまりにも多くのリソースを消費せずに確定するようにすること。これはCosmos SDK内でデフォルトで実装されており、[ブロックのgasメーター](#block-gas-meter)を介して行われます。
* エンドユーザーからのスパムや乱用を防ぐこと。このために、[`message`](../building-modules/02-messages-and-queries.md#messages)の実行中に消費される`gas`は通常価格が設定され、`fee` (`fees = gas * gas-prices`) に結びつきます。一般的に`fees`は`message`の送信者が支払う必要があります。Cosmos SDKはデフォルトでは`gas`の価格設定を強制しませんが、スパムを防ぐ他の方法（例：帯域幅スキーム）があるかもしれないためです。それにもかかわらず、ほとんどのアプリケーションはスパムを防ぐために[`AnteHandler`](#antehandler)を使用して`fee`メカニズムを実装しています。

## Gas Meter

Cosmos SDKでは、`gas`は単純な`uint64`の別名であり、_gas meter_ と呼ばれるオブジェクトによって管理されます。Gasメーターは`GasMeter`インターフェースを実装しています。

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/store/types/gas.go#L40-L51
```

以下に示す通り：

* `GasConsumed()`は、ガスメーターインスタンスによって消費されたガス量を返します。
* `GasConsumedToLimit()`は、ガスメーターインスタンスによって消費されたガス量、または制限に達した場合はその制限を返します。
* `GasRemaining()`は、ガスメーターに残っているガスを返します。
* `Limit()`は、ガスメーターインスタンスの制限を返します。ガスメーターが無限の場合は`0`です。
* `ConsumeGas(amount Gas, descriptor string)`は提供された`gas`の量を消費します。`gas`がオーバーフローすると、`descriptor`メッセージと共にパニックが発生します。ガスメーターが無限でない場合、消費される`gas`が制限を超えるとパニックが発生します。
* `RefundGas()`は、指定された量を消費されたガスから差し引きます。この機能を使用すると、EVM互換のチェーンがgo-ethereumのStateDBインターフェースを完全にサポートできるよう、トランザクションやブロックのガスプールにガスを返金することができます。
* `IsPastLimit()`は、ガスメーターインスタンスによって消費されたガス量が制限を厳密に超えている場合は`true`を、そうでない場合は`false`を返します。
* `IsOutOfGas()`は、ガスメーターインスタンスによって消費されたガス量が制限を上回るか、または制限と等しい場合は`true`を、そうでない場合は`false`を返します。

ガスメーターは通常、[`ctx`](../core/02-context.md)に保持され、ガスの消費は次のパターンで行われます：

```go
ctx.GasMeter().ConsumeGas(amount, "description")
```

デフォルトでは、Cosmos SDKは2つの異なるガスメーター、[メインガスメーター](#main-gas-metter)と[ブロックガスメーター](#block-gas-meter)を使用します。

### Main Gas Meter

`ctx.GasMeter()`はアプリケーションのメインガスメーターです。メインガスメーターは`FinalizeBlock`内で`setFinalizeBlockState`を介して初期化され、その後、ステート遷移につながる実行シーケンス中のガス消費を追跡します。つまり、元々[`FinalizeBlock`](../core/00-baseapp.md#finalizeblock)によってトリガーされたものです。各トランザクションの実行開始時に、メインガスメーターは**0に設定されなければなりません**。これは[`AnteHandler`](#antehandler)内で行われ、トランザクションごとのガス消費を追跡するためです。

ガス消費は、一般的にはモジュール開発者によって[`BeginBlocker`、`EndBlocker`](../building-modules/05-beginblock-endblock.md)または[`Msg`サービス](../building-modules/03-msg-services.md)内で手動で行われることがありますが、ほとんどの場合、ストアへの読み込みまたは書き込みがあるたびに自動的に行われます。この自動的なガス消費のロジックは、[`GasKv`](../core/04-store.md#gaskv-store)と呼ばれる特別なストアに実装されています。

### Block Gas Meter

`ctx.BlockGasMeter()`は、ガス消費をブロックごとに追跡し、特定の制限を超えないようにするためのガスメーターです。`FinalizeBlock`が呼び出されるたびに、`BlockGasMeter`の新しいインスタンスが作成されます。`BlockGasMeter`は有限であり、ブロックごとのガス制限はアプリケーションのコンセンサスパラメーターで定義されます。デフォルトでは、Cosmos SDKアプリケーションはCometBFTによって提供されるデフォルトのコンセンサスパラメーターを使用します。

```go reference
https://github.com/cometbft/cometbft/blob/v0.37.0/types/params.go#L66-L105
```

新しい[トランザクション](../core/01-transactions.md)が`FinalizeBlock`を介して処理される際に、`BlockGasMeter`の現在の値が制限を超えているかどうかをチェックします。制限を超えている場合、トランザクションは失敗し、失敗したトランザクションとしてコンセンサスエンジンに返されます。これは、ブロック内の最初のトランザクションでも発生する可能性があります。なぜなら、`FinalizeBlock`自体がガスを消費するからです。制限を超えていない場合、トランザクションは通常通り処理されます。`FinalizeBlock`の最後で、トランザクションの処理に消費されたガス量を`ctx.BlockGasMeter()`が追跡します。

```go
ctx.BlockGasMeter().ConsumeGas(
	ctx.GasMeter().GasConsumedToLimit(),
	"block gas meter",
)
```

## AnteHandler

`AnteHandler`は、`CheckTx`および`FinalizeBlock`の間にある各トランザクションごとに実行され、トランザクション内の各`sdk.Msg`に対してProtobufの`Msg`サービスメソッドが実行される前に実行されます。

`anteHandler`は、Core Cosmos SDKではなくモジュールに実装されています。ただし、ほとんどのアプリケーションは、現在は[`auth`モジュール](https://github.com/cosmos/cosmos-sdk/tree/main/x/auth)で定義されているデフォルトの実装を使用しています。通常のCosmos SDKアプリケーションで`anteHandler`が行うことは次のとおりです。

* トランザクションが正しいタイプであることを確認する。トランザクションタイプは、`anteHandler`を実装するモジュールで定義され、トランザクションインターフェースに従います：

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/types/tx_msg.go#L51-L56
```

これにより、開発者はアプリケーションのトランザクションにさまざまなタイプを使用できます。デフォルトの`auth`モジュールでは、デフォルトのトランザクションタイプは`Tx`です：

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/proto/cosmos/tx/v1beta1/tx.proto#L14-L27
```

* トランザクション内の各[`message`](../building-modules/02-messages-and-queries.md#messages)に対して署名を検証します。各`message`は1人以上の送信者によって署名され、これらの署名は`anteHandler`で検証される必要があります。
* `CheckTx`中に、トランザクションとともに提供されたガス価格がローカルの`min-gas-prices`よりも大きいことを検証します（ガス価格は次の式から差し引かれることを思い出してください：`fees = gas * gas-prices`）。 `min-gas-prices`は各フルノード固有のパラメーターであり、`CheckTx`中に使用され、最小の手数料を提供しないトランザクションを破棄します。これにより、メンプールにゴミトランザクションがスパムされることはありません。
* トランザクションの送信者が`fees`をカバーするための十分な資金を持っていることを検証します。エンドユーザーがトランザクションを生成する際、次の3つのパラメーターのうち2つを指定する必要があります（3つ目は暗黙的なものです）：`fees`、`gas`、`gas-prices`。これにより、ノードにトランザクションを実行するための支払いをどれだけ行うかが示されます。提供された`gas`の値は、後で使用するために`GasWanted`という名前のパラメーターに格納されます。
* `newCtx.GasMeter`を0に設定し、制限を`GasWanted`に設定します。 **このステップは非常に重要**です。これにより、トランザクションが無限のガスを消費することはできないだけでなく、`ctx.GasMeter`が各トランザクションの間にリセットされることも確認されます（`anteHandler`が実行された後に`ctx`が`newCtx`に設定され、`anteHandler`はトランザクションが実行されるたびに実行されます）。

上記で説明したように、`anteHandler`は実行中にトランザクションが消費できる最大の`gas`制限である`GasWanted`を返します。最終的に消費される実際の量は`GasUsed`と呼ばれ、したがって`GasUsed =< GasWanted`でなければなりません。`GasWanted`と`GasUsed`の両方は、[`FinalizeBlock`](../core/00-baseapp.md#finalizeblock)が戻るときに基本となるコンセンサスエンジンに中継されます。
