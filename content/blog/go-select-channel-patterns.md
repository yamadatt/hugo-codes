---
title: Goの <span>select</span> 文とチャネルの制御パターン
title_safe: Goのselect文とチャネルの制御パターン
highlight: purple
pack: duotone
icon: network-wired
draft: false
toc: true
comments: false
date: 2026-06-17
category: Go
tags:
  - go
  - concurrency
  - channels
---
Goの並行処理において、複数のチャネルを監視して準備ができた処理から実行する `select` 文は非常に強力です。今回は、タイムアウト、キャンセル処理、ノンブロッキング通信など、実務でよく使われる制御パターンを整理します。

<!--more-->

## 1. タイムアウト処理

外部APIの呼び出しや時間のかかる非同期処理を待つ際、一定時間内に応答がなければ処理を打ち切るパターンです。`time.After` チャネルを使用します。

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	ch := make(chan string, 1)

	go func() {
		time.Sleep(2 * time.Second) // 時間がかかる処理
		ch <- "success"
	}()

	select {
	case res := <-ch:
		fmt.Println("Result:", res)
	case <-time.After(1 * time.Second):
		fmt.Println("Error: request timed out")
	}
}
```

このコードでは、非同期処理が2秒かかるのに対し、タイムアウトを1秒に設定しているため、先に `time.After` のケースが実行されます。

## 2. context.Context によるキャンセル制御

Goの標準的なキャンセル伝播手法である `context.Context` と `select` を組み合わせるパターンです。呼び出し元が途中で処理を取り消した（あるいは上位レイヤーでタイムアウトが発生した）ことを検知します。

```go
func processTask(ctx context.Context, data chan int) error {
	for {
		select {
		case <-ctx.Done():
			// キャンセル（または親のタイムアウト）を検知して終了
			return ctx.Err()
		case v := <-data:
			// データを処理
			fmt.Println("Processed:", v)
		}
	}
}
```

`ctx.Done()` チャネルは、コンテキストがキャンセルされるとクローズされます。クローズされたチャネルからの受信はブロックされないため、即座にこのケースが選択されます。

## 3. ノンブロッキングチャネル操作 (default)

`select` に `default` 節を追加すると、チャネルへの送信または受信がブロックされる場合に、即座に別の処理に移るノンブロッキングな通信が可能です。

```go
package main

import "fmt"

func main() {
	ch := make(chan string, 1)

	// 最初の送信はバッファがあるのでブロックされない
	select {
	case ch <- "message":
		fmt.Println("Sent message")
	default:
		fmt.Println("Channel full, skipping")
	}

	// 2回目の送信はバッファが一杯のためブロックされるが、defaultのおかげでスキップされる
	select {
	case ch <- "another message":
		fmt.Println("Sent another message")
	default:
		fmt.Println("Channel full, skipping")
	}
}
```

このパターンは、例えば「レート制限を超えたパケットを破棄する」といったバッファ溢れの処理に有用です。

## 4. nilチャネルによる動的なケースの無効化

`select` 内で `nil` のチャネルに対する送受信を行うと、そのケースは**永久に選択されなくなります**。これを利用して、処理の途中で特定のチャネル監視を動的にオン/オフできます。

```go
func merge(ch1, ch2 <-chan int) {
	for ch1 != nil || ch2 != nil {
		select {
		case v, ok := <-ch1:
			if !ok {
				ch1 = nil // ch1がクローズされたら、監視対象から外す
				continue
			}
			fmt.Println("From ch1:", v)
		case v, ok := <-ch2:
			if !ok {
				ch2 = nil // ch2がクローズされたら、監視対象から外す
				continue
			}
			fmt.Println("From ch2:", v)
		}
	}
}
```

## まとめ

`select` 文は、単に複数の処理を多重化するだけでなく、タイムアウトやキャンセル管理、ノンブロッキング制御などをスマートに記述するためのGoのコア機能です。特に `context.Context` との連携は、堅牢な並行アプリケーションの設計に不可欠です。
