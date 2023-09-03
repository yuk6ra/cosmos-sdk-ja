---
sidebar_position: 0
---

# Packages

Cosmos SDKはGoモジュールのコレクションです。このセクションでは、Cosmos SDKチェーンの開発時に使用できるさまざまなパッケージのドキュメントを提供します。
これはCosmos SDKの一部としてのスタンドアロンのGoモジュールをすべてリストアップしています。

:::tip
SDKモジュールに関する詳細は、[SDK Modules](https://docs.cosmos.network/main/modules)セクションを参照してください。
SDKのツールに関する詳細は、[Tooling](https://docs.cosmos.network/main/tooling)セクションを参照してください。
:::

## Core

* [Core](https://pkg.go.dev/cosmossdk.io/core) - SDKインターフェースを定義するコアライブラリ ([ADR-063](https://docs.cosmos.network/main/architecture/adr-063-core-module-api))
* [API](https://pkg.go.dev/cosmossdk.io/api) - 生成されたSDK Pulsar APIを含むAPIライブラリ
* [Store](https://pkg.go.dev/cosmossdk.io/store) - Cosmos SDKストアの実装

## State Management

* [Collections](./02-collections.md) - ステート管理ライブラリ
* [ORM](./03-orm.md) - ステート管理ライブラリ

## Automation

* [Depinject](./01-depinject.md) - 依存関係注入フレームワーク
* [Client/v2](https://pkg.go.dev/cosmossdk.io/client/v2) - [AutoCLI](https://docs.cosmos.network/main/building-modules/autocli)をサポートするライブラリ

## Utilities

* [Log](https://pkg.go.dev/cosmossdk.io/log) - ロギングライブラリ
* [Errors](https://pkg.go.dev/cosmossdk.io/errors) - エラー処理ライブラリ
* [Math](https://pkg.go.dev/cosmossdk.io/math) - SDK算術操作のための数学ライブラリ

## Example

* [SimApp](https://pkg.go.dev/cosmossdk.io/simapp) - SimAppは**the**サンプルのCosmos SDKチェーンです。このパッケージはアプリケーションでインポートされるべきではありません。
