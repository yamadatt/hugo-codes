---
title: <span>errgroup</span> パッケージによる並行処理の同期とエラーハンドリング
title_safe: errgroupパッケージによる並行処理の同期とエラーハンドリング
highlight: violet
pack: duotone
icon: object-group
draft: false
toc: true
comments: false
date: 2026-07-06
category: Go
tags:
  - go
  - concurrency
  - error-handling
---
Goの標準パッケージ `sync.WaitGroup` は、複数のゴルーチンの完了を待機するのに便利です。しかし、「どれか1つでもエラーを返したら他のゴルーチンもキャンセルし、エラーを呼び出し元に返したい」といったユースケースでは、設計が複雑になります。これをスマートに解決するのが `golang.org/x/sync/errgroup` パッケージです。

<!--more-->

## 1. errgroup の基本

`errgroup` は、ゴルーチンのグループに対してエラー伝播とキャンセルのセマンティクスを提供します。

```go
package main

import (
	"context"
	"fmt"
	"net/http"
	"golang.org/x/sync/errgroup"
)

func main() {
	// エラー時に他のタスクをキャンセルするためのContext付きGroupを作成
	g, ctx := errgroup.WithContext(context.Background())

	urls := []string{
		"https://golang.org",
		"https://google.com",
		"https://invalid-domain-name.xxx",
	}

	for _, url := range urls {
		url := url // ゴルーチンにキャプチャさせるためのコピー
		g.Go(func() error {
			req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
			if err != nil {
				return err
			}
			resp, err := http.DefaultClient.Do(req)
			if err != nil {
				return err // このエラーがグループに報告される
			}
			resp.Body.Close()
			return nil
		})
	}

	// すべての処理の完了、または最初のエラー発生を待つ
	if err := g.Wait(); err != nil {
		fmt.Printf("Successfully handled error: %v\n", err)
	}
}
```

## 2. なぜ `errgroup.WithContext` を使うべきか？

- **最初のエラーで即キャンセル**: `errgroup.WithContext(ctx)` で得られるコンテキストは、グループ内のいずれか1つのタスクが `error` を返した時点で自動的にキャンセルされます。
- **リソースの浪費を防ぐ**: 1つのURL取得が失敗した時点で、他の並行処理も即座に通信を中断できるため、接続やCPU資源の無駄を防ぐことができます。

## まとめ

`sync.WaitGroup` でエラー管理をするために自前のチャネルや `sync.Once` を使って複雑なコードを書く前に、まずは `errgroup` を検討してみてください。並行処理のエラーハンドリングが劇的にシンプルになります。
