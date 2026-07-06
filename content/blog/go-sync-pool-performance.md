---
title: Goの <span>sync.Pool</span> によるメモリ割り当て削減とGC負荷軽減
title_safe: Goのsync.Poolによるメモリ割り当て削減とGC負荷軽減
highlight: rose
pack: duotone
icon: fire
draft: false
toc: true
comments: false
date: 2026-06-14
category: Go
tags:
  - go
  - performance
  - memory
---
GoはGC（ガベージコレクション）を搭載しているためメモリ管理が容易ですが、大量のオブジェクトアロケーションはGCの実行頻度と一時停止時間を増やし、アプリケーションのパフォーマンス低下に繋がります。今回は、オブジェクトを再利用してアロケーションを抑える `sync.Pool` の使い方と仕組みを解説します。

<!--more-->

## なぜオブジェクトを再利用するのか？

GoでHTTPリクエストを処理する際、リクエストごとに巨大なバッファ（スライスや構造体）を生成して破棄するコードを書くと、ヒープメモリへの割り当て（アロケーション）が多発します。
`sync.Pool` は、不要になったオブジェクトを回収してプールに保管し、次の要求時に再利用することで新規アロケーションを抑える並行安全なキャッシュ機構です。

## `sync.Pool` の基本構造

`sync.Pool` のインターフェースは非常にシンプルで、主に次の要素から構成されます。

- **`New` フィールド**: プールが空のときに新しいオブジェクトを生成するファクトリ関数。
- **`Get()` メソッド**: プールからオブジェクトを取り出します。プールが空なら `New` を呼び出します。
- **`Put(x)` メソッド**: オブジェクトをプールに戻し、再利用可能にします。

## 実装例：バイトバッファの再利用

文字列を結合する一時的なバイトバッファを再利用する例です。

```go
package main

import (
	"bytes"
	"fmt"
	"sync"
)

// bufferPool は bytes.Buffer をキャッシュするプールです。
var bufferPool = sync.Pool{
	New: func() interface{} {
		// プールが空のときに呼ばれる生成関数
		return new(bytes.Buffer)
	},
}

func logMessage(user string, action string) string {
	// プールから bytes.Buffer を取得
	buf := bufferPool.Get().(*bytes.Buffer)
	
	// 使用前に必ず初期化（リセット）を行う
	buf.Reset()
	
	buf.WriteString("User: ")
	buf.WriteString(user)
	buf.WriteString(" did action: ")
	buf.WriteString(action)
	
	res := buf.String()
	
	// 使用が終わったらプールに返却
	bufferPool.Put(buf)
	
	return res
}

func main() {
	msg := logMessage("alice", "login")
	fmt.Println(msg)
}
```

## `sync.Pool` の重要な特性と注意点

`sync.Pool` を使う際は、その内部的な挙動について以下の注意が必要です。

1. **GCによる自動解放**: プール内のオブジェクトは、ガベージコレクションが発生したタイミングで破棄される可能性があります。そのため、長期間有効なキャッシュとしてデータを保存する目的には適していません。
2. **状態のリセット**: `Get()` で取得したオブジェクトには前回の使用データが残っています。再利用する前には必ず `Reset()` などのメソッドを実行し、値をクリーンにする必要があります。
3. **アロケーションのオーバーヘッド**: 小さなオブジェクト（数バイトのプリミティブなど）に `sync.Pool` を使うと、プール自体のオーバーヘッド（インターフェース型へのキャストなど）の方が大きくなり、かえって遅くなることがあります。

## まとめ

`sync.Pool` は、ネットワークパケットの処理、JSONのエンコード/デコード用のバッファなど、**「頻繁に生成・破棄される」「比較的サイズが大きい」**オブジェクトを使い回す際に真価を発揮します。まずはプロファイラ (`go tool pprof`) でボトルネックを見つけ、効果的な場所に導入しましょう。
