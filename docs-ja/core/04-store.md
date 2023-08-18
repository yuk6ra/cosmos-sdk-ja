---
sidebar_position: 1
---

# Store

:::note Synopsis
ストアとは、アプリケーションの状態を保持するデータ構造である。
:::

:::note Pre-requisite Readings

* [Anatomy of a Cosmos SDK application](../basics/00-app-anatomy.md)

:::

## Introduction to Cosmos SDK Stores

Cosmos SDKには、アプリケーションの状態を永続化するための多くのストアが用意されています。デフォルトでは、Cosmos SDKアプリケーションのメインストアは「multistore」です。つまり、ストアの集合体です。開発者は、アプリケーションのニーズに応じて、multistoreに任意の数のキーと値のストアを追加できます。multistoreはCosmos SDKのモジュラリティをサポートするために存在し、各モジュールが状態のサブセットを宣言して管理できるようにします。multistoreのキーとしてアクセスできるのは、通常はストアを宣言したモジュールの[`keeper`](../building-modules/06-keeper.md)に保持されている特定のキャパビリティ「key」だけです。

```text
+-----------------------------------------------------+
|                                                     |
|    +--------------------------------------------+   |
|    |                                            |   |
|    |  KVStore 1 - Manage by keeper of Module 1  |
|    |                                            |   |
|    +--------------------------------------------+   |
|                                                     |
|    +--------------------------------------------+   |
|    |                                            |   |
|    |  KVStore 2 - Manage by keeper of Module 2  |   |
|    |                                            |   |
|    +--------------------------------------------+   |
|                                                     |
|    +--------------------------------------------+   |
|    |                                            |   |
|    |  KVStore 3 - Manage by keeper of Module 2  |   |
|    |                                            |   |
|    +--------------------------------------------+   |
|                                                     |
|    +--------------------------------------------+   |
|    |                                            |   |
|    |  KVStore 4 - Manage by keeper of Module 3  |   |
|    |                                            |   |
|    +--------------------------------------------+   |
|                                                     |
|    +--------------------------------------------+   |
|    |                                            |   |
|    |  KVStore 5 - Manage by keeper of Module 4  |   |
|    |                                            |   |
|    +--------------------------------------------+   |
|                                                     |
|                    Main Multistore                  |
|                                                     |
+-----------------------------------------------------+

                   Application's State
```

### Store Interface

Cosmos SDKの`store`は、`CacheWrapper`を保持し、`GetStoreType()`メソッドを持つオブジェクトです。

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/store/types/store.go#L15-L18
```

`GetStoreType`は、ストアのタイプを返すシンプルなメソッドです。一方、`CacheWrapper`は、ストアの読み取りキャッシングと`Write`メソッドを介した書き込みブランチングを実装するシンプルなインターフェースです。

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/store/types/store.go#L287-L320
```

ブランチングとキャッシュは、Cosmos SDK内で広く使用され、すべてのストアタイプで実装される必要があります。ストレージブランチは、ストアの孤立した一時的なブランチを作成し、メインの基本的なストアに影響を与えずに渡したり更新したりするために使用されます。これは一時的な状態遷移をトリガーし、後でエラーが発生した場合に元に戻すことができるようにするために使用されます。詳細は[こちら](./02-context.md#Store-branching)をご覧ください。

### Commit Store

コミットストアは、基礎となるツリーやデータベースへの変更をコミットする能力を持つストアです。Cosmos SDKは、基本的なストアインターフェースに`Committer`を拡張することで、シンプルなストアとコミットストアを区別しています。

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/store/types/store.go#L32-L37
```

`Committer`は、変更をディスクに永続化するためのメソッドを定義するインターフェースです。

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/store/types/store.go#L20-L30
```

`CommitID`は、状態ツリーの決定論的なコミットです。そのハッシュは基礎となるコンセンサスエンジンに返され、ブロックヘッダーに格納されます。コミットストアインターフェースはさまざまな目的で存在し、その1つはすべてのオブジェクトがストアをコミットできないようにすることです。Cosmos SDKの[オブジェクト権限モデル](./10-ocap.md)の一部として、`baseapp`だけがストアをコミットする権限を持つべきです。たとえば、モジュールが通常ストアにアクセスするために使用する`ctx.KVStore()`メソッドは、`KVStore`ではなく`CommitKVStore`を返す理由です。

Cosmos SDKにはさまざまな種類のストアが付属しており、最も使用されるのは[`CommitMultiStore`](#multistore)、[`KV Store`](#kvstore)、[`GasKv Store`](#gaskv-store)です。[その他のストアの種類](#other-stores)には、`Transient`ストアと`TraceKV`ストアが含まれます。

## Multistore

### Multistore Interface

各Cosmos SDKアプリケーションは、その状態を永続化するためにルートにマルチストアを保持しています。マルチストアは、`KVStores`のストアであり、`Multistore`インターフェースに従います：

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/store/types/store.go#L123-L155
```

トレースが有効にされている場合、マルチストアのブランチはまず、すべての基本となる`KVStore`を[`TraceKv.Store`](#tracekv-store)でラップします。

### CommitMultiStore

Cosmos SDKで使用される主要な`Multistore`のタイプは、`CommitMultiStore`で、これは`Multistore`インターフェースの拡張です：

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/store/types/store.go#L164-L227
```

具体的な実装に関しては、[`rootMulti.Store`]が`CommitMultiStore`インターフェースの主要な実装です。

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/store/rootmulti/store.go#L53-L77
```

`rootMulti.Store`は、複数の`KVStore`をマウントできる`db`の上に構築されたベースレイヤーマルチストアであり、[`baseapp`](./00-baseapp.md)で使用されるデフォルトのマルチストアストアです。

### CacheMultiStore

`rootMulti.Store`がブランチする必要がある場合、[`cachemulti.Store`](https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/store/cachemulti/store.go)が使用されます。

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/store/cachemulti/store.go#L19-L33
```

`cachemulti.Store`は、コンストラクタですべてのサブストア（各サブストアの仮想ストアを作成）をブランチし、それらを`Store.stores`に保持します。さらに、すべての読み取りクエリをキャッシュします。`Store.GetKVStore()`は`Store.stores`からストアを返し、`Store.Write()`はすべてのサブストアに対して再帰的に`CacheWrap.Write()`を呼び出します。


## Base-layer KVStores

### `KVStore` and `CommitKVStore` Interfaces

`KVStore`は、データの格納と取得に使用されるシンプルなキー・バリュー・ストアです。`CommitKVStore`は、`Committer`も実装した`KVStore`です。デフォルトでは、`baseapp`のメインの`CommitMultiStore`にマウントされるストアは、通常`CommitKVStore`です。`KVStore`インターフェースは、主にモジュールがコミッターにアクセスできないよう制限するために使用されます。

個々の`KVStore`は、モジュールがグローバルな状態のサブセットを管理するために使用されます。`KVStores`は、特定のキーを保持するオブジェクトからアクセスできます。この`key`は、ストアを定義するモジュールの[`keeper`](../building-modules/06-keeper.md)にのみ公開されるべきです。

`CommitKVStore`は、それぞれの`key`を代理として宣言され、アプリケーションの[マルチストア](#multistore)にマウントされます。同じファイルで、`key`はモジュールの`keeper`にも渡され、そのストアの管理を担当します。

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/store/types/store.go#L229-L266
```

従来の`Get`と`Set`メソッド以外に、`KVStore`は`BasicKVStore`インターフェースを介して実装する必要があります。さらに、`KVStore`は`Iterator(start, end)`メソッドを提供する必要があります。このメソッドは`Iterator`オブジェクトを返し、通常は共通の接頭辞を持つキーの範囲を繰り返し処理するために使用されます。以下は、銀行のモジュールの`keeper`からの例で、すべてのアカウント残高を繰り返し処理するために使用されます：


```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/bank/keeper/view.go#L125-L140
```

### `IAVL` Store

`baseapp`で使用される`KVStore`と`CommitKVStore`のデフォルト実装は、`iavl.Store`です。

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/store/iavl/store.go#L35-L40
```

`iavl`ストアは、[IAVL Tree](https://github.com/cosmos/iavl)に基づいており、自己バランシングバイナリツリーで以下を保証します：

* `Get`および`Set`操作の計算量はO(log n)で、ここでnはツリー内の要素数です。
* 範囲内の要素を効率的に取得する反復処理を提供します。
* 各ツリーバージョンは不変で、コミット後でも取得できます（剪定設定に依存）。

IAVL Treeのドキュメントは[こちら](https://github.com/cosmos/iavl/blob/master/docs/overview.md)にあります。

### `DbAdapter` Store

`dbadapter.Store`は、`dbm.DB`のアダプタで、`KVStore`インターフェースを満たすようにします。

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/store/dbadapter/store.go#L13-L16
```

`dbadapter.Store`は`dbm.DB`を埋め込んでおり、`KVStore`インターフェースのほとんどの関数が実装されています。その他の関数（主にその他の機能）は手動で実装されています。このストアは主に[一時ストア](#transient-store)内で使用されます。

### `Transient` Store 

`Transient.Store`は、ブロックの最後に自動的に破棄される基本的な`KVStore`です。

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/store/transient/store.go#L16-L19
```

`Transient.Store`は、`dbm.NewMemDB()`を持つ`dbadapter.Store`です。すべての`KVStore`メソッドが再利用されます。`Store.Commit()`が呼び出されると、新しい`dbadapter.Store`が割り当てられ、以前の参照が破棄され、ガベージコレクションされます。

このタイプのストアは、ブロックごとに関連する情報を保持するために有用です。例えば、パラメータの変更を格納する場合（ブロック内でパラメータが変更された場合に`true`に設定されるブール値など）です。

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/params/types/subspace.go#L21-L31
```

一時ストアは通常、[`context`](./02-context.md)を介して`TransientStore()`メソッドを使用してアクセスされます：

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/types/context.go#L340-L343
```

## KVStore Wrappers

### CacheKVStore

`cachekv.Store`は、基礎となる`KVStore`上でバッファ付きの書き込み/キャッシュされた読み取り機能を提供するラッパー`KVStore`です。

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/store/cachekv/store.go#L26-L36
```

これは、IAVLストアを分岐して孤立したストアを作成する必要がある場合に使用されるタイプです（通常、後で元に戻される可能性のあるステートを変更する必要がある場合など）。

#### `Get`

`Store.Get()`は、まず`Store.cache`がキーに関連する値を持っているかどうかをチェックします。値が存在する場合、関数はその値を返します。存在しない場合、関数は`Store.parent.Get()`を呼び出し、結果を`Store.cache`にキャッシュし、それを返します。

#### `Set`

`Store.Set()`はキーと値のペアを`Store.cache`に設定します。`cValue`には、キャッシュされた値が基になる値と異なるかどうかを示すdirtyブールフィールドがあります。`Store.Set()`が新しいペアをキャッシュすると、`cValue.dirty`が`true`に設定されるため、`Store.Write()`が呼び出されると基になるストアに書き込むことができます。

#### `Iterator`

`Store.Iterator()`はキャッシュされたアイテムと元のアイテムの両方をトラバースする必要があります。`Store.iterator()`では、それぞれのために2つのイテレーターが生成され、マージされます。`memIterator`は基本的に`KVPairs`のスライスであり、キャッシュされたアイテムに使用されます。`mergeIterator`は2つのイテレーターの組み合わせであり、両方のイテレーターで順序付けされてトラバースが行われます。

### `GasKv` Store

Cosmos SDKアプリケーションでは、リソースの使用状況を追跡しスパムを防ぐために[`gas`](../basics/04-gas-fees.md)を使用します。[`GasKv.Store`](https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/store/gaskv/store.go)は、自動的にガスを消費する`KVStore`ラッパーであり、ストアへの読み取りや書き込みごとにガス消費が行われるようになっています。これはCosmos SDKアプリケーションでストレージの使用状況を追跡するための選択肢です。

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/store/gaskv/store.go#L11-L17
```

親の`KVStore`のメソッドが呼び出される際、`GasKv.Store`は`Store.gasConfig`に応じて適切なガス量を自動的に消費します：

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/store/types/gas.go#L219-L228
```

デフォルトでは、すべての`KVStore`は取得時に`GasKv.Store`でラップされます。これは[`context`](./02-context.md)の`KVStore()`メソッドで行われます：

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/types/context.go#L335-L338
```

この場合、`context`に設定されたガス構成が使用されます。ガス構成は`context`の`WithKVGasConfig`メソッドを使用して設定できます。それ以外の場合は、次のデフォルト値が使用されます：

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/store/types/gas.go#L230-L241
```

### `TraceKv` Store

`tracekv.Store`は、基になる`KVStore`に対する操作トレース機能を提供するラッパー`KVStore`です。親の`MultiStore`でトレースが有効になっている場合、Cosmos SDKによってすべての`KVStore`に自動的に適用されます。

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/store/tracekv/store.go#L20-L43
```

各`KVStore`メソッドが呼び出されると、`tracekv.Store`は`traceOperation`を`Store.writer`に自動的にログします。`traceOperation.Metadata`はnilでない場合、`Store.context`で埋められます。`TraceContext`は`map[string]interface{}`です。

### `Prefix` Store

`prefix.Store`は、基になる`KVStore`に対する自動キー接頭辞機能を提供するラッパー`KVStore`です。

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/store/prefix/store.go#L15-L21
```

`Store.{Get, Set}()`が呼び出されると、キーは`Store.prefix`で接頭辞が付けられた状態で親に呼び出しが転送されます。

`Store.Iterator()`が呼び出される場合、単に`Store.prefix`を接頭辞として追加するだけでは意図した通りに動作しないため、そのような処理は行われません。この場合、いくつかの要素がプレフィックスで始まらない場合でもトラバースされることがあります。

### `ListenKv` Store

`listenkv.Store` is a wrapper `KVStore` which provides state listening capabilities over the underlying `KVStore`.
It is applied automatically by the Cosmos SDK on any `KVStore` whose `StoreKey` is specified during state streaming configuration.
Additional information about state streaming configuration can be found in the [store/streaming/README.md](https://github.com/cosmos/cosmos-sdk/tree/v0.50.0-alpha.0/store/streaming).

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/store/listenkv/store.go#L11-L18
```

`KVStore.Set`または`KVStore.Delete`メソッドが呼び出されると、`listenkv.Store`は自動的に操作を`Store.listeners`のセットに書き込みます。

## `BasicKVStore` interface

基本的なCRUD機能（`Get`、`Set`、`Has`、および`Delete`メソッド）のみを提供するインターフェースで、イテレーションやキャッシュは含まれていません。これは、より大きなストアのコンポーネントを一部公開するために使用されます。

