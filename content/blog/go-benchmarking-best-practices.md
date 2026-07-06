---
title: Goの <span>Benchmark</span> テストを正確に測定する手法
title_safe: GoのBenchmarkテストを正確に測定する手法
highlight: yellow
pack: duotone
icon: stopwatch
draft: false
toc: true
comments: false
date: 2026-07-06
category: Go
tags:
  - go
  - testing
  - benchmarking
---
Goには標準のテスティングツールにベンチマーク測定機能が組み込まれています。しかし、正確なベンチマークをとるためには、タイマーの制御やコンパイラの最適化による影響などを正しく理解する必要があります。今回はそのベストプラクティスを解説します。

<!--more-->

## 1. 基本的なベンチマークの書き方

ベンチマーク関数は `Benchmark` から始まり、引数に `*testing.B` を取ります。

```go
package mypkg

import (
	"strconv"
	"testing"
)

func BenchmarkItoa(b *testing.B) {
	for i := 0; i < b.N; i++ {
		_ = strconv.Itoa(i)
	}
}
```

実行するには `-bench` フラグを指定します。

```sh
go test -bench=. -benchmem
```
`-benchmem` を指定すると、実行ごとのメモリ割り当て回数（allocs/op）とバイト数（B/op）も出力されます。

## 2. タイマーの適切なコントロール

重い初期化処理が含まれる場合、測定対象から除外するために `b.ResetTimer()` を使用します。また、ループ内で一時的にタイマーを止めたい場合は `b.StopTimer()` と `b.StartTimer()` を使います。

```go
func BenchmarkProcess(b *testing.B) {
	// 重い初期化
	setupData()
	
	b.ResetTimer() // ここまでの実行時間をリセット
	for i := 0; i < b.N; i++ {
		runProcess()
	}
}
```

## 3. コンパイラの最適化（デッドコード削除）を防ぐ

Goのコンパイラは非常に優秀なため、変数への割り当てがあるだけで実際には使用されていないコード（デッドコード）を検知すると、処理そのものをコンパイル時に削除してしまうことがあります。これを受けると、ベンチマーク結果が異常に速く見えてしまいます。

これを避けるためには、グローバル変数に処理結果を書き出すパターンが推奨されます。

```go
var result string // パッケージレベルの変数

func BenchmarkFormat(b *testing.B) {
	var r string
	for i := 0; i < b.N; i++ {
		r = formatNumber(i) // ローカル変数に代入
	}
	result = r // 最終結果をグローバル変数に代入してコンパイラの最適化を防ぐ
}
```

## まとめ

Goのベンチマーク測定は非常に手軽ですが、正しい数値を出すためには「初期化処理の除外」と「コンパイラによる最適化への対策」が不可欠です。本番環境に近い条件で正確な測定を行い、確実なコード改善に繋げましょう。
