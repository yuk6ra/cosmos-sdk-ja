---
sidebar_position: 1
---

# Context

:::note Synopsis
「コンテキスト」は、アプリケーションの現在の状態に関する情報を関数から関数へ渡すためのデータ構造です。これには、分岐されたストレージ（全体のステートの安全なブランチ）へのアクセスや、`gasMeter`、`ブロックの高さ`、`コンセンサスパラメータ`などの便利なオブジェクトや情報が含まれます。
:::

:::note Pre-requisite Readings

* [Anatomy of a Cosmos SDK Application](../basics/00-app-anatomy.md)
* [Lifecycle of a Transaction](../basics/01-tx-lifecycle.md)

:::

## Context Definition

Cosmos SDKの`Context`は、Goのstdlib [`context`](https://pkg.go.dev/context)をベースにしたカスタムデータ構造であり、その定義にはCosmos SDK固有のさまざまな追加タイプが含まれています。この`Context`は、モジュールが[`multistore`](./04-store.md#multistore)内のそれぞれの[ストア](./04-store.md#base-layer-kvstores)に簡単にアクセスし、ブロックヘッダやガスメータのようなトランザクションコンテキストを取得できるようにするため、トランザクション処理に不可欠です。

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/types/context.go#L41-L67
```

* **Base Context:** ベースタイプはGoの[Context](https://pkg.go.dev/context)であり、詳細については下記の[Go Contextパッケージ](#go-context-package)セクションで説明されています。
* **Multistore:** 各アプリケーションの`BaseApp`には[`CommitMultiStore`](./04-store.md#multistore)が含まれており、`Context`が作成された際に提供されます。`KVStore()`および`TransientStore()`メソッドを呼び出すことで、モジュールはそれぞれの一意な`StoreKey`を使用して[`KVStore`](./04-store.md#base-layer-kvstores)を取得できます。
* **Header:** [ヘッダ](https://docs.cometbft.com/v0.37/spec/core/data_structures#header)はブロックチェーンのタイプです。ブロックの高さや現在のブロックの提案者など、ブロックチェーンの状態に関する重要な情報を保持しています。
* **Header Hash:** `abci.FinalizeBlock`中に取得される現在のブロックヘッダのハッシュ。
* **Chain ID:** ブロックが関連するブロックチェーンの一意の識別番号。
* **Transaction Bytes:** コンテキストを使用して処理されるトランザクションの`[]byte`表現。すべてのトランザクションは、Cosmos SDKやコンセンサスエンジン（例：CometBFT）のさまざまな部分で処理されますが、いくつかはトランザクションの種類を理解していないことがあります。したがって、トランザクションは一般的な`[]byte`タイプに、[Amino](./05-encoding.md)などの[エンコーディング形式](./05-encoding.md)を使用してマーシャリングされます。
* **Logger:** CometBFTライブラリからの`logger`。ログに関する詳細は[こちら](https://docs.cometbft.com/v0.37/core/configuration)をご覧ください。モジュールは、独自のモジュール固有のロガーを作成するためにこのメソッドを呼び出します。
* **VoteInfo:** ABCIタイプの[`VoteInfo`](https://docs.cometbft.com/master/spec/abci/abci.html#voteinfo)のリストで、バリデータの名前とブロックに署名しているかどうかを示すブール値が含まれています。
* **Gas Meters:** 具体的には、コンテキストを使用して処理されている現在のトランザクション用の[`gasMeter`](../basics/04-gas-fees.md#main-gas-meter)と、それが属しているブロック全体用の[`blockGasMeter`](../basics/04-gas-fees.md#block-gas-meter)が含まれています。ユーザーはトランザクションの実行にいくらの手数料を支払いたいかを指定します。これらのガスメーターは、トランザクションやブロックでこれまでに使用されたガスの量を追跡します。ガスメーターが切れると、実行が停止します。
* **CheckTxモード:** トランザクションを`CheckTx`モードまたは`DeliverTx`モードで処理するかどうかを示すブール値。
* **Min Gas Price:** ノードがトランザクションをブロックに含めるために受け入れる最小[ガス](../basics/04-gas-fees.md)価格。この価格は各ノード個別に設定されたローカルな値であり、したがって**ステート遷移につながるシーケンスで使用されるいかなる関数にも使用してはならない**。
* **Consensus Params:** ABCIタイプの[コンセンサスパラメータ](https://docs.cometbft.com/master/spec/abci/apps.html#consensus-parameters)であり、ブロックの最大ガスなど、ブロックチェーンに関する特定の制限を指定します。
* **Event Manager:** イベントマネージャは、`Context`へのアクセス権を持つ任意の呼び出し元が[`Events`](./08-events.md)を発行できるようにします。モジュールは、さまざまな`Types`や`Attributes`を定義することにより、モジュール固有の`Events`を定義したり、`types/`で見つかる共通の定義を使用したりできます。クライアントはこれらの`Events`に対して購読またはクエリを行うことができます。これらの`Events`は`FinalizeBlock`中に収集され、索引付けのためにCometBFTに返されます。
* **Priority:** トランザクションの優先度であり、`CheckTx`モードでのみ関連します。
* **KV `GasConfig`:** アプリケーションが`KVStore`に対してカスタムの`GasConfig`を設定できるようにします。
* **Transient KV `GasConfig`:** アプリケーションがトランザイアント`KVStore`に対してカスタムの`GasConfig`を設定できるようにします。
* **StreamingManager:** streamingManagerフィールドは、ブロックチェーンによって放出された状態の変更に購読することをモジュールに許可するstreaming managerへのアクセスを提供します。streaming managerは、[ADR 038](https://docs.cosmos.network/main/architecture/adr-038-state-listening)で説明されている状態のリスニングAPIで使用されます。
* **CometInfo:** 現在のブロックに関する情報（ブロックの高さ、時間、ハッシュなど）を含む軽量フィールド。この情報は、エビデンスの検証、歴史的データの提供、ユーザーエクスペリエンスの向上に使用できます。詳細については[こちら](https://github.com/cosmos/cosmos-sdk/blob/main/core/comet/service.go#L14)を参照してください。
* **HeaderInfo:** `headerInfo`フィールドには、現在のブロックヘッダに関する情報（チェーンID、ガスリミット、タイムスタンプなど）が含まれています。詳細については[こちら](https://github.com/cosmos/cosmos-sdk/blob/main/core/header/service.go#L14)を参照してください。



## Go Context Package

基本的な`Context`は[Golang Context Package](https://pkg.go.dev/context)で定義されています。`Context`は、APIとプロセスをまたがるリクエストスコープのデータを持ち運ぶ不変のデータ構造です。Contextは並行性を可能にし、ゴルーチンで使用するために設計されています。

Contextは**不変**であることが意図されており、編集されるべきではありません。代わりに、親のContextから子のContextを`With`関数を使用して作成するのが一般的です。例えば、以下のようになります。

```go
childCtx = parentCtx.WithBlockHeader(header)
```

[Golang Context Package](https://pkg.go.dev/context)のドキュメントは、開発者に対してプロセスの最初の引数として明示的にコンテキスト`ctx`を渡すよう指示しています。

## Store branching

`Context`には`MultiStore`が含まれており、`CacheMultiStore`を使用して分岐とキャッシング機能を提供します（`CacheMultiStore`のクエリは、将来のラウンドトリップを避けるためにキャッシュされます）。
(queries in `CacheMultiStore` are cached to avoid future round trips).

各`KVStore`は、安全で分離されたエフェメラルストレージで分岐しています。プロセスは変更を`CacheMultiStore`に書き込むことができます。状態遷移のシーケンスが問題なく実行された場合、ストアのブランチはシーケンスの終わりに基になるストアにコミットできます。また、何かがうまくいかない場合は、変更を破棄することができます。Contextの使用パターンは次のようになります: 


1. プロセスは、親プロセスからContext `ctx`を受け取ります。これにより、プロセスの実行に必要な情報が提供されます。
2. `ctx.ms`は**ブランチストア**であり、プロセスが実行する間に状態を変更できるようにするために[マルチストア](./04-store.md#multistore)のブランチが作成されます。これにより、実行中に変更を元の`ctx.ms`に反映させずに状態を変更できるため、変更が実行中のある時点で元に戻す必要がある場合に基になるマルチストアを保護するのに役立ちます。
3. プロセスは実行中に`ctx`から読み書きすることができます。必要に応じて、サブプロセスを呼び出して`ctx`を渡すことができます。
4. サブプロセスが返されると、結果が成功か失敗かをチェックします。失敗の場合、何もする必要はありません - ブランチ`ctx`は単に破棄されます。成功した場合、`CacheMultiStore`に行われた変更は`Write()`を介して元の`ctx.ms`にコミットできます。

例えば、[`baseapp`](./00-baseapp.md)の[`runTx`](./00-baseapp.md#runtx-antehandler-runmsgs-posthandler)関数からのスニペットは以下の通りです：

```go
runMsgCtx, msCache := app.cacheTxContext(ctx, txBytes)
result = app.runMsgs(runMsgCtx, msgs, mode)
result.GasWanted = gasWanted
if mode != runTxModeDeliver {
  return result
}
if result.IsOK() {
  msCache.Write()
}
```

以下がプロセスの手順です：

1. トランザクション内のメッセージに対して`runMsgs`を呼び出す前に、`app.cacheTxContext()`を使用してコンテキストとマルチストアをブランチしてキャッシュします。
2. ブランチしたストアを持つ`runMsgCtx`は、`runMsgs`内で使用され、結果を返します。
3. プロセスが[`checkTxMode`](./00-baseapp.md#checktx)で実行されている場合、変更を書き込む必要はありません。結果は直ちに返されます。
4. プロセスが[`deliverTxMode`](./00-baseapp.md#delivertx)で実行されており、結果がすべてのメッセージに対する成功した実行を示している場合、ブランチしたマルチストアは元のストアに書き戻されます。
