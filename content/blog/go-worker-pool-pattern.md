---
title: <span>Worker Pool</span> パターンによるゴルーチン制限と流量制御
title_safe: Worker Poolパターンによるゴルーチン制限と流量制御
highlight: emerald
pack: duotone
icon: users-gear
draft: false
toc: true
comments: false
date: 2026-07-06
category: Go
tags:
  - go
  - concurrency
  - worker-pool
---
Goではゴルーチンを非常に軽量に作成できますが、数百万個といった大量のタスクをそれぞれ別ゴルーチンで無制限に並行実行すると、メモリの枯渇やCPUコアの奪い合いが発生し、システム全体が停止することがあります。今回は並行数を一定に制限する「ワーカープール（Worker Pool）」パターンの実装方法を解説します。

<!--more-->

## 1. ワーカープールパターンの基本設計

このパターンは以下の要素から構成されます。
1. **タスクチャネル**: 実行待ちのタスクを投入するチャネル。
2. **結果チャネル**: 処理結果を受け取るためのチャネル。
3. **ワーカー**: チャネルからタスクを取り出して処理するゴルーチンの集まり（数は固定）。

## 2. 実装コード

固定されたワーカー数（例: 3つ）でタスクを処理する例です。

```go
package main

import (
	"fmt"
	"time"
)

// worker は、jobsチャネルから仕事を読み取って処理します。
func worker(id int, jobs <-chan int, results chan<- int) {
	for j := range jobs {
		fmt.Printf("worker:%d started job:%d\n", id, j)
		time.Sleep(time.Second) // 模擬タスク処理
		fmt.Printf("worker:%d finished job:%d\n", id, j)
		results <- j * 2
	}
}

func main() {
	const numJobs = 5
	jobs := make(chan int, numJobs)
	results := make(chan int, numJobs)

	// 3つのワーカー（ゴルーチン）をあらかじめ起動
	for w := 1; w <= 3; w++ {
		go worker(w, jobs, results)
	}

	// ジョブをチャネルに投入
	for j := 1; j <= numJobs; j++ {
		jobs <- j
	}
	close(jobs) // これ以上ジョブがないことを通知

	// すべての結果を回収
	for a := 1; a <= numJobs; a++ {
		<-results
	}
}
```

## 3. なぜこのパターンが必要なのか？

- **リソースの定常化**: ゴルーチン数が上限に固定されるため、急激なアクセス増があってもメモリ消費が急増せず、OOM（Out of Memory）を防ぐことができます。
- **コネクション競合の回避**: データベースや外部API接続のプールを消費し尽くすことを防ぎます。

## まとめ

ワーカープールパターンは、大量のタスクを安全かつ効率的に並行処理するための基本的なイディオムです。タスク数やI/O負荷に応じて適切なワーカー数を設定し、システムの安定運用を実現しましょう。
