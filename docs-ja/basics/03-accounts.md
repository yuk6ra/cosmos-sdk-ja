---
sidebar_position: 1
---

# Accounts

:::note Synopsis
This document describes the in-built account and public key system of the Cosmos SDK.
:::

:::note Pre-requisite Readings


* [Anatomy of a Cosmos SDK Application](./00-app-anatomy.md)

:::

## Account Definition

Cosmos SDKにおいて、_アカウント_は_公開鍵_ `PubKey` と_秘密鍵_ `PrivKey` のペアを指します。`PubKey` は、様々な `Addresses` を生成するために派生できます。これらの `Addresses` は、アプリケーション内でユーザー（および他の関係者）を識別するために使用されます。また、`Addresses` は [`message`](../building-modules/02-messages-and-queries.md#messages) にも関連付けられており、`message` の送信者を識別するために使用されます。`PrivKey` は [デジタル署名](#signatures) を生成するために使用され、`PrivKey` に関連付けられた `Address` が特定の `message` を承認したことを証明します。

HD鍵派生では、Cosmos SDKは[BIP32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki)と呼ばれる標準を使用しています。BIP32により、ユーザーはHDウォレットを作成できます（[BIP44](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki)で指定されています）。これは初期の秘密シードから派生した一連のアカウントを指します。シードは通常、12語または24語のニーモニックから作成されます。1つのシードは、一方向の暗号関数を使用して任意の数の `PrivKey` を派生できます。その後、`PubKey` を `PrivKey` から派生できます。もちろん、ニーモニックは最も機密性の高い情報であり、秘密鍵は常にニーモニックが保持されている場合に再生成できます。


```text
     Account 0                         Account 1                         Account 2

+------------------+              +------------------+               +------------------+
|                  |              |                  |               |                  |
|    Address 0     |              |    Address 1     |               |    Address 2     |
|        ^         |              |        ^         |               |        ^         |
|        |         |              |        |         |               |        |         |
|        |         |              |        |         |               |        |         |
|        |         |              |        |         |               |        |         |
|        +         |              |        +         |               |        +         |
|  Public key 0    |              |  Public key 1    |               |  Public key 2    |
|        ^         |              |        ^         |               |        ^         |
|        |         |              |        |         |               |        |         |
|        |         |              |        |         |               |        |         |
|        |         |              |        |         |               |        |         |
|        +         |              |        +         |               |        +         |
|  Private key 0   |              |  Private key 1   |               |  Private key 2   |
|        ^         |              |        ^         |               |        ^         |
+------------------+              +------------------+               +------------------+
         |                                 |                                  |
         |                                 |                                  |
         |                                 |                                  |
         +--------------------------------------------------------------------+
                                           |
                                           |
                                 +---------+---------+
                                 |                   |
                                 |  Master PrivKey   |
                                 |                   |
                                 +-------------------+
                                           |
                                           |
                                 +---------+---------+
                                 |                   |
                                 |  Mnemonic (Seed)  |
                                 |                   |
                                 +-------------------+
```

Cosmos SDKでは、鍵は[`Keyring`](#keyring)と呼ばれるオブジェクトを使用して保存および管理されます。

## Keys, accounts, addresses, and signatures

ユーザーの認証は、[デジタル署名](https://en.wikipedia.org/wiki/Digital_signature)を使用して行います。ユーザーは自分の秘密鍵を使用してトランザクションに署名します。署名の検証は関連する公開鍵で行われます。オンチェーンの署名検証のために、公開鍵は`Account`オブジェクトに格納されます（適切なトランザクションの検証に必要な他のデータとともに）。

ノードでは、すべてのデータはProtocol Buffersシリアル化を使用して保存されます。

Cosmos SDKは、デジタル署名を作成するための次のデジタル鍵スキームをサポートしています：

* `secp256k1`：[Cosmos SDKの`crypto/keys/secp256k1`パッケージ](https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/crypto/keys/secp256k1/secp256k1.go)で実装されています。
* `secp256r1`：[Cosmos SDKの`crypto/keys/secp256r1`パッケージ](https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/crypto/keys/secp256r1/pubkey.go)で実装されています。
* `tm-ed25519`：[Cosmos SDKの`crypto/keys/ed25519`パッケージ](https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/crypto/keys/ed25519/ed25519.go)で実装されています。このスキームは、コンセンサス検証のみでサポートされています。


|              | Address length in bytes | Public key length in bytes | Used for transaction authentication | Used for consensus (cometbft) |
| :----------: | :---------------------: | :------------------------: | :---------------------------------: | :-----------------------------: |
| `secp256k1`  |           20            |             33             |                 yes                 |               no                |
| `secp256r1`  |           32            |             33             |                 yes                 |               no                |
| `tm-ed25519` |     -- not used --      |             32             |                 no                  |               yes               |

## Addresses

`Addresses` および `PubKey` は、アプリケーション内のアクターを識別するための公開情報です。`Account` は認証情報を格納するために使用されます。基本的なアカウントの実装は `BaseAccount` オブジェクトによって提供されます。

各アカウントは、公開鍵から派生したバイトのシーケンスである `Address` を使用して識別されます。Cosmos SDKでは、アカウントが使用されるコンテキストを指定する3つのタイプのアドレスを定義しています：

* `AccAddress` はユーザーを識別します（`message`の送信者）。
* `ValAddress` はバリデーターオペレーターを識別します。
* `ConsAddress` はコンセンサスに参加するバリデーターノードを識別します。バリデーターノードは **`ed25519`** 曲線を使用して派生されます。

これらのタイプは `Address` インターフェースを実装しています：

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/types/address.go#L126-L134
```

アドレスの構築アルゴリズムは [ADR-28](https://github.com/cosmos/cosmos-sdk/blob/main/docs/architecture/adr-028-public-key-addresses.md) で定義されています。
ここでは `pub` 公開鍵からアカウントアドレスを取得するための標準的な方法が示されています：

```go
sdk.AccAddress(pub.Address().Bytes())
```

注意すべきは、`Marshal()` メソッドと `Bytes()` メソッドがどちらもアドレスの同じ生の `[]byte` 形式を返すことです。Protobufの互換性のために `Marshal()` メソッドが必要です。

ユーザーとのインタラクションにおいて、アドレスは [Bech32](https://en.bitcoin.it/wiki/Bech32) を使用してフォーマットされ、`String` メソッドによって実装されます。Bech32メソッドは、ブロックチェーンと対話する際に使用する唯一のサポートされている形式です。Bech32のヒューマンリーダブルパート（Bech32プレフィックス）は、アドレスのタイプを示すために使用されます。例：

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/types/address.go#L299-L316
```

|                    | Address Bech32 Prefix |
| ------------------ | --------------------- |
| Accounts           | cosmos                |
| Validator Operator | cosmosvaloper         |
| Consensus Nodes    | cosmosvalcons         |

### Public Keys

Public keys in Cosmos SDK are defined by `cryptotypes.PubKey` interface. Since public keys are saved in a store, `cryptotypes.PubKey` extends the `proto.Message` interface:

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/crypto/types/types.go#L8-L17
```

`secp256k1` および `secp256r1` シリアル化には圧縮フォーマットが使用されます。

* 最初のバイトは、`y`-座標が `x`-座標に関連付けられた2つの座標のうち、辞書的に最大のものである場合、`0x02` バイトです。
* それ以外の場合、最初のバイトは `0x03` です。

このプレフィックスの後に `x`-座標が続きます。

公開鍵はアカウント（またはユーザー）を参照するために使用されず、一般的にはトランザクションメッセージを作成する際には使用されません（いくつかの例外を除く：`MsgCreateValidator`、`Validator`、`Multisig` メッセージなど）。
ユーザーとのインタラクションでは、`PubKey` は Protobufs JSON 形式でフォーマットされます（[ProtoMarshalJSON](https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/codec/json.go#L14-L34) 関数）。例：

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/client/keys/output.go#L23-L39
```

## Keyring

`Keyring` はアカウントを格納し管理するオブジェクトです。Cosmos SDK では、`Keyring` の実装は `Keyring` インターフェースに従います：

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/crypto/keyring/keyring.go#L57-L105
```

デフォルトの `Keyring` の実装は、サードパーティの [`99designs/keyring`](https://github.com/99designs/keyring) ライブラリから提供されます。

`Keyring` メソッドに関するいくつかの注意点：

* `Sign(uid string, msg []byte) ([]byte, types.PubKey, error)` は `msg` バイトの署名に厳密に関連しています。トランザクションを正規の `[]byte` フォームに準備してエンコードする必要があります。Protobuf は決定論的ではないため、[ADR-020](../architecture/adr-020-protobuf-transaction-encoding.md) で署名する正規の `payload` は `SignDoc` 構造体であり、[ADR-027](../architecture/adr-027-deterministic-protobuf-serialization.md) を使用して確定的にエンコードされています。なお、署名の検証はCosmos SDKではデフォルトで実装されておらず、代わりに [`anteHandler`](../core/00-baseapp.md#antehandler) に処理が委譲されています。

```protobuf reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/proto/cosmos/tx/v1beta1/tx.proto#L50-L66
```

* `NewAccount(uid, mnemonic, bip39Passphrase, hdPath string, algo SignatureAlgo) (*Record, error)` は [`bip44 path`](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki) に基づいて新しいアカウントを作成し、ディスクに永続化します。`PrivKey` は**決して平文で保存されません**。代わりに、[パスフレーズで暗号化されます](https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/crypto/armor.go)。このメソッドのコンテキストでは、キータイプとシーケンス番号は、BIP44導出パスのセグメント（例：`0`、`1`、`2`など）を参照し、助記語から秘密鍵と公開鍵を導出するために使用されます。同じ助記語と導出パスを使用して、同じ `PrivKey`、`PubKey`、および `Address` が生成されます。次のキーは、キーリングでサポートされています：

* `secp256k1`
* `ed25519`

* `ExportPrivKeyArmor(uid, encryptPassphrase string) (armor string, err error)` は、指定したパスフレーズを使用して、ASCII-armored 暗号化形式で秘密鍵をエクスポートします。その後、`ImportPrivKey(uid, armor, passphrase string)` 関数を使用して再び秘密鍵をキーリングにインポートするか、`UnarmorDecryptPrivKey(armorStr string, passphrase string)` 関数を使用して生の秘密鍵に復号化できます。

### Create New Key Type

キーリングで使用する新しいキータイプを作成するには、`keyring.SignatureAlgo` インターフェースを満たす必要があります。

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/crypto/keyring/signing_algorithms.go#L10-L15
```

このインターフェースは、3つのメソッドから成り、`Name()` はアルゴリズムの名前を `hd.PubKeyType` として返し、`Derive()` および `Generate()` はそれぞれ以下の関数を返す必要があります：

```go reference
https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/crypto/hd/algo.go#L28-L31
```

`keyring.SignatureAlgo` が実装されると、キーリングの[サポートされているアルゴリズムのリスト](https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/crypto/keyring/keyring.go#L217)に追加する必要があります。

新しいキータイプの実装は、`crypto/hd` パッケージ内で行うべきです。

[algo.go](https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/crypto/hd/algo.go#L38)には、動作する `secp256k1` 実装の例があります。

#### Implementing secp256r1 algo

以下は、secp256r1 の実装例です。

まず、シークレット番号から秘密鍵を作成するための新しい関数が secp256r1 パッケージに必要です。この関数は次のようになる可能性があります。

```go
// cosmos-sdk/crypto/keys/secp256r1/privkey.go

// NewPrivKeyFromSecret creates a private key derived for the secret number
// represented in big-endian. The `secret` must be a valid ECDSA field element.
func NewPrivKeyFromSecret(secret []byte) (*PrivKey, error) {
	var d = new(big.Int).SetBytes(secret)
	if d.Cmp(secp256r1.Params().N) >= 1 {
		return nil, errorsmod.Wrap(errors.ErrInvalidRequest, "secret not in the curve base field")
	}
	sk := new(ecdsa.PrivKey)
	return &PrivKey{&ecdsaSK{*sk}}, nil
}
```

その後、`secp256r1Algo` を実装できます。

```go
// cosmos-sdk/crypto/hd/secp256r1Algo.go

package hd

import (
	"github.com/cosmos/go-bip39"
	
	"github.com/cosmos/cosmos-sdk/crypto/keys/secp256r1"
	"github.com/cosmos/cosmos-sdk/crypto/types"
)

// Secp256r1Type uses the secp256r1 ECDSA parameters.
const Secp256r1Type = PubKeyType("secp256r1")

var Secp256r1 = secp256r1Algo{}

type secp256r1Algo struct{}

func (s secp256r1Algo) Name() PubKeyType {
	return Secp256r1Type
}

// Derive derives and returns the secp256r1 private key for the given seed and HD path.
func (s secp256r1Algo) Derive() DeriveFn {
	return func(mnemonic string, bip39Passphrase, hdPath string) ([]byte, error) {
		seed, err := bip39.NewSeedWithErrorChecking(mnemonic, bip39Passphrase)
		if err != nil {
			return nil, err
		}

		masterPriv, ch := ComputeMastersFromSeed(seed)
		if len(hdPath) == 0 {
			return masterPriv[:], nil
		}
		derivedKey, err := DerivePrivateKeyForPath(masterPriv, ch, hdPath)

		return derivedKey, err
	}
}

// Generate generates a secp256r1 private key from the given bytes.
func (s secp256r1Algo) Generate() GenerateFn {
	return func(bz []byte) types.PrivKey {
		key, err := secp256r1.NewPrivKeyFromSecret(bz)
		if err != nil {
			panic(err)
		}
		return key
	}
}
```

最後に、アルゴリズムをキーリングの[サポートされているアルゴリズムのリスト](https://github.com/cosmos/cosmos-sdk/blob/v0.50.0-alpha.0/crypto/keyring/keyring.go#L217)に追加する必要があります。

```go
// cosmos-sdk/crypto/keyring/keyring.go

func newKeystore(kr keyring.Keyring, cdc codec.Codec, backend string, opts ...Option) keystore {
	// Default options for keybase, these can be overwritten using the
	// Option function
	options := Options{
		SupportedAlgos:       SigningAlgoList{hd.Secp256k1, hd.Secp256r1}, // added here
		SupportedAlgosLedger: SigningAlgoList{hd.Secp256k1},
	}
...
```

その後、アルゴリズムを使用して新しいキーを作成するには、フラグ `--algo` を指定する必要があります：

`simd keys add myKey --algo secp256r1`
