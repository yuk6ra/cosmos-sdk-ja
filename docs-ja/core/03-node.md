---
sidebar_position: 1
---

# Node Client (Daemon)

:::note Synopsis
Cosmos SDKアプリケーションの主要なエンドポイントは、デーモンクライアント、別名フルノードクライアントです。フルノードはジェネシスファイルから開始してステートマシンを実行します。同じクライアントを実行しているピアに接続し、トランザクションやブロック提案、署名を受信および中継します。フルノードは、Cosmos SDKで定義されたアプリケーションと、ABCIを介してアプリケーションに接続されたコンセンサスエンジンで構成されています。
:::

:::note Pre-requisite Readings

* [Anatomy of an SDK application](../basics/00-app-anatomy.md)

:::

## `main` function

任意のCosmos SDKアプリケーションのフルノードクライアントは、`main`関数を実行して構築されます。クライアントは一般的に、アプリケーション名に`-d`接尾辞を追加して命名されます（例：アプリケーション名`app`の場合、`appd`となります）。また、`main`関数は`./appd/cmd/main.go`ファイルに定義されます。この関数を実行すると、一連のコマンドが付属する実行可能な`appd`が作成されます。アプリ名が`app`の場合、メインコマンドは[`appd start`](#start-command)であり、これによってフルノードが起動します。

一般的に、開発者は`main.go`関数を以下の構造で実装します：

* 最初に、アプリケーションのために[`encodingCodec`](./05-encoding.md)がインスタンス化されます。
* 次に、`config`が取得され、構成パラメータが設定されます。主に、[address](../basics/03-accounts.md#addresses)のBech32プレフィックスが設定されます。

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/types/config.go#L14-L29
```

* [cobra](https://github.com/spf13/cobra)を使用して、フルノードクライアントのルートコマンドが作成されます。その後、アプリケーションのカスタムコマンドは、すべて`rootCmd`の`AddCommand()`メソッドを使用して追加されます。
* `server.AddCommands()`メソッドを使用して、デフォルトのサーバーコマンドを`rootCmd`に追加します。これらのコマンドは、Cosmos SDKレベルで定義された標準コマンドです。これらは、すべてのCosmos SDKベースのアプリケーションで共有されるべきです。最も重要なコマンドである[`start`コマンド](#start-command)も含まれます。
* `executor`を準備して実行します。
  
```go reference
https://github.com/cometbft/cometbft/blob/v0.37.0/libs/cli/setup.go#L74-L78
```

以下は、デモ目的のためのCosmos SDKアプリケーションである`simmapp`アプリケーションの`main`関数の例です：

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/simapp/simd/main.go
```

## `start` command

`start`コマンドは、Cosmos SDKの`/server`フォルダに定義されています。これはフルノードクライアントのルートコマンドに[`main`関数](#main-function)で追加され、エンドユーザーがノードを起動するために呼び出されます：

```bash
# For an example app named "app", the following command starts the full-node.
appd start

# Using the Cosmos SDK's own simapp, the following commands start the simapp node.
simd start
```

再確認ですが、フルノードは3つのコンセプチュアルなレイヤーで構成されています。ネットワーキングレイヤー、コンセンサスレイヤー、およびアプリケーションレイヤーです。最初の2つは一般的にコンセンサスエンジン（デフォルトでCometBFT）としてまとめられる一方、3番目はCosmos SDKのヘルプを得て定義されたステートマシンです。現在、Cosmos SDKはデフォルトのコンセンサスエンジンとしてCometBFTを使用しており、したがって、startコマンドはCometBFTノードを起動するために実装されています。

`start`コマンドの流れは非常にシンプルです。最初に、`context`から`config`を取得して`db`を開きます（デフォルトでは[`leveldb`](https://github.com/syndtr/goleveldb)インスタンス）。この`db`には、アプリケーションの最新の既知の状態が含まれています（アプリケーションが最初に起動された場合は空です）。

`db`を使用して、`start`コマンドは`appCreator`関数を使用してアプリケーションの新しいインスタンスを作成します：

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/server/start.go#L220
```

注意しておいてください。`appCreator`は`AppCreator`のシグネチャを満たす関数です：

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/server/types/app.go#L68
```

実際には、[アプリケーションのコンストラクタ](../basics/00-app-anatomy.md#constructor-function)が`appCreator`として渡されます。

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/simapp/simd/cmd/root_v2.go#L294-L308
```

その後、`app`のインスタンスは新しいCometBFTノードをインスタンス化するために使用されます：

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/server/start.go#L341-L378
```

CometBFTノードは`app`で作成できる理由は、後者が[`baseapp`](./00-baseapp.md)を拡張しているため、[`abci.Application`インターフェース](https://github.com/cometbft/cometbft/blob/v0.37.0/abci/types/application.go#L9-L35)（`app`が[`baseapp`](./00-baseapp.md)を拡張しているため）を満たすからです。`node.New`メソッドの一部として、CometBFTはアプリケーションの高さ（つまり、ジェネシスからのブロック数）がCometBFTノードの高さと等しいことを確認します。これら2つの高さの間の違いは常に負またはnullである必要があります。それが厳密に負の場合、`node.New`は、アプリケーションの高さがCometBFTノードの高さに達するまでブロックを再生するでしょう。最後に、アプリケーションの高さが`0`である場合、CometBFTノードはジェネシスファイルから状態を初期化するためにアプリケーションの[`InitChain`](./00-baseapp.md#initchain)を呼び出します。

CometBFTノードがインスタンス化され、アプリケーションと同期されると、ノードを起動できます：

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/server/start.go#L350-L352
```

起動すると、ノードはRPCサーバーとP2Pサーバーをブートストラップし、ピアとのダイアルを開始します。ピアとのハンドシェイク中に、ノードがピアが先行していることに気付いた場合、ノードはキャッチアップするためにすべてのブロックを順次クエリします。その後、バリデータからの新しいブロック提案とブロック署名を待ちます。

## Other commands

ノードを具体的に実行してそれと対話する方法については、[ノードの実行、API、およびCLI](../run-node/01-run-node.md)ガイドを参照してください。
