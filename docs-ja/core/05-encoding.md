---
sidebar_position: 1
---

# Encoding

:::note Synopsis
Cosmos SDKにおけるエンコーディングは、以前は主に`go-amino`コーデックで処理されていましたが、Cosmos SDKはステートとクライアント側のエンコーディングの両方に`gogoprotobuf`の使用に移行しています。
:::

:::note Pre-requisite Readings

* [Anatomy of a Cosmos SDK application](../basics/00-app-anatomy.md)

:::

## Encoding

Cosmos SDKは、2つのバイナリワイヤーエンコーディングプロトコル、オブジェクトエンコーディング仕様である[Amino](https://github.com/tendermint/go-amino/)と、Proto3のサブセットでインターフェースサポートの拡張機能を持つ[Protocol Buffers](https://developers.google.com/protocol-buffers)を使用しています。Proto3に関する詳細は、[Proto3仕様](https://developers.google.com/protocol-buffers/docs/proto3)を参照してください。

AminoはProto3と主に互換性がありますが（Proto2とは互換性がありません）、Aminoは重要なクロス言語/クライアントサポートを持たず、性能上の欠点があり、リフレクションベースであるため、Protocol Buffers、特に[gogoprotobuf](https://github.com/cosmos/gogoproto/)がAminoの代わりに使用されています。ただし、Protocol BuffersをAminoの代わりに使用するプロセスはまだ進行中です。

Cosmos SDKの型のバイナリワイヤーエンコーディングは、主に2つの主要なカテゴリ、クライアントエンコーディングとストアエンコーディングに分けることができます。クライアントエンコーディングは主にトランザクション処理と署名に関連し、一方、ストアエンコーディングはステートマシンの遷移で使用される型や最終的にMerkleツリーに格納される内容に関連します。

ストアエンコーディングでは、プロトコルバッファ定義が任意の型に存在し、通常Aminoベースの「中間」型を持っています。具体的には、プロトコルバッファベースの型定義はシリアル化と永続性に使用され、一方、Aminoベースの型はビジネスロジックに使用され、ステートマシン内で相互変換されることがあります。ただし、将来的にAminoベースの型は徐々に段階的に廃止される可能性があるため、開発者は可能な限りプロトコルバッファメッセージ定義を使用するよう注意する必要があります。

`codec`パッケージには、`BinaryCodec`と`JSONCodec`という2つのコアインターフェースが存在し、前者は現在のAminoインターフェースをカプセル化しており、通常の`interface{}`型ではなく、後者を実装する型で動作します。

さらに、`Codec`の2つの実装が存在します。1つ目は`AminoCodec`で、バイナリとJSONのシリアル化がAminoを介して処理されます。2つ目は`ProtoCodec`で、バイナリとJSONのシリアル化がProtobufを介して処理されます。

これにより、モジュールはAminoまたはProtobufエンコーディングを使用できますが、型は`ProtoMarshaler`を実装する必要があります。モジュールが自分の型に対してこのインターフェースを実装するのを避けたい場合は、直接Aminoコーデックを使用できます。

### Amino

各モジュールは、型やインターフェースをシリアライズするためにAminoコーデックを使用します。このコーデックには通常、そのモジュールのドメイン内でのみ登録された型やインターフェース（例：メッセージ）が含まれますが、`x/gov`のような例外もあります。各モジュールは、ユーザーがコーデックを提供してすべての型を登録できる`RegisterLegacyAminoCodec`

モジュールにプロトコルバッファベースの型定義がない場合（以下を参照）、Aminoを使用して生のワイヤーバイトを具体的な型やインターフェースにエンコードおよびデコードします。

```go
bz := keeper.cdc.MustMarshal(typeOrInterface)
keeper.cdc.MustUnmarshal(bz, &typeOrInterface)
```

注意：上記の機能には長さプレフィックス付きのバリアントもあり、これはデータをストリーム化するか、グループ化する必要がある場合に通常使用されます（例：`ResponseDeliverTx.Data`）。

#### Authz authorizations and Gov/Group proposals

authzの`MsgExec`および`MsgGrant`メッセージタイプ、およびgovとgroupの`MsgSubmitProposal`は異なるメッセージインスタンスを含む可能性があるため、開発者はこれらのメッセージの`init`メソッド内で次のコードを追加する必要があります。モジュールの`codec.go`ファイル：
```go
import (
  authzcodec "github.com/cosmos/cosmos-sdk/x/authz/codec"
  govcodec "github.com/cosmos/cosmos-sdk/x/gov/codec"
  groupcodec "github.com/cosmos/cosmos-sdk/x/group/codec"
)

init() {
    // Register all Amino interfaces and concrete types on the authz and gov Amino codec so that this can later be
    // used to properly serialize MsgGrant, MsgExec and MsgSubmitProposal instances
    RegisterLegacyAminoCodec(authzcodec.Amino)
    RegisterLegacyAminoCodec(govcodec.Amino)
    RegisterLegacyAminoCodec(groupcodec.Amino)
}
```
これにより、`x/authz`モジュールは、Aminoを使用して`MsgExec`インスタンスを適切にシリアライズおよびデシリアライズできるようになります。このようなメッセージをLedgerで署名する際に必要です。

### Gogoproto

各モジュールは、それぞれの型に対してProtobufエンコーディングを活用することをお勧めします。Cosmos SDKでは、公式の[Google protobuf実装](https://github.com/protocolbuffers/protobuf)と比較して、速度と開発体験の向上を提供するProtobuf仕様の[Gogoproto](https://github.com/cosmos/gogoproto)特有の実装を使用しています。

### Guidelines for protobuf message definitions

[公式のProtocol Bufferガイドラインに従う](https://developers.google.com/protocol-buffers/docs/proto3#simple)ことに加えて、インターフェースを扱う場合に.protoファイルで次の注釈を使用することをお勧めします：

- `cosmos_proto.accepts_interface`を使用して、インターフェースを受け入れる`Any`フィールドを注釈する
    - `protoName`として`InterfaceRegistry.RegisterInterface`に完全修飾名を指定する
    - 例：`(cosmos_proto.accepts_interface) = "cosmos.gov.v1beta1.Content"`（単に`Content`ではなく）
- インターフェースの実装を`cosmos_proto.implements_interface`で注釈する
    - `protoName`として`InterfaceRegistry.RegisterInterface`に完全修飾名を指定する
    - 例：`(cosmos_proto.implements_interface) = "cosmos.authz.v1beta1.Authorization"`（単に`Authorization`ではなく）

その後、コード生成ツールは、`accepts_interface`および`implements_interface`の注釈と、特定の`Any`フィールドにProtobufメッセージをパックすることが許可されているかどうかを判断します。

### Transaction Encoding

Protobufのもう一つの重要な用途は、[トランザクション](./01-transactions.md)のエンコーディングとデコーディングです。トランザクションは、アプリケーションまたはCosmos SDKによって定義されますが、その後、基盤となるコンセンサスエンジンに渡され、他のピアに中継されます。基盤となるコンセンサスエンジンはアプリケーションに対して無関心であるため、コンセンサスエンジンは生のバイト形式のトランザクションのみを受け入れます。

* `TxEncoder`オブジェクトはエンコーディングを実行します。
* `TxDecoder`オブジェクトはデコーディングを実行します。

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/types/tx_msg.go#L91-L95
```

これらのオブジェクトの標準的な実装は、[`auth/tx`モジュール](../modules/auth/tx/README.md)にあります：

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/auth/tx/decoder.go
```

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/auth/tx/encoder.go
```

トランザクションのエンコード方法の詳細については、[ADR-020](../architecture/adr-020-protobuf-transaction-encoding.md)をご覧ください。

### Interface Encoding and Usage of `Any`

Protobuf DSLは強く型付けされており、変数型のフィールドを挿入することが難しいことがあります。`Profile`というProtobufメッセージを作成したいと考えてみましょう。これは、[アカウント](../basics/03-accounts.md)の上にラッパーとして機能します：

```protobuf
message Profile {
  // account is the account associated to a profile.
  cosmos.auth.v1beta1.BaseAccount account = 1;
  // bio is a short description of the account.
  string bio = 4;
}
```

この`Profile`の例では、`account`を`BaseAccount`としてハードコードしています。しかし、[ベスティングに関連するユーザーアカウント](../modules/auth/1-vesting.md)には他にも`BaseVestingAccount`や`ContinuousVestingAccount`など、いくつかの異なるタイプのアカウントがあります。これらすべてのアカウントは異なりますが、すべて`AccountI`インターフェースを実装しています。どのようにして、これらのタイプのアカウントを受け入れる`AccountI`インターフェースを持つ`account`フィールドを持つ`Profile`を作成できるでしょうか？

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/types/account.go#L15-L32
```

[ADR-019](../architecture/adr-019-protobuf-state-encoding.md)では、protobuf内のインターフェースをエンコードするために[`Any`](https://github.com/protocolbuffers/protobuf/blob/master/src/google/protobuf/any.proto)を使用することが決定されました。`Any`は、任意のシリアライズされたメッセージと、そのメッセージのタイプをグローバルに一意の識別子として機能し、そのタイプに解決するURLを含んでいます。この戦略により、任意のGoの型をprotobufメッセージ内に詰め込むことができます。新しい`Profile`は次のようになります：

```protobuf
message Profile {
  // account is the account associated to a profile.
  google.protobuf.Any account = 1 [
    (cosmos_proto.accepts_interface) = "cosmos.auth.v1beta1.AccountI"; // Asserts that this field only accepts Go types implementing `AccountI`. It is purely informational for now.
  ];
  // bio is a short description of the account.
  string bio = 4;
}
```

プロファイル内にアカウントを追加するには、まず`codectypes.NewAnyWithValue`を使用して、アカウントを`Any`内に「詰め込む」必要があります：

```go
var myAccount AccountI
myAccount = ... // Can be a BaseAccount, a ContinuousVestingAccount or any struct implementing `AccountI`

// Pack the account into an Any
accAny, err := codectypes.NewAnyWithValue(myAccount)
if err != nil {
  return nil, err
}

// Create a new Profile with the any.
profile := Profile {
  Account: accAny,
  Bio: "some bio",
}

// We can then marshal the profile as usual.
bz, err := cdc.Marshal(profile)
jsonBz, err := cdc.MarshalJSON(profile)
```

要約すると、インターフェースをエンコードするには、1/ インターフェースを`Any`内に詰め込み、2/ `Any`をマーシャルする必要があります。便宜のため、Cosmos SDKはこれらの2つのステップをまとめる`MarshalInterface`メソッドを提供しています。[x/authモジュールの実際の例](https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/auth/keeper/keeper.go#L240-L243)をご覧ください。

`Any`内から具体的なGoの型を取得する逆の操作（「アンパック」）は、`Any`上の`GetCachedValue()`を使用して行われます。

```go
profileBz := ... // The proto-encoded bytes of a Profile, e.g. retrieved through gRPC.
var myProfile Profile
// Unmarshal the bytes into the myProfile struct.
err := cdc.Unmarshal(profilebz, &myProfile)

// Let's see the types of the Account field.
fmt.Printf("%T\n", myProfile.Account)                  // Prints "Any"
fmt.Printf("%T\n", myProfile.Account.GetCachedValue()) // Prints "BaseAccount", "ContinuousVestingAccount" or whatever was initially packed in the Any.

// Get the address of the accountt.
accAddr := myProfile.Account.GetCachedValue().(AccountI).GetAddress()
```

`GetCachedValue()`が機能するためには、`Profile`（および`Profile`を埋め込む他の構造体）が`UnpackInterfaces`メソッドを実装している必要があることに注意してください：

```go
func (p *Profile) UnpackInterfaces(unpacker codectypes.AnyUnpacker) error {
  if p.Account != nil {
    var account AccountI
    return unpacker.UnpackAny(p.Account, &account)
  }

  return nil
}
```

`UnpackInterfaces`は、このメソッドを実装するすべての構造体に再帰的に呼び出され、すべての`Any`の`GetCachedValue()`が正しく設定されるようにします。

インターフェースのエンコードに関する詳細情報、特に`UnpackInterfaces`および`Any`の`type_url`が`InterfaceRegistry`を使用して解決される方法については、[ADR-019](../architecture/adr-019-protobuf-state-encoding.md)を参照してください。

#### `Any` Encoding in the Cosmos SDK

上記の例の`Profile`は、教育目的で使用される架空の例です。Cosmos SDKでは、`Any`エンコーディングをいくつかの場所で使用しています（網羅的なリストではありません）：

- 異なる種類の公開鍵をエンコードするための`cryptotypes.PubKey`インターフェース
- トランザクション内の異なる`Msg`をエンコードするための`sdk.Msg`インターフェース
- x/authのクエリ応答内の異なるアカウントの種類（上記の例と類似）をエンコードするための`AccountI`インターフェース
- x/evidenceモジュール内の異なる種類のエビデンスをエンコードするための`Evidencei`インターフェース
- x/authz認可の異なる種類をエンコードするための`AuthorizationI`インターフェース
- バリデータに関する情報を含む[`Validator`](https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/staking/types/staking.pb.go#L340-L377)構造体

x/staking内のValidator構造体内で公開鍵を`Any`としてエンコードする実際の例は、以下の通りです：

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/x/staking/types/validator.go#L41-L64
```

#### `Any`'s TypeURL

protobufメッセージを`Any`内に詰め込む際、メッセージのタイプはそのタイプURLによって一意に定義されます。タイプURLは、メッセージの完全修飾名を`/`（スラッシュ）文字で前置したものです。gogoprotoなどの一部の`Any`の実装では、通常、[解決可能なプレフィックスがあります（例：`type.googleapis.com`）](https://github.com/gogo/protobuf/blob/b03c65ea87cdc3521ede29f62fe3ce239267c1bc/protobuf/google/protobuf/any.proto#L87-L91)。ただし、Cosmos SDKでは、タイプURLを短くするためにそのようなプレフィックスを含めないことを決定しました。Cosmos SDKの`Any`の実装は`github.com/cosmos/cosmos-sdk/codec/types`にあります。

Cosmos SDKは、gogoprotoから公式の`google.golang.org/protobuf`（Protobuf API v2としても知られる）に切り替えています。そのデフォルトの`Any`の実装には[`type.googleapis.com`](https://github.com/protocolbuffers/protobuf-go/blob/v1.28.1/types/known/anypb/any.pb.go#L266)プレフィックスも含まれています。SDKとの互換性を維持するためには、「google.golang.org/protobuf/types/known/anypb」からの以下のメソッドは使用しないでください：

* `anypb.New`
* `anypb.MarshalFrom`
* `anypb.Any#MarshalFrom`

代わりに、Cosmos SDKは「`github.com/cosmos/cosmos-proto/anyutil`」にヘルパー関数を提供しており、公式の「anypb.Any」をプレフィックスを挿入せずに作成することができます：

* `anyutil.New`
* `anyutil.MarshalFrom`

例えば、`sdk.Msg`の「internalMsg」を詰め込む場合、次のように使用します：

```diff
import (
- 	"google.golang.org/protobuf/types/known/anypb"
+	"github.com/cosmos/cosmos-proto/anyutil"
)

- anyMsg, err := anypb.New(internalMsg.Message().Interface())
+ anyMsg, err := anyutil.New(internalMsg.Message().Interface())

- fmt.Println(anyMsg.TypeURL) // type.googleapis.com/cosmos.bank.v1beta1.MsgSend
+ fmt.Println(anyMsg.TypeURL) // /cosmos.bank.v1beta1.MsgSend
```

## FAQ

### How to create modules using protobuf encoding

#### Defining module types

Protobuf types can be defined to encode:

* state
* [`Msg`s](../building-modules/02-messages-and-queries.md#messages)
* [Query services](../building-modules/04-query-services.md)
* [genesis](../building-modules/08-genesis.md)

#### Naming and conventions

開発者には、業界のガイドラインに従うことをお勧めします: [Protocol Buffersスタイルガイド](https://developers.google.com/protocol-buffers/docs/style)および[Buf](https://buf.build/docs/style-guide)。詳細については、[ADR 023](../architecture/adr-023-protobuf-naming.md)をご覧ください。

### How to update modules to protobuf encoding

もしモジュールにインターフェース（例：`Account`や`Content`）が含まれていない場合、既存の型を単にマイグレーションすることができます。これらの型は、具体的なAminoコーデックを使用してエンコードおよび永続化されているものです（詳細についてはガイドライン1.を参照）。その際、コーデックとして`ProtoCodec`を使用し、さらなるカスタマイズは必要ありません。

しかし、もしモジュールの型がインターフェースを組み合わせている場合、その型は`sdk.Any`（`/types`パッケージからのもの）でラップする必要があります。そのために、モジュールレベルの.protoファイルで、対応するメッセージ型インターフェースに対して[`google.protobuf.Any`](https://github.com/protocolbuffers/protobuf/blob/master/src/google/protobuf/any.proto)を使用する必要があります。

例えば、`x/evidence`モジュールでは`Evidence`インターフェースが定義されており、`MsgSubmitEvidence`で使用されています。構造の定義は、証拠ファイルをラップするために`sdk.Any`を使用する必要があります。.protoファイルでは以下のように定義します：

```protobuf
// proto/cosmos/evidence/v1beta1/tx.proto

message MsgSubmitEvidence {
  string              submitter = 1;
  google.protobuf.Any evidence  = 2 [(cosmos_proto.accepts_interface) = "cosmos.evidence.v1beta1.Evidence"];
}
```

Cosmos SDKの`codec.Codec`インターフェースは、状態を`Any`にエンコードするためのサポートメソッド`MarshalInterface`と`UnmarshalInterface`を提供しています。

モジュールは、インターフェースを`InterfaceRegistry`を使用して登録する必要があります。これには、インターフェースを登録するメカニズムが提供されています：`RegisterInterface(protoName string, iface interface{}, impls ...proto.Message)`および実装：`RegisterImplementations(iface interface{}, impls ...proto.Message)`。これらは、Aminoの型登録と同様にAnyから安全にアンパックできるものです：

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/codec/types/interface_registry.go#L28-L75
```

さらに、デシリアライズ時には`UnpackInterfaces`フェーズを導入して、必要になる前にインターフェースをアンパックする必要があります。直接またはメンバーのいずれかを介してprotobufの`Any`を含むProtobufタイプは、`UnpackInterfacesMessage`インターフェースを実装する必要があります。

```go
type UnpackInterfacesMessage interface {
  UnpackInterfaces(InterfaceUnpacker) error
}
```
