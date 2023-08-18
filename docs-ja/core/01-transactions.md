---
sidebar_position: 1
---

# Transactions

:::note Synopsis
`Transactions` are objects created by end-users to trigger state changes in the application.
:::

:::note Pre-requisite Readings

* [Anatomy of a Cosmos SDK Application](../basics/00-app-anatomy.md)

:::

## Transactions

トランザクションは、[コンテキスト](./02-context.md)内で保持されるメタデータと、モジュールのProtobuf [`Msg`サービス](../building-modules/03-msg-services.md)を介してモジュール内の状態変更をトリガする`sdk.Msg`で構成されています。

ユーザーがアプリケーションとやり取りし、ステート変更（例：コインの送信）を行いたい場合、トランザクションを作成します。トランザクションの各`sdk.Msg`は、トランザクションがネットワークにブロードキャストされる前に、適切なアカウントに関連付けられた秘密鍵を使用して署名する必要があります。その後、トランザクションはブロックに含まれ、ネットワークによってコンセンサスプロセスを通じて検証および承認される必要があります。トランザクションのライフサイクルについて詳しくは[こちら](../basics/01-tx-lifecycle.md)をクリックしてください。

## Type Definition

トランザクションオブジェクトは、`Tx`インターフェースを実装したCosmos SDKの型です。

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/types/tx_msg.go#L51-L56
```

以下のメソッドが含まれています：

* **GetMsgs:** トランザクションを展開し、含まれる`sdk.Msg`のリストを返します。1つのトランザクションには1つ以上のメッセージが含まれる場合があり、これはモジュール開発者によって定義されます。
* **ValidateBasic:** ABCIメッセージ [`CheckTx`](./00-baseapp.md#checktx) と [`DeliverTx`](./00-baseapp.md#delivertx) で使用される軽量な[_ステートレス_](../basics/01-tx-lifecycle.md#types-of-checks) チェック。トランザクションが無効でないことを確認します。たとえば、[`auth`](https://github.com/cosmos/cosmos-sdk/tree/main/x/auth) モジュールの `ValidateBasic` 関数は、トランザクションが正しい数の署名者によって署名されていることや、手数料がユーザーの最大値を超えていないことを確認します。[`runTx`](./00-baseapp.md#runtx) が[`auth`](https://github.com/cosmos/cosmos-sdk/tree/main/x/auth/spec) モジュールから作成されたトランザクションをチェックしている場合、まず各メッセージに対して`ValidateBasic`を実行し、その後 `auth` モジュールのAnteHandlerがトランザクション自体に対して`ValidateBasic`を呼び出します。

    :::note
この関数は、廃止された`sdk.Msg`の[`ValidateBasic`](../basics/01-tx-lifecycle.md#ValidateBasic)メソッドとは異なります。これは、メッセージに対して基本的な妥当性チェックのみを実行していました。
    :::

開発者としては、`Tx`を直接操作することはほとんどありません。なぜなら、`Tx`は実際にはトランザクション生成に使用される中間型であり、開発者は代わりに`TxBuilder`インターフェースを好むべきです。詳細については[以下](#transaction-generation)で学ぶことができます。

### Signing Transactions

トランザクション内のすべてのメッセージは、その`GetSigners`で指定されたアドレスによって署名される必要があります。現在、Cosmos SDKではトランザクションの署名を2つの異なる方法で許可しています。

#### `SIGN_MODE_DIRECT` (preferred)

`Tx`インターフェースの最も一般的な実装は、Protobuf `Tx`メッセージであり、`SIGN_MODE_DIRECT`で使用されます。

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/proto/cosmos/tx/v1beta1/tx.proto#L13-L26
```

Protobufシリアライゼーションは決定的ではないため、Cosmos SDKはトランザクションが署名されるピン留めバイトを示す追加の`TxRaw`型を使用します。ユーザーはトランザクションのために有効な`body`と`auth_info`を生成し、これらの2つのメッセージをProtobufを使用してシリアライズできます。その後、`TxRaw`はユーザーの`body`と`auth_info`の正確なバイナリ表現である`body_bytes`と`auth_info_bytes`をピン留めします。トランザクションのすべての署名者によって署名されるドキュメントは`SignDoc`です（[ADR-027](../architecture/adr-027-deterministic-protobuf-serialization.md)を使用して決定的にシリアライズされます）：

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/proto/cosmos/tx/v1beta1/tx.proto#L48-L65
```

すべての署名者によって署名された後、`body_bytes`、`auth_info_bytes`、および`signatures`は`TxRaw`にまとめられ、そのシリアライズされたバイトはネットワーク上でブロードキャストされます。

#### `SIGN_MODE_LEGACY_AMINO_JSON`

`Tx`インターフェースの旧実装は、`x/auth`からの`StdTx`構造体です：

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/auth/migrations/legacytx/stdtx.go#L83-L90
```

すべての署名者によって署名されるドキュメントは`StdSignDoc`です：

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/auth/migrations/legacytx/stdsign.go#L31-L45
```

これはAmino JSONを使用してバイトにエンコードされます。すべての署名が`StdTx`にまとめられると、`StdTx`はAmino JSONを使用してシリアライズされ、これらのバイトはネットワーク上でブロードキャストされます。

#### Other Sign Modes

Cosmos SDKは特定のユースケースに対していくつかの他の署名モードも提供しています。

#### `SIGN_MODE_DIRECT_AUX`

`SIGN_MODE_DIRECT_AUX`は、Cosmos SDK v0.46でリリースされた複数の署名者を対象とする署名モードです。`SIGN_MODE_DIRECT`は、各署名者が`TxBody`と`AuthInfo`の両方に署名することを想定しています（これには他のすべての署名者の署名者情報、つまりアカウントシーケンス、公開鍵、モード情報が含まれます）。一方、`SIGN_MODE_DIRECT_AUX`は、N-1の署名者が`TxBody`と_自分自身の_署名者情報に対して署名することを許可します。さらに、各補助署名者（つまり`SIGN_MODE_DIRECT_AUX`を使用する署名者）は手数料に署名する必要はありません：

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/proto/cosmos/tx/v1beta1/tx.proto#L67-L98
```

ユースケースは、複数の署名者がいるトランザクションで、署名者の1人がすべての署名を収集し、署名をブロードキャストし、手数料を支払う役割を担当し、他の署名者はトランザクション本体に関心を持つだけです。これにより、一般的にはより良いマルチサイニングのUXが可能になります。例えば、Alice、Bob、Charlieが3人の署名者のトランザクションの一部である場合、AliceとBobは両方とも`SIGN_MODE_DIRECT_AUX`を使用して`TxBody`と自分自身の署名者情報に署名できます（`SIGN_MODE_DIRECT`のように他の署名者の情報を収集する追加のステップは不要です）。手数料の指定はサインドキュメント内に必要ありません。その後、CharlieはAliceとBobから両方の署名を収集し、手数料を追加して最終的なトランザクションを作成できます。トランザクションの手数料を支払う者（我々の場合はCharlie）は、手数料に対して署名する必要があり、`SIGN_MODE_DIRECT`または`SIGN_MODE_LEGACY_AMINO_JSON`を使用する必要があります。

具体的なユースケースは、[トランザクションチップ](./14-tips.md)で実装されています。チップを提示する人は、トランザクションにチップを指定するために`SIGN_MODE_DIRECT_AUX`を使用し、実際のトランザクション手数料に署名する必要はありません。その後、手数料を支払いトランザクションをブロードキャストする代わりに、手数料を支払い、トランザクションの手数料としてトランザクションチップを受け取ることができます。

#### `SIGN_MODE_TEXTUAL`

`SIGN_MODE_TEXTUAL`は、ハードウェアウォレットでの署名体験を向上させるための新しい署名モードであり、現在はまだ実装中です。詳細については、[ADR-050](https://github.com/cosmos/cosmos-sdk/pull/10701)を参照してください。

#### Custom Sign modes

Cosmos-SDKに独自のカスタム署名モードを追加する機会があります。署名モードの実装はリポジトリには受け入れられませんが、カスタム署名モードをSignMode列挙体に追加するためのプルリクエストは受け入れることができます。[こちら](https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/proto/cosmos/tx/signing/v1beta1/signing.proto#L17)にSignMode列挙体があります。

## Transaction Process

エンドユーザーがトランザクションを送信するプロセスは次のとおりです：

* トランザクションに含めるメッセージを決定する。
* Cosmos SDKの`TxBuilder`を使用してトランザクションを生成する。
* 利用可能なインターフェースのいずれかを使用してトランザクションをブロードキャストする。

次の段落では、これらのコンポーネントそれぞれについて、この順序で説明します。

### Messages

:::tip
モジュールの`sdk.Msg`は、CometBFTとアプリケーション層間の相互作用を定義する[ABCIメッセージ](https://docs.cometbft.com/v0.37/spec/abci/)とは混同しないでください。
:::

**メッセージ**（または`sdk.Msg`）は、モジュール固有のオブジェクトであり、それが属するモジュールのスコープ内でステートの遷移をトリガーします。モジュールの開発者は、Protobuf [`Msg`サービス](../building-modules/03-msg-services.md)にメソッドを追加してモジュールのメッセージを定義し、対応する`MsgServer`も実装します。

各`sdk.Msg`は、各モジュールの`tx.proto`ファイル内で定義された、1つのProtobuf [`Msg`サービス](../building-modules/03-msg-services.md) RPCに関連付けられています。SDKアプリのルータは、自動的に各`sdk.Msg`を対応するRPCにマッピングします。Protobufは各モジュールの`Msg`サービスに対して`MsgServer`インターフェースを生成し、モジュール開発者がこのインターフェースを実装する必要があります。この設計はモジュール開発者により多くの責任を負わせ、アプリケーション開発者が繰り返しステート遷移ロジックを実装する必要なく、共通の機能を再利用できるようにします。

Protobuf `Msg`サービスと`MsgServer`の実装方法について詳しくは、[こちら](../building-modules/03-msg-services.md)をクリックしてください。

メッセージはステート遷移ロジックの情報を含む一方、トランザクションのその他のメタデータと関連情報は`TxBuilder`と`Context`に保存されます。

### Transaction Generation

`TxBuilder`インターフェースには、トランザクションの生成に密接に関連するデータが含まれており、エンドユーザーはこれを自由に設定して必要なトランザクションを生成できます：

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/client/tx_config.go#L40-L53
```

* トランザクションに含まれるメッセージの配列である[メッセージ](#messages)。
* ユーザーが支払う必要のあるガス量を計算するためのオプションである`GasLimit`。
* トランザクションと一緒に送信されるメモまたはコメントである`Memo`。
* ユーザーが手数料として支払うことを承認する最大金額である`FeeAmount`。
* トランザクションが有効なブロックの高さである`TimeoutHeight`。
* トランザクションのすべての署名者からの署名の配列である`Signatures`。

現在、トランザクションに署名するための2つの署名モードがあるため、`TxBuilder`の実装も2つあります：

* `SIGN_MODE_DIRECT`用の[wrapper](https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/auth/tx/builder.go#L26-L43)、
* `SIGN_MODE_LEGACY_AMINO_JSON`用の[StdTxBuilder](https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/auth/migrations/legacytx/stdtx_builder.go#L14-L17)。

ただし、`TxBuilder`の2つの実装はエンドユーザーから隠蔽されるべきであり、代わりに総合的な`TxConfig`インターフェースの使用を優先する必要があります：

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/client/tx_config.go#L24-L34
```

`TxConfig`は、トランザクションを管理するためのアプリ全体の設定です。最も重要なのは、各トランザクションを`SIGN_MODE_DIRECT`または`SIGN_MODE_LEGACY_AMINO_JSON`で署名するかどうかの情報を保持していることです。`txBuilder := txConfig.NewTxBuilder()`と呼び出すことで、適切な署名モードで新しい`TxBuilder`が作成されます。

`TxBuilder`が上記で公開されたセッターで正しく設定された場合、`TxConfig`はバイトを正しくエンコードするのも適切に処理します（再度、`SIGN_MODE_DIRECT`または`SIGN_MODE_LEGACY_AMINO_JSON`を使用）。以下に、`TxEncoder()`メソッドを使用してトランザクションを生成およびエンコードするための疑似コードスニペットを示します：

```go
txBuilder := txConfig.NewTxBuilder()
txBuilder.SetMsgs(...) // and other setters on txBuilder

bz, err := txConfig.TxEncoder()(txBuilder.GetTx())
// bz are bytes to be broadcasted over the network
```

### Broadcasting the Transaction

トランザクションバイトが生成されたら、現在は3つの方法でそれをブロードキャストすることができます。

#### CLI

アプリケーション開発者は、[コマンドラインインターフェース](../core/07-cli.md)、[gRPCおよび/またはRESTインターフェース](../core/06-grpc_rest.md)を作成して、アプリケーションの`./cmd`フォルダに通常配置します。これらのインターフェースは、ユーザーがコマンドラインを介してアプリケーションと対話できるようにします。

[コマンドラインインターフェース](../building-modules/09-module-interfaces.md#cli)の場合、モジュール開発者は、アプリケーションのトップレベルトランザクションコマンド`TxCmd`の子として追加するサブコマンドを作成します。CLIコマンドは、実際にはトランザクション処理のすべてのステップを1つの簡単なコマンドにまとめています：メッセージの作成、トランザクションの生成、およびブロードキャスト。具体的な例については、[ノードとの対話](../run-node/02-interact-node.md)セクションを参照してください。CLIを使用して作成されたトランザクションの例は次のようになります：

```bash
simd tx send $MY_VALIDATOR_ADDRESS $RECIPIENT 1000stake
```

#### gRPC

[gRPC](https://grpc.io)は、Cosmos SDKのRPCレイヤーの主要なコンポーネントです。主な使用方法は、モジュールの[`Query`サービス](../building-modules/04-query-services.md)のコンテキストで行われます。ただし、Cosmos SDKはいくつかの他のモジュールに依存しないgRPCサービスも公開しており、その1つが`Tx`サービスです：

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/proto/cosmos/tx/v1beta1/service.proto
```

`Tx`サービスは、トランザクションをシミュレートしたり、トランザクションをクエリしたりするためのユーティリティ関数をいくつか公開しており、またトランザクションをブロードキャストするための1つのメソッドも公開しています。

トランザクションのブロードキャストとシミュレートの例については、[こちら](../run-node/03-txs.md#programmatically-with-go)を参照してください。

#### REST

各gRPCメソッドには対応するRESTエンドポイントがあり、[gRPC-gateway](https://github.com/grpc-ecosystem/grpc-gateway)を使用して生成されます。したがって、gRPCを使用する代わりに、`POST /cosmos/tx/v1beta1/txs`エンドポイントを介して同じトランザクションをHTTPでブロードキャストすることもできます。

例は[こちら](../run-node/03-txs.md#using-rest)で確認できます。

#### CometBFT RPC

上記に示した3つのメソッドは、実際にはCometBFT RPCの`/broadcast_tx_{async,sync,commit}`エンドポイントの上位抽象化です。これは、必要に応じてCometBFT RPCエンドポイントを直接使用してトランザクションをブロードキャストすることもできることを意味します。
