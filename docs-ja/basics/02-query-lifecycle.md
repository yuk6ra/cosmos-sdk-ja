---
sidebar_position: 1
---

# Query Lifecycle

:::note Synopsis
このドキュメントでは、Cosmos SDKアプリケーションにおけるクエリのライフサイクルについて説明します。ユーザーインターフェースからアプリケーションストアへ、そして戻るまでの流れを示します。このクエリは`MyQuery`と呼ばれます。
:::

:::note Pre-requisite Readings

* [Transaction Lifecycle](./01-tx-lifecycle.md)
:::

## Query Creation

[**クエリ**](../building-modules/02-messages-and-queries.md#queries)は、エンドユーザーがインターフェースを介して行う情報の要求であり、フルノードによって処理されます。ユーザーはネットワークに関する情報、アプリケーション自体の情報、およびアプリケーションのステートに関する情報をアプリケーションのストアやモジュールから直接クエリできます。クエリは[トランザクション](../core/01-transactions.md)とは異なります（ライフサイクルは[こちら](./01-tx-lifecycle.md)を参照）。特に、クエリは処理にコンセンサスを必要とせず（ステート遷移をトリガーしないため）、一つのフルノードで完全に処理されることがあります。

クエリのライフサイクルを説明する目的で、クエリ `MyQuery` は、特定の委任者アドレスによって行われた委任のリストを `simapp` というアプリケーションからリクエストしていると仮定しましょう。予想されるように、[`staking`](../modules/staking/README.md)モジュールがこのクエリを処理します。しかし、最初にユーザーが `MyQuery` をどのように作成できるかにはいくつかの方法があります。

### CLI

アプリケーションの主なインターフェースはコマンドラインインターフェースです。ユーザーはフルノードに接続し、CLIを直接自分のマシンから実行します。CLIは直接フルノードとやり取りします。ユーザーは端末から以下のコマンドを入力して`MyQuery`を作成します。


```bash
simd query staking delegations <delegatorAddress>
```

このクエリコマンドは、[`staking`](../modules/staking/README.md)モジュールの開発者によって定義され、アプリケーション開発者によってCLIの作成時にサブコマンドのリストに追加されました。

一般的なフォーマットは次のようになります：

```bash
simd query [moduleName] [command] <arguments> --flag <flagArg>
```

`--node`（CLIが接続するフルノード）などの値を提供するために、ユーザーは[`app.toml`](../run-node/01-run-node.md#configuring-the-node-using-apptoml-and-configtoml)設定ファイルを使用してそれらを設定するか、フラグとして提供することができます。

CLIは特定のコマンドセットを理解し、アプリケーション開発者によって階層構造で定義されます。[ルートコマンド](../core/07-cli.md#root-command)（`simd`）からコマンドの種類（`Myquery`）、コマンドを含むモジュール（`staking`）、そしてコマンド自体（`delegations`）まで、CLIはこのコマンドをどのモジュールが処理するかを正確に把握し、呼び出しを直接そこに渡します。

### gRPC

ユーザーがクエリを行うための別のインターフェースは、[gRPC](https://grpc.io)サーバーへの[gRPCリクエスト](../core/06-grpc_rest.md#grpc-server)です。エンドポイントは、`.proto`ファイル内の[Protocol Buffers](https://developers.google.com/protocol-buffers)サービスメソッドとして定義されており、Protobuf独自の言語に依存しないインターフェース定義言語（IDL）で記述されています。Protobufエコシステムは、`*.proto`ファイルからさまざまな言語にコードを生成するためのツールを開発しました。これらのツールを使用すると、簡単にgRPCクライアントを構築できます。

そのようなツールの一つが[grpcurl](https://github.com/fullstorydev/grpcurl)で、このクライアントを使用して`MyQuery`のgRPCリクエストは次のようになります：

```bash
grpcurl \
    -plaintext                                           # We want results in plain test
    -import-path ./proto \                               # Import these .proto files
    -proto ./proto/cosmos/staking/v1beta1/query.proto \  # Look into this .proto file for the Query protobuf service
    -d '{"address":"$MY_DELEGATOR"}' \                   # Query arguments
    localhost:9090 \                                     # gRPC server endpoint
    cosmos.staking.v1beta1.Query/Delegations             # Fully-qualified service method name
```

### REST

ユーザーがクエリを行うための別のインターフェースは、[RESTサーバー](../core/06-grpc_rest.md#rest-server)へのHTTPリクエストを通じて行うことができます。RESTサーバーは、[gRPC-gateway](https://github.com/grpc-ecosystem/grpc-gateway)を使用して、Protocol Buffersサービスから完全に自動生成されます。

`MyQuery`の例としてのHTTPリクエストは次のようになります：

```bash
GET http://localhost:1317/cosmos/staking/v1beta1/delegators/{delegatorAddr}/delegations
```

## How Queries are Handled by the CLI

前述の例は、外部ユーザーがノードと対話してその状態をクエリする方法を示しています。クエリの正確なライフサイクルをより詳細に理解するために、CLIがクエリを準備する方法とノードがそれを処理する方法について詳しく見てみましょう。ユーザーの視点からの対話は少し異なりますが、基本的な機能はほぼ同じです。なぜなら、それらはすべてモジュール開発者によって定義された同じコマンドの実装だからです。この処理ステップは、CLI、gRPC、またはRESTサーバー内で行われ、`client.Context`を大いに活用します。

### Context

CLIコマンドの実行中に最初に作成されるものは、`client.Context`です。`client.Context`は、ユーザー側でリクエストを処理するために必要なすべてのデータを格納するオブジェクトです。特に、`client.Context`には次の情報が格納されます：

* **Codec**: アプリケーションで使用される[エンコーダー/デコーダー](../core/05-encoding.md)は、CometBFT RPCリクエストを作成する前にパラメータとクエリをマーシャリングし、返された応答をJSONオブジェクトにアンマーシャリングするために使用されます。CLIで使用されるデフォルトのコーデックはProtobufです。
* **アカウントデコーダー**: [`auth`](../modules/auth/README.md)モジュールからのアカウントデコーダーは、`[]byte`をアカウントに変換します。
* **RPCクライアント**: リクエストがリレーされるCometBFT RPCクライアントまたはノード。
* **キーリング**: トランザクションに署名し、キーを使用して他の操作を処理するために使用される[キーマネージャー](../basics/03-accounts.md#keyring)。
* **出力ライター**: 応答を出力するために使用される[ライター](https://pkg.go.dev/io/#Writer)。
* **設定**: ユーザーがこのコマンドに対して設定したフラグが含まれます。これには`--height`（クエリするブロックチェーンの高さを指定するもの）や、JSON応答にインデントを追加することを示す`--indent`などがあります。

`client.Context`には、RPCクライアントを取得し、クエリをフルノードにリレーするためのABCIコールを行う「Query()」などのさまざまな関数も含まれています。

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/client/context.go#L25-L68
```

`client.Context`の主な役割は、エンドユーザーとの対話中に使用されるデータを格納し、このデータとの対話に使用するメソッドを提供することです。この役割は、クエリがフルノードで処理される前と後の両方で使用されます。特に、`MyQuery`を処理する際に、`client.Context`はクエリのパラメータをエンコードし、フルノードを取得し、出力を書き込むために使用されます。クエリは、アプリケーションに関する知識を持たず、特定の型を理解しないフルノードにリレーされるため、クエリは`[]byte`形式にエンコードされる必要があります。フルノード（RPCクライアント）自体は、ユーザーのCLIが接続されているノードを知っている`client.Context`を使用して取得されます。クエリはこのフルノードにリレーされて処理されます。最後に、`client.Context`には、応答が返されたときに出力を書き込むための`Writer`が含まれています。これらのステップは、後のセクションで詳しく説明されています。

### Arguments and Route Creation

このライフサイクルの段階では、ユーザーはクエリに含めたいすべてのデータを含むCLIコマンドを作成しています。`client.Context`は、`MyQuery`の残りのプロセスで支援するために存在します。次のステップは、コマンドまたはリクエストを解析し、引数を抽出し、すべてをエンコードすることです。これらのステップは、ユーザーが対話するインターフェース内でユーザー側で行われます。

#### Encoding

私たちの場合（アドレスの委任をクエリする場合）、`MyQuery`には唯一の引数として[アドレス](./03-accounts.md#addresses)`delegatorAddress`が含まれています。ただし、リクエストには最終的には`[]byte`しか含まれないため、これはアプリケーションタイプの特定の知識を持たないフルノード（例：CometBFT）のコンセンサスエンジンにリレーされます。そのため、`client.Context`の`codec`を使用してアドレスをマーシャルします。

以下は、CLIコマンドのコードの例です：

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/staking/client/cli/query.go#L315-L318
```

#### gRPC Query Client Creation

Cosmos SDKは、Protobufサービスから生成されたコードを活用してクエリを行います。`staking`モジュールの`MyQuery`サービスは`queryClient`を生成し、CLIはこれを使用してクエリを実行します。以下が関連するコードです：

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/staking/client/cli/query.go#L308-L343
```

内部では、`client.Context`には`Query()`関数があり、事前に設定されたノードを取得し、クエリをリレーするために使用されます。この関数は、クエリの完全修飾サービスメソッド名をパスとして（この場合は：`/cosmos.staking.v1beta1.Query/Delegations`）、引数をパラメータとして取ります。まず、このクエリをリレーするためにユーザーによって設定されたRPCクライアント（[**ノード**](../core/03-node.md)と呼ばれる）を取得し、`ABCIQueryOptions`（ABCI呼び出しのためにフォーマットされたパラメータ）を作成します。その後、ノードはABCI呼び出し、`ABCIQueryWithOptions()`を行うために使用されます。

以下がコードの例です：

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/client/query.go#L79-L113
```

## RPC

`ABCIQueryWithOptions()`を呼び出すことで、`MyQuery`は[フルノード](../core/05-encoding.md)に受信され、その後リクエストが処理されます。RPCはフルノードのコンセンサスエンジン（例：CometBFT）に行われるものの、クエリはコンセンサスの一部ではなく、ネットワーク全体にブロードキャストされることはありません。なぜなら、クエリはネットワークが合意する必要のあるものではないためです。

ABCIクライアントとCometBFT RPCについて詳しくは、[CometBFTのドキュメント](https://docs.cometbft.com/v0.37/spec/rpc/)をご覧ください。

## Application Query Handling

クエリが基盤となるコンセンサスエンジンからリレーされてフルノードに受信されると、その時点でアプリケーション特化型のタイプを理解し、ステートのコピーを持つ環境内で処理されることになります。[`baseapp`](../core/00-baseapp.md)はABCI [`Query()`](../core/00-baseapp.md#query) 関数を実装し、gRPCクエリを処理します。クエリルートは解析され、既存のサービスメソッドの完全修飾名と一致すると、おそらくモジュールの1つに含まれるサービスメソッドにリレーされます。その後、`baseapp`はリクエストを関連するモジュールに中継します。

`MyQuery`は`staking`モジュールのProtobuf完全修飾サービスメソッド名を持つため（`/cosmos.staking.v1beta1.Query/Delegations`を思い出してください）、`baseapp`はまずパスを解析し、次に独自の内部`GRPCQueryRouter`を使用して対応するgRPCハンドラーを取得し、クエリをモジュールにルーティングします。gRPCハンドラーは、このクエリを認識し、アプリケーションのストアから適切な値を取得し、応答を返す役割を担います。クエリサービスについて詳しくは[こちら](../building-modules/04-query-services.md)をご覧ください。

クエリャーから結果を受け取ったら、`baseapp`はユーザーに応答を返すプロセスを開始します。

## Response

`Query()`はABCI関数であるため、`baseapp`は応答を[`abci.ResponseQuery`](https://docs.cometbft.com/master/spec/abci/abci.html#query-2)型として返します。`client.Context`の`Query()`ルーチンが応答を受け取ります。

### CLI Response

アプリケーションの[`codec`](../core/05-encoding.md)は応答をJSONにアンマーシャルし、`client.Context`は出力をコマンドラインに表示し、出力タイプ（テキスト、JSON、またはYAML）などの設定を適用します。

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/client/context.go#L341-L349
```

以上で完了です！クエリの結果は、CLIによってコンソールに出力されます。
