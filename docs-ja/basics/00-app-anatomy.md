---
sidebar_position: 1
---

:::note 概要
このドキュメントでは、Cosmos SDKアプリケーションの中核部分について説明します。このドキュメント全体を通して、プレースホルダーアプリケーションである `app` という名前のアプリケーションが使用されています。
:::

## Node Client

デーモン、または [フルノードクライアント](../core/03-node.md) は、Cosmos SDKベースのブロックチェーンの中核プロセスです。ネットワークの参加者は、このプロセスを実行してステートマシンを初期化し、他のフルノードに接続し、新しいブロックが到着するたびにステートマシンを更新します。


```text
                ^  +-------------------------------+  ^
                |  |                               |  |
                |  |  State-machine = Application  |  |
                |  |                               |  |   Built with Cosmos SDK
                |  |            ^      +           |  |
                |  +----------- | ABCI | ----------+  v
                |  |            +      v           |  ^
                |  |                               |  |
Blockchain Node |  |           Consensus           |  |
                |  |                               |  |
                |  +-------------------------------+  |   CometBFT
                |  |                               |  |
                |  |           Networking          |  |
                |  |                               |  |
                v  +-------------------------------+  v
```

ブロックチェーンのフルノードは、通常「デーモン」を意味する `-d` サフィックスが付いたバイナリとして提供されます（例： `appd` は `app` のためのもの、 `gaiad` は `gaia` のためのもの）。このバイナリは、`./cmd/appd/` に配置されたシンプルな [`main.go`](../core/03-node.md#main-function) 関数を実行することでビルドされます。通常、この操作は [Makefile](#dependencies-and-makefile) を介して行われます。

メインバイナリがビルドされたら、ノードは [`start` コマンド](../core/03-node.md#start-command) を実行して起動できます。このコマンド関数は主に次の三つのことを行います：

1. [`app.go`](#core-application-file) で定義されたステートマシンのインスタンスを作成します。
2. `~/.app/data` フォルダに保存されている `db` から抽出された最新の既知のステートでステートマシンを初期化します。この時点で、ステートマシンの高さは `appBlockHeight` です。
3. 新しい CometBFT インスタンスを作成して開始します。ノードは他のノードとハンドシェイクを行います。これによって最新の `blockHeight` を取得し、ローカルの `appBlockHeight` よりも大きい場合にはその高さに合わせてブロックを再生して同期します。ノードはジェネシスから開始し、CometBFT は `app` に対して ABCI を介して `InitChain` メッセージを送信し、これによって [`InitChainer`](#initchainer) がトリガーされます。

:::note
CometBFT インスタンスを起動する際、ジェネシスファイルは高さ `0` であり、ジェネシスファイル内のステートはブロック高さ `1` でコミットされます。ノードのステートをクエリする際に、ブロック高さ 0 をクエリするとエラーが返されます。
:::

## Core Application File

一般的に、ステートマシンの中核部分は `app.go` というファイルに定義されます。このファイルには、**アプリケーションの型定義**とそれを**作成して初期化するための関数**が主に含まれます。

### Type Definition of the Application

`app.go` で最初に定義されるのは、アプリケーションの `type` です。一般的に、これは次のパーツで構成されています：

* **[`baseapp`](../core/00-baseapp.md)への参照。** `app.go` で定義されたカスタムアプリケーションは `baseapp` の拡張です。CometBFT からアプリケーションにリレーされるトランザクションは、`app` が `baseapp` のメソッドを使用して適切なモジュールにルーティングします。 `baseapp` は、[ABCIメソッド](https://docs.cometbft.com/v0.37/spec/abci/)と[ルーティングロジック](../core/00-baseapp.md#routing)を含むアプリケーションのほとんどのコアロジックを実装しています。
* **ストアキーのリスト。** ステート全体を含む[ストア](../core/04-store.md)は、Cosmos SDK内で[`multistore`](../core/04-store.md#multistore)（ストアのストア）として実装されています。各モジュールは、マルチストア内の1つ以上のストアを使用してステートの一部を永続化します。これらのストアは、`app` 型で宣言される特定のキーでアクセスできます。これらのキーは、`keepers` とともに[オブジェクト機能モデル](../core/10-ocap.md)のCosmos SDKの中心です。
* **モジュールの`keeper`リスト。** 各モジュールは[`keeper`](../building-modules/06-keeper.md)と呼ばれる抽象化を定義しており、このモジュールのストアの読み取りと書き込みを処理します。1つのモジュールの`keeper`メソッドは、他のモジュールから呼び出すことができます（承認されている場合）。そのため、これらのメソッドはアプリケーションの型で宣言され、インターフェースとして他のモジュールにエクスポートされており、後者は承認された関数にのみアクセスできます。
* **[`appCodec`](../core/05-encoding.md)への参照。** アプリケーションの`appCodec`は、データ構造をシリアライズおよびデシリアライズするために使用されます。ストアは`[]bytes`のみ永続化できるためです。デフォルトのコーデックは[Protocol Buffers](../core/05-encoding.md)です。
* **[`legacyAmino`](../core/05-encoding.md)コーデックへの参照。** Cosmos SDKの一部の部分は、上記の`appCodec`を使用するように移行されていません。それらは引き続きAminoを使用するようにハードコードされています。他の一部は明示的に後方互換性のためにAminoを使用します。これらの理由により、アプリケーションには引き続きレガシーなAminoコーデックへの参照があります。ただし、Aminoコーデックは今後のSDKリリースで削除される予定です。
* **[モジュールマネージャ](../building-modules/01-module-manager.md#manager)と[基本モジュールマネージャ](../building-modules/01-module-manager.md#basicmanager)への参照。** モジュールマネージャは、アプリケーションのモジュールのリストを含むオブジェクトです。これにより、これらのモジュールに関連する操作が容易になります。例えば、[`Msg`サービス](../core/00-baseapp.md#msg-services)と[gRPC `Query`サービス](../core/00-baseapp.md#grpc-query-services)の登録、または[`InitChainer`](#initchainer)、[`BeginBlocker`および`EndBlocker`](#beginblocker-and-endblocker)のためのモジュール間での実行順序の設定などです。

アプリケーションの型定義の例を、Cosmos SDKのデモとテスト用途に使用される`simapp`から示します：

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/simapp/app.go#L173-L212
```

### Constructor Function

`app.go` には、前述のセクションで定義された型の新しいアプリケーションを構築するコンストラクタ関数も定義されています。この関数は、アプリケーションのデーモンコマンドの[`start`コマンド](../core/03-node.md#start-command)で使用されるために`AppCreator`シグネチャを満たす必要があります。

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/server/types/app.go#L66-L68
```

この関数によって実行される主なアクションは次のとおりです。

* 新しい[`codec`](../core/05-encoding.md)をインスタンス化し、[基本マネージャー](../building-modules/01-module-manager.md#basicmanager)を使用してアプリケーションの各モジュールの`codec`を初期化します。
* `baseapp` インスタンス、codec、および適切なストアキーを参照する新しいアプリケーションをインスタンス化します。
* アプリケーションの全ての[`keeper`](#keeper)オブジェクトを、各モジュールの`NewKeeper`関数を使用してインスタンス化します。なお、`keeper`は正しい順序でインスタンス化する必要があり、あるモジュールの`NewKeeper`は別のモジュールの`keeper`への参照を必要とする可能性があります。
* アプリケーションの[モジュールマネージャー](../building-modules/01-module-manager.md#manager)を、アプリケーションの各モジュールの[`AppModule`](#application-module-interface)オブジェクトとともにインスタンス化します。
* モジュールマネージャーを使用して、アプリケーションの[`Msg`サービス](../core/00-baseapp.md#msg-services)、[gRPC `Query`サービス](../core/00-baseapp.md#grpc-query-services)、[レガシー`Msg`ルート](../core/00-baseapp.md#routing)、および[レガシークエリルート](../core/00-baseapp.md#query-routing)を初期化します。CometBFTを介してABCIを通じてアプリケーションに中継されるトランザクションは、ここで定義されたルートを使用して、適切なモジュールの[`Msg`サービス](#msg-services)にルーティングされます。同様に、gRPCクエリリクエストがアプリケーションによって受信されると、gRPCルートを使用して適切なモジュールの[`gRPCクエリサービス`](#grpc-query-services)にルーティングされます。Cosmos SDKは引き続き、レガシー`Msg`とレガシーCometBFTクエリをサポートしており、それぞれレガシー`Msg`ルートとレガシークエリルートを使用してルーティングされます。
* モジュールマネージャーを使用して、[アプリケーションのモジュールの不変条件](../building-modules/07-invariants.md)を登録します。不変条件は、ブロックの最後に評価される変数（たとえばトークンの総供給量など）です。不変条件をチェックするプロセスは、[`InvariantsRegistry`](../building-modules/07-invariants.md#invariant-registry)と呼ばれる特別なモジュールを介して行われます。不変条件の値は、モジュールで定義された予測値と等しい必要があります。値が予測値と異なる場合、不変条件レジストリで定義された特別なロジックがトリガーされます（通常、チェーンが停止します）。これにより、重大なバグが見逃されないようにし、修正が難しい長期間の影響を生じないようにするのに役立ちます。
* モジュールマネージャーを使用して、[アプリケーションのモジュール](#application-module-interface)の`InitGenesis`、`BeginBlocker`、および`EndBlocker`関数の実行順序を設定します。すべてのモジュールがこれらの関数を実装しているわけではないことに注意してください。
* 残りのアプリケーションパラメータを設定します：
    * [`InitChainer`](#initchainer)：アプリケーションの最初の起動時に初期化されます。
    * [`BeginBlocker`、`EndBlocker`](#beginblocker-and-endlbocker)：すべてのブロックの始まりと終わりに呼び出されます。
    * [`anteHandler`](../core/00-baseapp.md#antehandler)：手数料と署名の検証を処理するために使用されます。
* ストアをマウントします。
* アプリケーションを返します。

注意：コンストラクタ関数はアプリのインスタンスを作成するだけであり、実際のステートは、ノードが再起動される場合は`~/.app/data`フォルダから引き継がれ、ノードが初めて起動される場合はジェネシスファイルから生成されます。

`simapp`からのアプリケーションコンストラクタの例を参照してください：

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/simapp/app.go#L223-L575
```

### InitChainer

`InitChainer`は、アプリケーションのステートをジェネシスファイルから初期化する関数です（例：ジェネシスアカウントのトークン残高）。これは、アプリケーションがCometBFTエンジンから`InitChain`メッセージを受け取ったときに呼び出されます。これは、ノードが`appBlockHeight == 0`（つまり、ジェネシス時）に起動されるときに行われます。アプリケーションは、[`SetInitChainer`](https://pkg.go.dev/github.com/cosmos/cosmos-sdk/baseapp#BaseApp.SetInitChainer)メソッドを使用して、その[コンストラクタ](#constructor-function)内で`InitChainer`を設定する必要があります。

一般的に、`InitChainer`は、アプリケーションの各モジュールの[`InitGenesis`](../building-modules/08-genesis.md#initgenesis)関数から主に構成されています。これは、モジュールマネージャの`InitGenesis`関数を呼び出すことで行われ、そのモジュールマネージャが含む各モジュールの`InitGenesis`関数を呼び出します。モジュールの`InitGenesis`関数を呼び出す順序は、モジュールマネージャの[モジュールマネージャ](../building-modules/01-module-manager.md)の`SetOrderInitGenesis`メソッドを使用して設定する必要があります。これは、[アプリケーションのコンストラクタ](#application-constructor)で行われ、`SetInitChainer`の前に`SetOrderInitGenesis`を呼び出す必要があります。

`simapp`からの`InitChainer`の例を参照してください：

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/simapp/app.go#L626-L634
```

### BeginBlocker and EndBlocker

Cosmos SDKは、開発者に対してアプリケーションの一部としてコードの自動実行を実装する機能を提供しています。これは、`BeginBlocker`と`EndBlocker`という2つの関数を介して実装されます。これらの関数は、アプリケーションがCometBFTコンセンサスエンジンから`FinalizeBlock`メッセージを受け取ったときに呼び出され、それぞれブロックの開始と終了時に行われます。アプリケーションは、[`SetBeginBlocker`](https://pkg.go.dev/github.com/cosmos/cosmos-sdk/baseapp#BaseApp.SetBeginBlocker)メソッドと[`SetEndBlocker`](https://pkg.go.dev/github.com/cosmos/cosmos-sdk/baseapp#BaseApp.SetEndBlocker)メソッドを使用して、その[コンストラクタ](#constructor-function)内で`BeginBlocker`と`EndBlocker`を設定する必要があります。

一般的に、`BeginBlocker`および`EndBlocker`関数は、アプリケーションの各モジュールの[`BeginBlock`と`EndBlock`](../building-modules/05-beginblock-endblock.md)関数から主に構成されています。これは、モジュールマネージャの`BeginBlock`と`EndBlock`関数を呼び出すことで行われ、そのモジュールマネージャが含む各モジュールの`BeginBlock`と`EndBlock`関数を呼び出します。モジュールの`BeginBlock`および`EndBlock`関数が呼び出される順序は、モジュールマネージャの`SetOrderBeginBlockers`および`SetOrderEndBlockers`メソッドを使用して設定する必要があります。これは、[モジュールマネージャ](../building-modules/01-module-manager.md)を介して[アプリケーションのコンストラクタ](#application-constructor)で行われ、`SetOrderBeginBlockers`および`SetOrderEndBlockers`メソッドは、`SetBeginBlocker`および`SetEndBlocker`関数よりも前に呼び出す必要があります。

余談ですが、アプリケーション特化型のブロックチェーンは決定論的です。開発者は`BeginBlocker`または`EndBlocker`で非決定性を導入しないよう注意する必要があり、また、`BeginBlocker`および`EndBlocker`の実行コストは[ガス](./04-gas-fees.md)に制約されないため、計算負荷が高すぎないよう注意する必要があります。

`simapp`からの`BeginBlocker`および`EndBlocker`関数の例を参照してください：

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/simapp/app.go#L613-L620
```

### Register Codec

`EncodingConfig`構造体は、`app.go`ファイルの最後の重要な部分です。この構造体の目的は、アプリケーション全体で使用されるコーデックを定義することです。


```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/simapp/params/encoding.go#L9-L16
```

以下は、各フィールドの説明です：

* `InterfaceRegistry`（インターフェースレジストリ）：`InterfaceRegistry`は、Protobufコーデックが、[`google.protobuf.Any`](https://github.com/protocolbuffers/protobuf/blob/master/src/google/protobuf/any.proto)を使用してエンコードおよびデコード（「アンパック」とも言います）するインターフェースを処理するために使用されます。`Any`は、具体的な型を実装する型の`type_url`（名前）と`value`（エンコードされたバイト）を含む構造体と考えることができます。`InterfaceRegistry`は、`Any`から安全にアンパックできるインターフェースと実装を登録するためのメカニズムを提供します。各アプリケーションモジュールは、モジュール固有のインターフェースと実装を登録するために`RegisterInterfaces`メソッドを実装します。
    * `Any`について詳しくは、[ADR-019](../architecture/adr-019-protobuf-state-encoding.md)をご覧ください。
    * より詳細に進むと、Cosmos SDKは、Protobuf仕様の実装である[`gogoprotobuf`](https://github.com/cosmos/gogoproto)を使用しています。デフォルトで、[gogo protobufの`Any`の実装](https://pkg.go.dev/github.com/cosmos/gogoproto/types)は、`Any`に詰められた値を具体的なGo型にデコードするために[グローバル型登録](https://github.com/cosmos/gogoproto/blob/master/proto/properties.go#L540)を使用します。これにより、依存関係ツリー内の任意の悪意あるモジュールがグローバルなprotobufレジストリに型を登録し、それを`type_url`フィールドで参照するトランザクションによって読み込まれ、アンマーシャル化される脆弱性が導入されます。詳細については、[ADR-019](../architecture/adr-019-protobuf-state-encoding.md)を参照してください。
* `Codec`（コーデック）：Cosmos SDK全体で使用されるデフォルトのコーデックです。エンコードおよびデコードに使用される`BinaryCodec`と、ユーザーにデータを出力するために使用される`JSONCodec`から構成されます（たとえば、[CLI](#cli)で使用されます）。デフォルトでは、SDKはProtobufを`Codec`として使用します。
* `TxConfig`（トランザクション設定）：`TxConfig`は、クライアントがアプリケーション定義の具体的なトランザクション型を生成するために使用できるインターフェースを定義します。現在、SDKは2つのトランザクションタイプを処理しています：`SIGN_MODE_DIRECT`（オーバーザワイヤーエンコーディングとしてProtobufバイナリを使用）および`SIGN_MODE_LEGACY_AMINO_JSON`（Aminoに依存）。トランザクションについて詳しくは[こちら](../core/01-transactions.md)をご覧ください。
* `Amino`（アミノ）：Cosmos SDKの一部のレガシーパーツは、後方互換性のためにまだAminoを使用しています。各モジュールは、モジュール固有の型をAmino内で登録するための`RegisterLegacyAmino`メソッドを公開します。この`Amino`コーデックはもはやアプリケーション開発者によって使用されるべきではなく、将来のリリースで削除されます。

アプリケーションは、独自のエンコーディング設定を作成する必要があります。
`simapp`の`simappparams.EncodingConfig`の例を次に示します：


```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/simapp/params/encoding.go#L11-L16
```

## Modules

[モジュール](../building-modules/01-intro.md)は、Cosmos SDKアプリケーションの中核です。それらは、ステートマシン内にネストされたステートマシンと考えることができます。トランザクションが基盤となるCometBFTエンジンからABCIを介してアプリケーションに中継されると、[`baseapp`](../core/00-baseapp.md)によって適切なモジュールにルーティングされて処理されます。このパラダイムにより、開発者は複雑なステートマシンを簡単に構築できます。なぜなら、必要なモジュールの多くはすでに存在していることが多いからです。**開発者にとって、Cosmos SDKアプリケーションを構築する際に最も重要な作業の多くは、まだ存在しないアプリケーションに必要なカスタムモジュールを構築し、既に存在するモジュールと統合して一貫したアプリケーションに組み込むことです**。アプリケーションディレクトリでは、モジュールを`x/`フォルダに格納するのが標準的な方法です（すでに構築されたモジュールを含むCosmos SDKの`x/`フォルダとは異なることに注意してください）。

### Application Module Interface

モジュールは、Cosmos SDKで定義された[インターフェース](../building-modules/01-module-manager.md#application-module-interfaces)である、[`AppModuleBasic`](../building-modules/01-module-manager.md#appmodulebasic)および[`AppModule`](../building-modules/01-module-manager.md#appmodule)を実装する必要があります。前者はモジュールの基本的な非依存要素（`codec`など）を実装し、後者はモジュールのほとんどのメソッドを処理します（他のモジュールの`keeper`への参照が必要なメソッドも含む）。`AppModule`と`AppModuleBasic`の両方の型は、慣習的に`module.go`という名前のファイルに定義されます。

`AppModule`は、モジュール上で有用なメソッドのコレクションを公開し、モジュールを一貫したアプリケーションに組み合わせるのを容易にします。これらのメソッドは、アプリケーションのモジュールコレクションを管理する[`モジュールマネージャ`](../building-modules/01-module-manager.md#manager)から呼び出されます。

### `Msg` Services

各アプリケーションモジュールは、メッセージを処理する1つの`Msg`サービスと、クエリを処理するgRPC `Query`サービスの2つの[Protobufサービス](https://developers.google.com/protocol-buffers/docs/proto#services)を定義します。モジュールをステートマシンと考えると、`Msg`サービスはステート遷移RPCメソッドのセットです。
各Protobuf `Msg`サービスメソッドは、`sdk.Msg`インターフェースを実装する必要がある、1:1の関連性を持つProtobufリクエストタイプに関連付けられています。
注意すべきは、`sdk.Msg`が[トランザクション](../core/01-transactions.md)にまとめられ、各トランザクションには1つ以上のメッセージが含まれていることです。

有効なトランザクションブロックがフルノードに受信されると、CometBFTはそれぞれのトランザクションを[`DeliverTx`](https://docs.cometbft.com/v0.37/spec/abci/abci++_app_requirements#specifics-of-responsedelivertx)を介してアプリケーションに中継します。その後、アプリケーションはトランザクションを処理します：

1. トランザクションを受信した後、アプリケーションはまず`[]byte`からアンマーシャルします。
2. その後、トランザクションに関して[手数料支払いと署名](./04-gas-fees.md#antehandler)などのいくつかの情報を検証し、トランザクションに含まれる`Msg`(s)を抽出します。
3. `sdk.Msg`はProtobufの[`Any`](#register-codec)を使用してエンコードされます。各`Any`の`type_url`を分析して、baseappの`msgServiceRouter`は`sdk.Msg`を対応するモジュールの`Msg`サービスにルーティングします。
4. メッセージが正常に処理された場合、ステートが更新されます。

詳細については、[トランザクションライフサイクル](./01-tx-lifecycle.md)を参照してください。

モジュール開発者は、独自のモジュールを構築する際にカスタムの`Msg`サービスを作成します。一般的な慣習は、`Msg` Protobufサービスを`tx.proto`ファイルに定義することです。例えば、`x/bank`モジュールは、トークンを転送するための2つのメソッドを持つサービスを定義しています：

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/proto/cosmos/bank/v1beta1/tx.proto#L13-L36
```

サービスメソッドは、モジュールのステートを更新するために`keeper`を使用します。

各モジュールは、[`AppModule`インターフェース](#application-module-interface)の一部として`RegisterServices`メソッドも実装する必要があります。このメソッドは、生成されたProtobufコードが提供する`RegisterMsgServer`関数を呼び出すべきです。


### gRPC `Query` Services

gRPC `Query` サービスを使用することで、ユーザーは[gRPC](https://grpc.io)を通じてステートをクエリできます。これらはデフォルトで有効になっており、[`app.toml`](../run-node/01-run-node.md#configuring-the-node-using-apptoml-and-configtoml)内の `grpc.enable` および `grpc.address` フィールドで設定できます。

gRPC `Query` サービスは、モジュールのProtobuf定義ファイル内、具体的には`query.proto`内で定義されます。`query.proto` 定義ファイルは、単一の `Query` [Protobufサービス](https://developers.google.com/protocol-buffers/docs/proto#services)を公開します。各gRPCクエリエンドポイントは、`Query` サービス内の `rpc` キーワードで始まるサービスメソッドに対応します。

Protobufは、各モジュールごとに `QueryServer` インターフェースを生成します。これにはすべてのサービスメソッドが含まれています。それから、モジュールの[`keeper`](#keeper)はこの `QueryServer` インターフェースを実装する必要があります。各サービスメソッドの具体的な実装は、対応するgRPCクエリエンドポイントのハンドラです。

最後に、各モジュールは[`AppModule`インターフェース](#application-module-interface)の一部として `RegisterServices` メソッドも実装する必要があります。このメソッドは、生成されたProtobufコードが提供する `RegisterQueryServer` 関数を呼び出すべきです。

### Keeper

[`Keepers`](../building-modules/06-keeper.md) は、それぞれのモジュールのストアの管理者です。モジュールのストアへの読み書きを行う際には、必ずそのモジュールの `keeper` のメソッドを介する必要があります。これは、Cosmos SDKの[object-capabilities](../core/10-ocap.md)モデルによって保証されています。ストアのキーを保持するオブジェクトだけがアクセスでき、モジュールの `keeper` だけがモジュールのストアのキーを保持すべきです。

`Keepers` は一般的に `keeper.go` というファイルに定義されます。このファイルには `keeper` の型定義とメソッドが含まれます。

`keeper` の型定義は通常、以下の要素から構成されます：

* マルチストア内のモジュールのストアへの**キー**。
* 他のモジュールの `keeper` への**参照**。これは、他のモジュールのストアにアクセスする必要がある場合にのみ必要です（読み書きのいずれか）。
* アプリケーションの**コーデック**への参照。`keeper` は、ストアが値として`[]bytes`のみを受け入れるため、値を格納する前に構造体をマーシャリングしたり、取得時にアンマーシャリングしたりする際にコーデックが必要です。

型定義とともに、`keeper.go` ファイルのもう一つの重要な部分は `keeper` のコンストラクタ関数である `NewKeeper` です。この関数は、上記で定義された型の新しい `keeper` をインスタンス化し、`codec` を保持し、`keys` をストアし、必要に応じて他のモジュールの `keeper` を参照するものです。`NewKeeper` 関数は、[アプリケーションのコンストラクタ](#constructor-function) から呼び出されます。ファイルの残りの部分は、主にゲッターとセッターからなる `keeper` のメソッドを定義しています。

### Command-Line, gRPC Services and REST Interfaces

各モジュールは、コマンドラインコマンド、gRPCサービス、およびRESTルートを定義して、それらを[アプリケーションのインターフェース](#application-interfaces)を介してエンドユーザーに公開します。これにより、エンドユーザーはモジュールで定義されたタイプのメッセージを作成したり、モジュールが管理するステートの一部をクエリしたりできます。

#### CLI

一般的に、[モジュールに関連するコマンド](../building-modules/09-module-interfaces.md#cli)は、モジュールのフォルダ内の`client/cli`というフォルダに定義されます。CLIは、`client/cli/tx.go`および`client/cli/query.go`に定義された、トランザクションとクエリという2つのカテゴリのコマンドに分割されます。両方のコマンドは、[Cobraライブラリ](https://github.com/spf13/cobra)を基にして構築されています：

* トランザクションコマンドは、ユーザーが新しいトランザクションを生成してブロックに含め、最終的にステートを更新できるようにします。各[メッセージタイプ](#message-types)に対して1つのコマンドを作成する必要があります。コマンドは、エンドユーザーによって提供されるパラメータでメッセージのコンストラクタを呼び出し、それをトランザクションにラップします。Cosmos SDKは署名と他のトランザクションメタデータの追加を処理します。
* クエリは、ユーザーがモジュールで定義されたステートの一部をクエリするのに使用されます。クエリコマンドは、[アプリケーションのクエリルーター](../core/00-baseapp.md#query-routing)にクエリを転送し、それを適切な[クエリエ](#querier)（提供された`queryRoute`パラメータによって）にルーティングします。

#### gRPC

[gRPC](https://grpc.io)は、複数の言語でサポートされているモダンなオープンソースの高性能RPCフレームワークです。これは、外部クライアント（ウォレット、ブラウザ、他のバックエンドサービスなど）がノードと対話するための推奨される方法です。

各モジュールは、[モジュールのProtobuf `query.proto`ファイル](#grpc-query-services)で定義されるgRPCエンドポイントを公開できます。サービスメソッドは、その名前、入力引数、および出力レスポンスによって定義されます。モジュールは次のアクションを実行する必要があります：

* `AppModuleBasic`に`RegisterGRPCGatewayRoutes`メソッドを定義して、クライアントのgRPCリクエストをモジュール内の適切なハンドラに接続します。
* 各サービスメソッドに対して、対応するハンドラを定義します。ハンドラは、gRPCリクエストを処理するために必要なコアロジックを実装し、`keeper/grpc_query.go`ファイルに配置されます。

#### gRPC-gateway REST Endpoints

一部の外部クライアントはgRPCを使用したくない場合があります。この場合、Cosmos SDKはgRPCゲートウェイサービスを提供しており、各gRPCサービスを対応するRESTエンドポイントとして公開します。詳細については、[grpc-gateway](https://grpc-ecosystem.github.io/grpc-gateway/)のドキュメントを参照してください。

RESTエンドポイントは、ProtobufファイルでgRPCサービスと一緒に、Protobufアノテーションを使用して定義されます。RESTクエリを公開したいモジュールは、`rpc`メソッドに`google.api.http`アノテーションを追加する必要があります。SDKで定義されたすべてのRESTエンドポイントは、デフォルトで`/cosmos/`プレフィックスで始まるURLを持っています。

また、Cosmos SDKはこれらのRESTエンドポイントのための[Swagger](https://swagger.io/)定義ファイルを生成するための開発エンドポイントも提供しています。このエンドポイントは、`app.toml`（[アプリケーションの設定ファイル](../run-node/01-run-node.md#configuring-the-node-using-apptoml-and-configtoml)）の`api.swagger`キーの下で有効にすることができます。

## Application Interface

[インターフェース](#command-line-grpc-services-and-rest-interfaces)は、エンドユーザーがフルノードクライアントと対話するためのものです。これは、フルノードからデータをクエリしたり、新しいトランザクションを作成して送信したりすることを意味します。これらのトランザクションはフルノードによってリレーされ、最終的にはブロックに含まれます。

主なインターフェースは[コマンドラインインターフェース](../core/07-cli.md)です。Cosmos SDKアプリケーションのCLIは、アプリケーションで使用される各モジュールで定義された[CLIコマンド](#cli)を集約して構築されます。アプリケーションのCLIはデーモン（例：`appd`）と同じであり、`appd/main.go`というファイルで定義されます。このファイルには以下が含まれます：

* **`main()`関数**は、`appd`インターフェースクライアントを構築するために実行されます。この関数は各コマンドを準備し、それらを`rootCmd`に追加してからビルドします。`appd`のルートにおいて、この関数は`status`、`keys`、`config`などの一般的なコマンド、クエリコマンド、トランザクションコマンド、および`rest-server`などのジェネリックコマンドを追加します。
* **クエリコマンド**は、`queryCmd`関数を呼び出すことで追加されます。この関数は、各アプリケーションモジュールで定義されたクエリコマンド（`main()`関数から`sdk.ModuleClients`の配列として渡される）だけでなく、ブロックやバリデータのクエリなどの他の低レベルクエリコマンドも含むCobraコマンドを返します。クエリコマンドは、CLIのコマンド`appd query [query]`を使用して呼び出されます。
* **トランザクションコマンド**は、`txCmd`関数を呼び出すことで追加されます。`queryCmd`と同様に、この関数は、各アプリケーションモジュールで定義されたトランザクションコマンドだけでなく、トランザクションの署名やブロードキャストなどの他の低レベルトランザクションコマンドも含むCobraコマンドを返します。トランザクションコマンドは、CLIのコマンド`appd tx [tx]`を使用して呼び出されます。

[Cosmos Hub](https://github.com/cosmos/gaia)からアプリケーションのメインコマンドラインファイルの例を見てみましょう。

```go reference
https://github.com/cosmos/gaia/blob/26ae7c2/cmd/gaiad/cmd/root.go#L39-L80
```

## Dependencies and Makefile

このセクションは任意です。開発者は依存関係マネージャーやプロジェクトのビルド方法を自由に選択できます。ただし、現在最も使用されているバージョン管理のフレームワークは[`go.mod`](https://github.com/golang/go/wiki/Modules)です。これにより、アプリケーション全体で使用される各ライブラリが正しいバージョンでインポートされることが保証されます。

以下は、[Cosmos Hub](https://github.com/cosmos/gaia)の`go.mod`の例です。

```go reference
https://github.com/cosmos/gaia/blob/26ae7c2/go.mod#L1-L28
```

アプリケーションをビルドするためには、一般的に[Makefile](https://en.wikipedia.org/wiki/Makefile)が使用されます。Makefileは主に、`go.mod`がアプリケーションの2つのエントリポイント（[`appd`](#node-client)と[`appd`](#application-interface)）をビルドする前に実行されることを確認します。

以下は、[Cosmos HubのMakefile](https://github.com/cosmos/gaia/blob/main/Makefile)の例です。
