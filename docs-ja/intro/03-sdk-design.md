---
sidebar_position: 1
---

# Main Components of the Cosmos SDK

Cosmos SDKは、CometBFTの上で安全なステートマシンの開発を支援するフレームワークです。Cosmos SDKは、本質的にはGolangでの[ABCI](./02-sdk-app-architecture.md#abci)のボイラープレート実装です。データを永続化するための[`multistore`](../core/04-store.md#multistore)やトランザクションを処理するための[`router`](../core/00-baseapp.md#routing)も含まれています。

以下は、Cosmos SDK上で構築されたアプリケーションがCometBFTから`DeliverTx`経由でトランザクションを処理する簡略化されたビューです。

1. CometBFTの合意エンジンから受け取った`transactions`をデコードします（CometBFTは`[]bytes`のみを扱うことを覚えておいてください）。
2. `transactions`から`messages`を抽出し、基本的な正当性チェックを行います。
3. 各メッセージを適切なモジュールにルーティングして処理します。
4. 状態変更をコミットします。


## `baseapp`

`baseapp`はCosmos SDKアプリケーションのボイラープレート実装です。これには、ABCIの実装が含まれており、基盤となる合意エンジンとの接続を処理します。一般的に、Cosmos SDKアプリケーションは、`baseapp`を拡張することで、[`app.go`](../basics/00-app-anatomy.md#core-application-file)に埋め込まれます。

以下は、Cosmos SDKのデモアプリである`simapp`からの例です。

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/simapp/app.go#L170-L212
```

`baseapp`の目的は、ストアと拡張可能な状態機械との間に安全なインターフェースを提供することです。同時に、状態機械に関する情報を最小限に抑え、ABCIに忠実なものとしています。

`baseapp`の詳細については、[こちら](../core/00-baseapp.md)をクリックしてください。

## Multistore

Cosmos SDKは、状態の永続化のための[`multistore`](../core/04-store.md#multistore)を提供しています。multistoreを使用すると、開発者は任意の数の[`KVStores`](../core/04-store.md#base-layer-kvstores)を宣言できます。これらの`KVStores`は値として`[]byte`型のみを受け入れるため、任意のカスタム構造体は、格納する前に[コーデック](../core/05-encoding.md)を使用してマーシャリングする必要があります。

multistoreの抽象化は、状態を異なる区画に分割するために使用され、それぞれのモジュールによって管理されます。multistoreの詳細については、[こちら](../core/04-store.md#multistore)をクリックしてください。

## Modules

Cosmos SDKの力はそのモジュラリティにあります。Cosmos SDKアプリケーションは、相互運用可能な複数のモジュールを集約して構築されます。各モジュールは状態の一部を定義し、独自のメッセージ/トランザクションプロセッサを含みます。一方、Cosmos SDKは各メッセージをそれぞれのモジュールにルーティングする責任があります。

ここでは、有効なブロックで受信されたトランザクションが各フルノードのアプリケーションでどのように処理されるかの簡略したビューを示します：

```text
                                      +
                                      |
                                      |  Transaction relayed from the full-node's
                                      |  CometBFT engine to the node's application
                                      |  via DeliverTx
                                      |
                                      |
                +---------------------v--------------------------+
                |                 APPLICATION                    |
                |                                                |
                |     Using baseapp's methods: Decode the Tx,    |
                |     extract and route the message(s)           |
                |                                                |
                +---------------------+--------------------------+
                                      |
                                      |
                                      |
                                      +---------------------------+
                                                                  |
                                                                  |
                                                                  |  Message routed to
                                                                  |  the correct module
                                                                  |  to be processed
                                                                  |
                                                                  |
+----------------+  +---------------+  +----------------+  +------v----------+
|                |  |               |  |                |  |                 |
|  AUTH MODULE   |  |  BANK MODULE  |  | STAKING MODULE |  |   GOV MODULE    |
|                |  |               |  |                |  |                 |
|                |  |               |  |                |  | Handles message,|
|                |  |               |  |                |  | Updates state   |
|                |  |               |  |                |  |                 |
+----------------+  +---------------+  +----------------+  +------+----------+
                                                                  |
                                                                  |
                                                                  |
                                                                  |
                                       +--------------------------+
                                       |
                                       | Return result to CometBFT
                                       | (0=Ok, 1=Err)
                                       v
```

各モジュールは小さなステートマシンと見なすことができます。開発者は、モジュールが処理するステートのサブセットを定義し、ステートを変更するカスタムメッセージタイプを定義する必要があります（*注意:* `messages`は`baseapp`によって`transactions`から抽出されます）。一般的に、各モジュールは自身の`KVStore`を`multistore`に宣言して、定義されたステートのサブセットを永続化します。ほとんどの開発者は、独自のモジュールを構築する際に他の第三者モジュールにアクセスする必要があります。Cosmos SDKはオープンなフレームワークであるため、一部のモジュールは悪意を持つ可能性があり、これによりモジュール間の相互作用について考えるためのセキュリティ原則が必要です。これらの原則は[オブジェクト・キャパビリティ](../core/10-ocap.md)に基づいています。実際には、各モジュールが他のモジュールのためにアクセス制御リストを保持する代わりに、各モジュールは他のモジュールに渡すことができる特別なオブジェクト「キーパー」を実装し、事前に定義された一連の機能を付与できるようにします。

Cosmos SDKモジュールは、Cosmos SDKの`x/`フォルダに定義されています。一部のコアモジュールは次のとおりです：

* `x/auth`：アカウントと署名の管理に使用されます。
* `x/bank`：トークンとトークンの転送を有効にするために使用されます。
* `x/staking` + `x/slashing`：Proof-Of-Stakeブロックチェーンの構築に使用されます。

`x/`の既存のモジュールに加えて、誰でもアプリケーションで使用できるものとして、Cosmos SDKは独自のカスタムモジュールを構築できるようにします。詳細は[tutorialの例](https://tutorials.cosmos.network/)をご覧ください。

