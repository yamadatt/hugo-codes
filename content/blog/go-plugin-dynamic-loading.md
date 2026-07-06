---
title: Goの <span>plugin</span> パッケージによる動的モジュールロード
title_safe: Goのpluginパッケージによる動的モジュールロード
highlight: purple
pack: duotone
icon: lock-open
draft: false
toc: true
comments: false
date: 2026-06-20
category: Go
tags:
  - go
  - plugins
  - architecture
---
Goは通常、依存関係をすべて静的リンクして単一のバイナリを作成しますが、標準パッケージ `plugin` を使用すると、コンパイル済みの共有ライブラリ（`.so` ファイル）を実行時に動的にロードして関数を呼び出すことができます。今回はその実装方法と運用の制限について解説します。

<!--more-->

## 1. プラグイン（共有ライブラリ）の実装

まず、メインプログラムからロードされるプラグイン側のコードを作成します。パッケージ名は必ず `main` とし、エクスポートしたい変数や関数を定義します。

```go
// myplugin.go
package main

import "fmt"

// エクスポートされる変数
var DeveloperName = "Alice"

// エクスポートされる関数
func Greet(msg string) {
	fmt.Printf("Plugin says: %s\n", msg)
}
```

このファイルを以下のコマンドでプラグイン形式にビルドします。

```sh
go build -buildmode=plugin -o myplugin.so myplugin.go
```

## 2. メインプログラムでの動的ロード

メイン側では `plugin` パッケージを使用して `.so` ファイルを読み込み、シンボル検索を行って関数を実行します。

```go
// main.go
package main

import (
	"log"
	"plugin"
)

func main() {
	// プラグインのロード
	p, err := plugin.Open("myplugin.so")
	if err != nil {
		log.Fatal(err)
	}

	// Greet 関数のシンボルを検索
	symGreet, err := p.Lookup("Greet")
	if err != nil {
		log.Fatal(err)
	}

	// アサーションによって関数型にキャストして実行
	greetFunc, ok := symGreet.(func(string))
	if !ok {
		log.Fatal("unexpected type for Greet symbol")
	}

	greetFunc("Hello from main!")
}
```

## 3. plugin パッケージの厳しい制限

この動的ロード機能は魅力的に見えますが、本番システムで採用されることは稀です。それには以下の厳しい理由があります。

1. **プラットフォーム制限**: 動的プラグインは macOS と Linux のみで動作し、Windows では動作しません。
2. **ビルド環境の完全な一致が必要**: メインバイナリと `.so` プラグインは、**全く同じGoコンパイラバージョン**かつ**同一の依存ライブラリのバージョン**でビルドされている必要があります。僅かでも異なるとロード時に拒否されます。

## まとめ

Goにおける動的拡張を実現したい場合、標準の `plugin` パッケージの代わりに、**「gRPC や WebAssembly を用いたサブプロセス型プラグイン」**を設計する方が、安全性とポータビリティの観点から現在では一般的です。
