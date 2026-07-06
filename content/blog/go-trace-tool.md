---
title: <span>go tool trace</span> による並行処理の可視化と最適化
title_safe: go tool traceによる並行処理の可視化と最適化
highlight: cyan
pack: duotone
icon: eye
draft: false
toc: true
comments: false
date: 2026-07-06
category: Go
tags:
  - go
  - concurrency
  - debugging
---
Goアプリケーションの並行処理やゴルーチンのスケジューリングにおけるボトルネックを追求するには、`pprof` だけでは不十分な場合があります。その際に活躍するのが、実行トレースを可視化する `go tool trace` です。

<!--more-->

## 1. トレースデータの出力方法

コード内で `runtime/trace` パッケージを使用して、トレースデータをファイルに書き出します。

```go
package main

import (
	"os"
	"runtime/trace"
)

func main() {
	f, err := os.Create("trace.out")
	if err != nil {
		panic(err)
	}
	defer f.Close()

	// トレースの開始
	if err := trace.Start(f); err != nil {
		panic(err)
	}
	defer trace.Stop()

	// 測定対象の並行処理など
	runApp()
}
```

## 2. トレースビュワーの起動

出力された `trace.out` を使って、ブラウザ上の可視化ツールを起動します。

```sh
go tool trace trace.out
```

このコマンドを実行すると、ローカルサーバーが立ち上がり、自動的にブラウザが開きます。

## 3. 主な解析メニュー

- **View trace**: 最も強力なタイムラインビューです。各OSスレッド、論理プロセッサ (P)、ゴルーチンがどの時点で動き、何によってブロックされていたか（ネットワークI/O、チャネル、ロック競合など）がマイクロ秒単位でグラフ化されます。
- **Goroutine analysis**: 各ゴルーチンの実行時間や、ブロックされていた総時間を分析できます。
- **Network blocking profile**: ネットワークI/Oによるブロックを引き起こしている呼び出し履歴を特定します。

## 4. `trace` で発見できる問題

- **ゴルーチンのシリアライズ（直列化）**: 本来並行で動くべき処理が、共通のミューテックス（ロック）やチャネルの詰まりによって順番待ちになっていないか。
- **Stop-the-World (STW) の影響**: GCによる一時停止がアプリケーション全体に与えているミリ秒単位の影響の確認。
- **CPUコアの未利用**: ゴルーチンの数が少なすぎる、または不適切な並行設計によってCPUパワーを余らせていないか。

## まとめ

`go tool trace` は、極めて細かい時間軸でゴルーチンの動作とランタイムの振る舞いを追跡できるデバッグツールです。並行処理が思ったようにスケールしないと感じたときは、ぜひトレースを取得してタイムラインを確認してみましょう。
