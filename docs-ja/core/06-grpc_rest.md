---
sidebar_position: 1
---

# gRPC, REST, and CometBFT Endpoints

:::note Synopsis
このドキュメントでは、ノードが公開するすべてのエンドポイントについて概要を説明します。gRPC、REST、およびその他のエンドポイントも含まれます。
:::

## An Overview of All Endpoints

各ノードは、ユーザーがノードと対話するために以下のエンドポイントを公開します。各エンドポイントは異なるポートで提供されます。各エンドポイントの設定方法の詳細は、該当するエンドポイントのセクションで提供されます。

* gRPCサーバー（デフォルトポート：`9090`）
* RESTサーバー（デフォルトポート：`1317`）
* CometBFT RPCエンドポイント（デフォルトポート：`26657`）

:::tip
ノードは他にもCometBFT P2Pエンドポイントや[Prometheusエンドポイント](https://docs.cometbft.com/v0.37/core/metrics)など、Cosmos SDKと直接関係のないエンドポイントを公開します。これらのエンドポイントに関する詳細については、[CometBFTドキュメント](https://docs.cometbft.com/v0.37/core/configuration)を参照してください。
:::

## gRPC Server

Cosmos SDKでは、Protobufが主要な[エンコーディング](./encoding)ライブラリです。これにより、Cosmos SDKに組み込むことができるさまざまなProtobufベースのツールが提供されます。その中の一つが[grpc](https://grpc.io)で、これはモダンなオープンソースの高性能RPCフレームワークであり、いくつかの言語でのクライアントサポートが十分に提供されています。

各モジュールは、状態クエリを定義する[Protobuf `Query`サービス](../building-modules/02-messages-and-queries.md#queries)を公開します。状態クエリを実行する`Query`サービスとトランザクションをブロードキャストするためのトランザクションサービスは、以下のアプリケーション内の関数を介してgRPCサーバーに接続されます：

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/server/types/app.go#L46-L48
```

注意: gRPCを介して[Protobuf `Msg`サービス](../building-modules/02-messages-and-queries.md#messages)エンドポイントを公開することはできません。トランザクションは、gRPCを使用してブロードキャストされる前に、CLIまたはプログラムで生成および署名する必要があります。詳細については、[トランザクションの生成、署名、およびブロードキャスト](../run-node/03-txs.md)を参照してください。

`grpc.Server`は具体的なgRPCサーバーであり、すべてのgRPCクエリリクエストとブロードキャストトランザクションリクエストを生成して提供します。このサーバーは`~/.simapp/config/app.toml`内で設定できます：

* `grpc.enable = true|false`フィールドは、gRPCサーバーが有効になるかどうかを定義します。デフォルトは`true`です。
* `grpc.address = {string}`フィールドは、サーバーがバインドする`ip:port`を定義します。デフォルトは`localhost:9090`です。

:::tip
`~/.simapp`は、ノードの設定とデータベースが保存されるディレクトリです。デフォルトでは、`~/.{app_name}`に設定されています。
:::

gRPCサーバーが開始されると、gRPCクライアントを使用してリクエストを送信できます。いくつかの例は、[ノードと対話する](../run-node/02-interact-node.md#using-grpc)チュートリアルで提供されています。

Cosmos SDKで提供されるすべての利用可能なgRPCエンドポイントの概要については、[Protobufドキュメント](https://buf.build/cosmos/cosmos-sdk)を参照してください。

## REST Server

Cosmos SDKは、gRPC-gatewayを介してRESTルートをサポートしています。

すべてのルートは、`~/.simapp/config/app.toml`内の以下のフィールドで構成されます：

* `api.enable = true|false`フィールドは、RESTサーバーを有効にするかどうかを定義します。デフォルトは`false`です。
* `api.address = {string}`フィールドは、サーバーがバインドする`ip:port`を定義します。デフォルトは`tcp://localhost:1317`です。
* いくつかの追加のAPI設定オプションは、コメントと共に`~/.simapp/config/app.toml`に定義されており、直接そのファイルを参照してください。

### gRPC-gateway REST Routes

さまざまな理由でgRPCを使用できない場合（たとえば、Webアプリケーションを構築しており、ブラウザがgRPCが構築されているHTTP2をサポートしていない場合）、Cosmos SDKはgRPC-gatewayを介してRESTルートを提供します。

[gRPC-gateway](https://grpc-ecosystem.github.io/grpc-gateway/)は、gRPCエンドポイントをRESTエンドポイントとして公開するためのツールです。Protobuf `Query`サービスで定義された各gRPCエンドポイントに対して、Cosmos SDKはRESTの等価なエンドポイントを提供します。たとえば、残高のクエリは、`/cosmos.bank.v1beta1.QueryAllBalances` gRPCエンドポイントを介して行うこともできますし、代わりにgRPC-gatewayの`"/cosmos/bank/v1beta1/balances/{address}"` RESTエンドポイントを介して行うこともできます。いずれも同じ結果が返されます。Protobuf `Query`サービスで定義された各RPCメソッドに対して、対応するRESTエンドポイントはオプションとして定義されています：

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/proto/cosmos/bank/v1beta1/query.proto#L23-L30
```

アプリケーション開発者にとって、gRPC-gatewayのRESTルートはRESTサーバーに接続する必要があります。これはModuleManager上で`RegisterGRPCGatewayRoutes`関数を呼び出すことで行われます。

### Swagger

APIサーバーの`/swagger`ルートの下には、[Swagger](https://swagger.io/)（またはOpenAPIv2）仕様ファイルが公開されています。Swaggerは、サーバーが提供するAPIエンドポイントを説明し、各エンドポイントに関する説明、入力引数、戻り値の型など、多くの情報を含むオープンな仕様です。

`/swagger`エンドポイントを有効にするには、`~/.simapp/config/app.toml`内の`api.swagger`フィールドで設定できます。デフォルトではtrueに設定されています。

アプリケーション開発者にとって、カスタムモジュールに基づいて独自のSwagger定義を生成したい場合があります。Cosmos SDKの[Swagger生成スクリプト](https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/scripts/protoc-swagger-gen.sh)は、良い出発点です。


## CometBFT RPC

Cosmos SDKとは独立して、CometBFTもRPCサーバーを公開しています。このRPCサーバーは、`~/.simapp/config/config.toml`の`rpc`テーブル内のパラメータを調整して設定できます。デフォルトのリスニングアドレスは`tcp://localhost:26657`です。CometBFTのすべてのRPCエンドポイントのOpenAPI仕様は、[こちら](https://docs.cometbft.com/main/rpc/)で利用可能です。

Some CometBFT RPC endpoints are directly related to the Cosmos SDK:

* `/abci_query`: このエンドポイントはアプリケーションのステートをクエリします。`path`パラメーターとして、次の文字列を送信できます：
    * `/cosmos.bank.v1beta1.Query/AllBalances`など、任意のProtobuf完全修飾サービスメソッド。その場合、`data`フィールドにはメソッドのリクエストパラメーターをProtobufを使用してバイト列としてエンコードしたものを含める必要があります。
    * `/app/simulate`: これによりトランザクションがシミュレートされ、使用されたガスなどの情報が返されます。
    * `/app/version`: これによりアプリケーションのバージョンが返されます。
    * `/store/{storeName}/key`: これにより、名前付きストアが`data`パラメーターで表されるキーに関連付けられたデータを直接クエリできます。
    * `/store/{storeName}/subspace`: これにより、名前付きストアがキーが`data`パラメーターの値をプレフィックスとして持つキー/値ペアを直接クエリできます。
    * `/p2p/filter/addr/{port}`: これにより、ノードのP2Pピアのアドレスポートでフィルタリングされたリストが返されます。
    * `/p2p/filter/id/{id}`: これにより、ノードのP2PピアがIDでフィルタリングされたリストが返されます。
* `/broadcast_tx_{aync,async,commit}`: これらの3つのエンドポイントは、トランザクションを他のピアにブロードキャストします。CLI、gRPC、およびRESTはすべて、[トランザクションのブロードキャスト方法](./01-transactions.md#broadcasting-the-transaction)を公開していますが、その内部ではこれらの3つのCometBFT RPCが使用されています。

## Comparison Table

| 名前           | 利点                                                                                                   | 欠点                                                                                                            |
| -------------- | ------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| gRPC           | - 各種言語でコード生成されたスタブを使用できる<br /> - ストリーミングと双方向通信をサポート（HTTP2）<br /> - 小さなワイヤーバイナリサイズ、高速な送信 | - HTTP2ベースであり、ブラウザでは使用できない<br /> - 学習コストがある（主にProtobufによるもの）                      |
| REST           | - 広く普及している<br/> - すべての言語でクライアントライブラリがあり、高速な実装                       | - 単一の要求-応答通信しかサポートしていない（HTTP1.1）<br/> - 大きなワイヤーメッセージサイズ（JSON）                  |
| CometBFT RPC   | - 簡単に使用できる                                                                                    | - 大きなワイヤーメッセージサイズ（JSON）                                                                    |
