---
title: Goの <span>range-over-func</span> で作る独自イテレータ
title_safe: Goのrange-over-funcで作る独自イテレータ
highlight: emerald
pack: duotone
icon: infinity
draft: false
toc: true
comments: false
date: 2026-07-06T12:00:00+09:00
category: Go
tags:
  - go
  - iterator
---
Go 1.23では、`for range`が関数を直接受け取れるようになる「range-over-func」と、それを支える`iter`パッケージが導入されました。これにより、スライスやマップに変換せずとも、任意のデータ構造を`for range`でそのまま走査できます。今回は、range-over-funcの基本と独自イテレータの書き方を整理します。

<!--more-->

## range-over-func とは

Go 1.23以降、`for range`は次の3種類の関数シグネチャを受け付けます。

```go
func(yield func() bool)
func(yield func(V) bool)
func(yield func(K, V) bool)
```

後ろの2つは標準ライブラリの`iter`パッケージで`iter.Seq[V]`、`iter.Seq2[K, V]`という型として定義されています。`for range`のループ本体は、この`yield`関数として扱われます。イテレータ側は要素を1つ生成するたびに`yield`を呼び出し、戻り値が`false`ならループが打ち切られたことを知って処理を中断します。

## iter.Seq / iter.Seq2 で独自イテレータを書く

独自の型に対して`iter.Seq`を返すメソッドを定義すると、`for range`でそのまま走査できます。

```go
package main

import "fmt"

// Queue は先入れ先出しのキューです。
type Queue[T any] struct {
	items []T
}

func (q *Queue[T]) Push(v T) {
	q.items = append(q.items, v)
}

// All は要素を先頭から順に返すイテレータです。
func (q *Queue[T]) All() func(yield func(int, T) bool) {
	return func(yield func(int, T) bool) {
		for i, v := range q.items {
			if !yield(i, v) {
				return // 呼び出し側がbreakした
			}
		}
	}
}

func main() {
	q := &Queue[string]{}
	q.Push("go")
	q.Push("rust")
	q.Push("zig")

	for i, v := range q.All() {
		fmt.Println(i, v)
	}
}
```

`iter.Seq2[int, string]`は`func(yield func(int, string) bool)`と同じ形なので、明示的に型を書かなくてもそのまま`for range`に渡せます。

## slices.Values や maps.Keys との連携

`slices`パッケージと`maps`パッケージには、既存のコレクションを`iter.Seq`に変換する関数が用意されています。

```go
package main

import (
	"fmt"
	"maps"
	"slices"
)

func main() {
	nums := []int{10, 20, 30}

	// slices.Values はスライスの要素を順に返すiter.Seqを返す
	for v := range slices.Values(nums) {
		fmt.Println(v)
	}

	m := map[string]int{"a": 1, "b": 2}

	// maps.Keys はマップのキーを返すiter.Seq。ソートしてスライスに戻す
	keys := slices.Sorted(maps.Keys(m))
	fmt.Println(keys) // [a b]
}
```

`slices.Sorted`や`slices.Collect`のようにイテレータをスライスへ戻す関数もあり、既存の値ベースAPIと自由に行き来できます。

## 早期break時のyield falseの扱い

`for range`側でbreakやreturnが実行されると、`yield`は`false`を返します。イテレータ実装者はこの戻り値を必ずチェックし、以降の処理を中断する必要があります。

```go
func Count(n int) func(yield func(int) bool) {
	return func(yield func(int) bool) {
		for i := 0; i < n; i++ {
			if !yield(i) {
				// ここでリソース解放などの後始末を行う
				return
			}
		}
	}
}
```

`yield`の戻り値を無視して処理を続けると、breakしたはずのループ内で余分な値が生成され続けたり、パニックを招くことがあります。ループ内でリソース（ファイルやロックなど）を扱う場合は、`false`が返ってきた時点で確実に解放するようにします。

## 使いどころと注意点

- **使うべき場面**:
  - ツリーやグラフなど、スライス化するとメモリ効率が悪い構造の走査
  - ファイルの行走査やページネーションAPIなど、途中で打ち切られる可能性がある逐次処理
- **避けるべき場面**:
  - 単純な固定長スライス・マップの走査（既存の`for range`で十分）
  - 並行処理からの呼び出し（`yield`はイテレータを呼び出したゴルーチンと同じゴルーチンで、かつ1回のループ実行につき1回のみ呼び出すのが前提）

## まとめ

range-over-funcと`iter`パッケージにより、独自データ構造を中間スライスに変換せず`for range`で直接走査できるようになりました。`yield`の戻り値を必ず確認して早期終了に対応することが、安全なイテレータ実装の鍵になります。
