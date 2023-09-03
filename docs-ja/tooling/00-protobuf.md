---
sidebar_position: 1
---

# Protocol Buffers

Cosmos SDKがプロトコルバッファを広範に使用していることは知られていますが、このドキュメントはcosmos-sdkでの使用方法に関するガイドを提供することを目的としています。

protoファイルを生成するために、Cosmos SDKはDockerイメージを使用します。このイメージもすべてのユーザーが使用できるよう提供されています。最新バージョンは`ghcr.io/cosmos/proto-builder:0.12.x`です。

以下は、protobufファイルの生成、リンティング、およびフォーマットに関するCosmos SDKのコマンドの例であり、任意のアプリケーションのmakefileで再利用できます。

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/Makefile#L411-L432
```

protobufファイルを生成するためのスクリプトは`scripts/`ディレクトリにあります。

```shell reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/scripts/protocgen.sh
```

## Buf

[Buf](https://buf.build)は、`protoc`ツールチェーンの複雑さを抽象化し、多くのエコシステムとの整合性を保つためのツールです。cosmos-sdkリポジトリ内にはいくつかのbufプレフィックスを持つファイルがあります。まずトップレベルから各ディレクトリを詳しく見ていきましょう。

### Workspace

At the root level directory a workspace is defined using [buf workspaces](https://docs.buf.build/configuration/v1/buf-work-yaml). This helps if there are one or more protobuf containing directories in your project. 

Cosmos SDKの例:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/main/buf.work.yaml#L6-L9
```

### Proto Directory

次は、すべてのprotobufファイルが格納されている`proto/`ディレクトリです。ここには、異なる目的に応じて定義された多くのbufファイルがあります。

```bash
├── README.md
├── buf.gen.gogo.yaml
├── buf.gen.pulsar.yaml
├── buf.gen.swagger.yaml
├── buf.lock
├── buf.md
├── buf.yaml
├── cosmos
└── tendermint
```

上の図は、Cosmos SDKの `proto/`ディレクトリ内のすべてのファイルとディレクトリを示しています。

#### `buf.gen.gogo.yaml`

`buf.gen.gogo.yaml`は、モジュール内で使用するためのprotobufファイルの生成方法を定義します。このファイルは[gogoproto](https://github.com/gogo/protobuf)を使用しており、google go-protoジェネレータとは異なるジェネレータです。これは、さまざまなオブジェクトの操作をよりエルゴノミックにするためのもので、エンコードおよびデコードのステップがより高性能です。

```go reference
https://github.com/cosmos/cosmos-sdk/blob/main/proto/buf.gen.gogo.yaml#L1-L9
```

:::tip
`gen`ファイルの定義方法の例は[こちら](https://docs.buf.build/tour/generate-go-code)で見ることができます。
:::

#### `buf.gen.pulsar.yaml`

`buf.gen.pulsar.yaml`は、[新しいgolang apiv2のprotobuf](https://go.dev/blog/protobuf-apiv2)を使用してprotobufファイルがどのように生成されるべきかを定義しています。このジェネレータは、google go-protoジェネレータの代わりに使用されます。なぜなら、それはCosmos SDKアプリケーションのための追加のヘルパーを持っており、google go-protoジェネレータよりもエンコードとデコードが高性能であるからです。このジェネレータの開発は[こちら](https://github.com/cosmos/cosmos-proto)

```go reference
https://github.com/cosmos/cosmos-sdk/blob/main/proto/buf.gen.pulsar.yaml#L1-L18
```

:::tip
`gen`ファイルの定義方法の例は[こちら](https://docs.buf.build/tour/generate-go-code)で見ることができます。
:::

#### `buf.gen.swagger.yaml`

`buf.gen.swagger.yaml`は、チェーンのクエリとメッセージのためのswaggerドキュメントを生成します。これは、クエリとmsgサーバで定義されたREST APIエンドポイントのみを定義します。この例は[こちら](https://github.com/cosmos/cosmos-sdk/blob/main/proto/cosmos/bank/v1beta1/query.proto#L19)で見ることができます。

```go reference
https://github.com/cosmos/cosmos-sdk/blob/main/proto/buf.gen.swagger.yaml#L1-L6
```

:::tip
`gen`ファイルの定義方法の例は[こちら](https://docs.buf.build/tour/generate-go-code)で見ることができます。
:::

#### `buf.lock`

これは、`.gen`ファイルに必要な依存関係に基づいて自動生成されるファイルです。現在のものをコピーする必要はありません。もしcosmos-sdkのproto定義に依存している場合、Cosmos SDKのための新しいエントリを提供する必要があります。使用する必要がある依存関係は`buf.build/cosmos/cosmos-sdk`です。

```go reference
https://github.com/cosmos/cosmos-sdk/blob/main/proto/buf.lock#L1-L16
```

#### `buf.yaml`

`buf.yaml`は、[パッケージの名前](https://github.com/cosmos/cosmos-sdk/blob/main/proto/buf.yaml#L3)、使用する[破損チェッカー](https://docs.buf.build/tour/detect-breaking-changes)、およびprotobufファイルを[lintする方法](https://docs.buf.build/tour/lint-your-api)を定義しています。

```go reference
https://github.com/cosmos/cosmos-sdk/blob/main/proto/buf.yaml#L1-L24
```

Cosmos SDKのprotobufファイルのためにさまざまなリンターを使用しています。このリポジトリもCIでこれをチェックします。

GitHubアクションへの参照は[こちら](https://github.com/cosmos/cosmos-sdk/blob/main/.github/workflows/proto.yml#L1-L32)で見ることができます。

```go reference
https://github.com/cosmos/cosmos-sdk/blob/main/.github/workflows/proto.yml#L1-L32
```
