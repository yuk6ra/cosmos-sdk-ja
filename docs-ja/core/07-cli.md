---
sidebar_position: 1
---

# Command-Line Interface

:::note Synopsis
この文書では、高レベルでコマンドラインインターフェース（CLI）の動作について説明します。[**アプリケーション**](../basics/00-app-anatomy.md)向けのCLIの実装については、別の文書で説明します。Cosmos SDK [**モジュール**](../building-modules/01-intro.md)用のCLIの実装については、[こちら](../building-modules/09-module-interfaces.md#cli)を参照してください。
:::

## Command-Line Interface

### Example Command

CLIの作成方法には決まった方法はありませんが、Cosmos SDKモジュールでは通常、[Cobraライブラリ](https://github.com/spf13/cobra)が使用されます。Cobraを使用してCLIを構築する際には、コマンド、引数、フラグを定義する必要があります。[**コマンド**](#root-command)は、ユーザーが実行したいアクションを理解します。たとえば、トランザクションを作成するための `tx` や、アプリケーションをクエリするための `query` などです。それぞれのコマンドには、特定のトランザクションタイプを指定するために必要なネストされたサブコマンドも含まれます。ユーザーはまた、コインを送信するためのアカウント番号などの**引数**と、コマンドのさまざまな側面を変更するための[**フラグ**](#flags)を提供します。これにはガス料金や送信先ノードなどが含まれます。

以下は、ユーザーがいくつかのトークンを送信するために simapp CLI `simd` とやり取りするために入力する可能性のあるコマンドの例です：

```bash
simd tx bank send $MY_VALIDATOR_ADDRESS $RECIPIENT 1000stake --gas auto --gas-prices <gasPrices>
```

最初の4つの文字列はコマンドを指定します：

* アプリケーション全体のルートコマンド `simd`。
* ユーザーがトランザクションを作成するためのすべてのコマンドを含むサブコマンド `tx`。
* コマンドをルーティングするためのモジュールを示すサブコマンド `bank`（この場合は[`x/bank`](../modules/bank/README.md)モジュール）。
* トランザクションのタイプ `send`。

次の2つの文字列は引数です：送信元の `from_address`、受信者の `to_address`、送信する `amount`。最後に、コマンドの最後のいくつかの文字列は、トランザクションを実行するために使用されるガスの量とユーザーが提供したガス価格に基づいていくら支払うかを示すオプションのフラグです。

このCLIは、このコマンドを処理するために[ノード](../core/03-node.md)とやり取りします。インターフェース自体は、`main.go` ファイルで定義されています。

### Building the CLI

`main.go` ファイルには、すべてのアプリケーションコマンドがサブコマンドとして追加されるルートコマンドを作成する `main()` 関数が必要です。ルートコマンドはさらに次の処理を行います：

* 設定ファイル（Cosmos SDK設定ファイルなど）を読み込むことで、**setting configuration**。
* `--chain-id` などのフラグを含むように、**フラグの追加**。
* アプリケーションコーデックをインジェクションして、**`codec`のインスタンス化**。[`codec`](../core/05-encoding.md)はアプリケーションのデータ構造をエンコードおよびデコードするために使用されます。ストアは `[]byte` のみを永続化できるため、開発者はデータ構造のシリアル化形式を定義するか、デフォルトのProtobufを使用する必要があります。
* [トランザクションコマンド](#transaction-commands)や[クエリコマンド](#query-commands)を含む、すべての可能なユーザーインタラクションのための**サブコマンドの追加**。

`main()` 関数は最終的に実行者を作成し、ルートコマンドを[実行します](https://pkg.go.dev/github.com/spf13/cobra#Command.Execute)。`simapp` アプリケーションからの`main()`関数の例を以下に示します：

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/simapp/simd/main.go#L12-L24
```

ドキュメントの残りの部分では、各ステップごとに実装する必要がある内容と、`simapp` CLIファイルからのコードの一部を詳しく説明します。


## Adding Commands to the CLI

すべてのアプリケーションCLIは、まずルートコマンドを構築し、`rootCmd.AddCommand()`を使用してサブコマンド（さらにネストされたサブコマンドがあることもあります）を集約することで機能を追加します。アプリケーションの固有の機能の大部分は、トランザクションコマンド（`TxCmd`）およびクエリコマンド（`QueryCmd`）にあります。

### Root Command

ルートコマンド（`rootCmd`と呼ばれる）は、ユーザーがコマンドラインに最初に入力することで、どのアプリケーションとやり取りしたいかを示すものです。コマンドを呼び出すために使用される文字列（「Use」フィールド）は、通常、アプリケーションの名前に `-d` が付いたものです。たとえば、`simd` や `gaiad` です。ルートコマンドには通常、アプリケーションの基本的な機能をサポートする以下のコマンドが含まれています。

* Cosmos SDK rpcクライアントツールからの**Status**コマンド。これは、接続された[`Node`](../core/03-node.md)のステータスに関する情報を表示します。ノードのステータスには`NodeInfo`、`SyncInfo`、`ValidatorInfo`が含まれます。
* Cosmos SDKクライアントツールの[**Keys**コマンド](https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/client/keys)。これには、Cosmos SDK暗号ツールのキー関数を使用するためのサブコマンドのコレクションが含まれており、新しいキーを追加してキーリングに保存したり、キーリングに格納されているすべての公開キーの一覧を表示したり、キーを削除したりするための機能が備わっています。例えば、ユーザーは `simd keys add <name>` と入力して新しいキーを追加し、キーリングに暗号化されたコピーを保存することができます。フラグ `--recover` を使用してシードフレーズからプライベートキーを回復したり、フラグ `--multisig` を使用して複数のキーをグループ化してマルチシグキーを作成したりすることもできます。`add`キーコマンドの詳細については、[こちらのコード](https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/client/keys/add.go)を参照してください。キーの認証情報の保存に対する `--keyring-backend` の使用方法の詳細については、[キーリングのドキュメント](../run-node/00-keyring.md)を参照してください。
* Cosmos SDKサーバーパッケージからの**Server**コマンド。これらのコマンドは、ABCI CometBFTアプリケーションを起動するために必要なメカニズムを提供し、アプリケーションを完全にブートストラップするためのCLIフレームワーク（[cobra](https://github.com/spf13/cobra)に基づく）を提供します。このパッケージは、`StartCmd` および `ExportCmd` という2つのコア関数を公開しており、それぞれアプリケーションを起動するコマンドと状態をエクスポートするコマンドを作成します。詳細は[こちら](https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/server)を参照してください。
* [**トランザクション**](#transaction-commands)コマンド。
* [**クエリ**](#query-commands)コマンド。

次に、`simapp` アプリケーションからの`rootCmd`関数の例です。この関数はルートコマンドをインスタンス化し、[*永続*フラグ](#flags)と、すべての実行前に実行される `PreRun` 関数、および必要なすべてのサブコマンドを追加します。

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/simapp/simd/cmd/root_v2.go#L47-L130
```

:::tip
Use the `EnhanceRootCommand()` from the AutoCLI options to automatically add auto-generated commands from the modules to the root command.
Additionnally it adds all manually defined modules commands (`tx` and `query`) as well.
Read more about [AutoCLI](https://docs.cosmos.network/main/building-modules/autocli#getting-started) in its dedicated section.
:::

`rootCmd` has a function called `initAppConfig()` which is useful for setting the application's custom configs.
By default app uses CometBFT app config template from Cosmos SDK, which can be over-written via `initAppConfig()`.
Here's an example code to override default `app.toml` template.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/simapp/simd/cmd/root_v2.go#L144-L199
```

The `initAppConfig()` also allows overriding the default Cosmos SDK's [server config](https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/server/config/config.go#L235). One example is the `min-gas-prices` config, which defines the minimum gas prices a validator is willing to accept for processing a transaction. By default, the Cosmos SDK sets this parameter to `""` (empty string), which forces all validators to tweak their own `app.toml` and set a non-empty value, or else the node will halt on startup. This might not be the best UX for validators, so the chain developer can set a default `app.toml` value for validators inside this `initAppConfig()` function.

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/simapp/simd/cmd/root_v2.go#L164-L180
```

The root-level `status` and `keys` subcommands are common across most applications and do not interact with application state. The bulk of an application's functionality - what users can actually *do* with it - is enabled by its `tx` and `query` commands.

### Transaction Commands

[Transactions](./01-transactions.md) are objects wrapping [`Msg`s](../building-modules/02-messages-and-queries.md#messages) that trigger state changes. To enable the creation of transactions using the CLI interface, a function `txCommand` is generally added to the `rootCmd`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/simapp/simd/cmd/root_v2.go#L222-L229
```

This `txCommand` function adds all the transaction available to end-users for the application. This typically includes:

* **Sign command** from the [`auth`](../modules/auth/README.md) module that signs messages in a transaction. To enable multisig, add the `auth` module's `MultiSign` command. Since every transaction requires some sort of signature in order to be valid, the signing command is necessary for every application.
* **Broadcast command** from the Cosmos SDK client tools, to broadcast transactions.
* **All [module transaction commands](../building-modules/09-module-interfaces.md#transaction-commands)** the application is dependent on, retrieved by using the [basic module manager's](../building-modules/01-module-manager.md#basic-manager) `AddTxCommands()` function, or enhanced by [AutoCLI](https://docs.cosmos.network/main/building-modules/autocli#getting-started).

Here is an example of a `txCommand` aggregating these subcommands from the `simapp` application:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/simapp/simd/cmd/root_v2.go#L270-L292
```

:::tip
When using AutoCLI to generate module transaction commands, `EnhanceRootCommand()` automatically adds the module `tx` command to the root command.
Read more about [AutoCLI](https://docs.cosmos.network/main/building-modules/autocli#getting-started) in its dedicated section.
:::

### Query Commands

[**Queries**](../building-modules/02-messages-and-queries.md#queries) are objects that allow users to retrieve information about the application's state. To enable the creation of queries using the CLI interface, a function `queryCommand` is generally added to the `rootCmd`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/simapp/simd/cmd/root_v2.go#L222-L229
```

This `queryCommand` function adds all the queries available to end-users for the application. This typically includes:

* **QueryTx** and/or other transaction query commands from the `auth` module which allow the user to search for a transaction by inputting its hash, a list of tags, or a block height. These queries allow users to see if transactions have been included in a block.
* **Account command** from the `auth` module, which displays the state (e.g. account balance) of an account given an address.
* **Validator command** from the Cosmos SDK rpc client tools, which displays the validator set of a given height.
* **Block command** from the Cosmos SDK RPC client tools, which displays the block data for a given height.
* **All [module query commands](../building-modules/09-module-interfaces.md#query-commands)** the application is dependent on, retrieved by using the [basic module manager's](../building-modules/01-module-manager.md#basic-manager) `AddQueryCommands()` function, or enhanced by [AutoCLI](https://docs.cosmos.network/main/building-modules/autocli#getting-started).

Here is an example of a `queryCommand` aggregating subcommands from the `simapp` application:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/simapp/simd/cmd/root_v2.go#L249-L268
```

:::tip
When using AutoCLI to generate module query commands, `EnhanceRootCommand()` automatically adds the module `query` command to the root command.
Read more about [AutoCLI](https://docs.cosmos.network/main/building-modules/autocli#getting-started) in its dedicated section.
:::

## Flags

Flags are used to modify commands; developers can include them in a `flags.go` file with their CLI. Users can explicitly include them in commands or pre-configure them by inside their [`app.toml`](../run-node/01-run-node.md#configuring-the-node-using-apptoml-and-configtoml). Commonly pre-configured flags include the `--node` to connect to and `--chain-id` of the blockchain the user wishes to interact with.

A *persistent* flag (as opposed to a *local* flag) added to a command transcends all of its children: subcommands will inherit the configured values for these flags. Additionally, all flags have default values when they are added to commands; some toggle an option off but others are empty values that the user needs to override to create valid commands. A flag can be explicitly marked as *required* so that an error is automatically thrown if the user does not provide a value, but it is also acceptable to handle unexpected missing flags differently.

Flags are added to commands directly (generally in the [module's CLI file](../building-modules/09-module-interfaces.md#flags) where module commands are defined) and no flag except for the `rootCmd` persistent flags has to be added at application level. It is common to add a *persistent* flag for `--chain-id`, the unique identifier of the blockchain the application pertains to, to the root command. Adding this flag can be done in the `main()` function. Adding this flag makes sense as the chain ID should not be changing across commands in this application CLI.

## Environment variables

Each flag is bound to it's respecteve named environment variable. Then name of the environment variable consist of two parts - capital case `basename` followed by flag name of the flag. `-` must be substituted with `_`. For example flag `--home` for application with basename `GAIA` is bound to `GAIA_HOME`. It allows reducing the amount of flags typed for routine operations. For example instead of:

```shell
gaia --home=./ --node=<node address> --chain-id="testchain-1" --keyring-backend=test tx ... --from=<key name>
```

this will be more convenient:

```shell
# define env variables in .env, .envrc etc
GAIA_HOME=<path to home>
GAIA_NODE=<node address>
GAIA_CHAIN_ID="testchain-1"
GAIA_KEYRING_BACKEND="test"

# and later just use
gaia tx ... --from=<key name>
```

## Configurations

It is vital that the root command of an application uses `PersistentPreRun()` cobra command property for executing the command, so all child commands have access to the server and client contexts. These contexts are set as their default values initially and maybe modified, scoped to the command, in their respective `PersistentPreRun()` functions. Note that the `client.Context` is typically pre-populated with "default" values that may be useful for all commands to inherit and override if necessary.

Here is an example of an `PersistentPreRun()` function from `simapp`:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/simapp/simd/cmd/root_v2.go#L81-L120
```

The `SetCmdClientContextHandler` call reads persistent flags via `ReadPersistentCommandFlags` which creates a `client.Context` and sets that on the root command's `Context`.

The `InterceptConfigsPreRunHandler` call creates a viper literal, default `server.Context`, and a logger and sets that on the root command's `Context`. The `server.Context` will be modified and saved to disk. The internal `interceptConfigs` call reads or creates a CometBFT configuration based on the home path provided. In addition, `interceptConfigs` also reads and loads the application configuration, `app.toml`, and binds that to the `server.Context` viper literal. This is vital so the application can get access to not only the CLI flags, but also to the application configuration values provided by this file.

:::tip
When willing to configure which logger is used, do not to use `InterceptConfigsPreRunHandler`, which sets the default SDK logger, but instead use `InterceptConfigsAndCreateContext` and set the server context and the logger manually:

```diff
-return server.InterceptConfigsPreRunHandler(cmd, customAppTemplate, customAppConfig, customCMTConfig)

+serverCtx, err := server.InterceptConfigsAndCreateContext(cmd, customAppTemplate, customAppConfig, customCMTConfig)
+if err != nil {
+	return err
+}

+// overwrite default server logger
+logger, err := server.CreateSDKLogger(serverCtx, cmd.OutOrStdout())
+if err != nil {
+	return err
+}
+serverCtx.Logger = logger.With(log.ModuleKey, "server")

+// set server context
+return server.SetCmdServerContext(cmd, serverCtx)
```

:::
