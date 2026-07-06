---
title: Go 1.22の <span>ServeMux</span> で強化されたルーティングパターン
title_safe: Go 1.22のServeMuxで強化されたルーティングパターン
highlight: orange
pack: duotone
icon: network-wired
draft: false
toc: true
comments: false
date: 2026-07-06T13:00:00+09:00
category: Go
tags:
  - go
  - routing
---
Go 1.21までの `net/http.ServeMux` はパスの完全一致とプレフィックス一致しかサポートしておらず、HTTPメソッドごとのルーティングやパスパラメータの取得にはサードパーティルーターが事実上必須でした。Go 1.22でこの標準ライブラリのルーティング機能が大きく強化され、多くの用途で外部依存なしにAPIサーバーを組めるようになりました。今回は、`ServeMux` に追加されたパターン構文と移行時の判断ポイントを整理します。

<!--more-->

## HTTPメソッドを指定したルーティング

パターンの先頭にHTTPメソッド名とスペースを付けることで、メソッドごとにハンドラーを振り分けられます。

```go
package main

import (
	"fmt"
	"net/http"
)

func main() {
	mux := http.NewServeMux()

	// GET /users はユーザー一覧を返す
	mux.HandleFunc("GET /users", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, "list users")
	})

	// POST /users は新規ユーザーを作成する
	mux.HandleFunc("POST /users", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, "create user")
	})

	http.ListenAndServe(":8080", mux)
}
```

メソッドを省略した従来通りのパターン（例: `"/users"`）は全メソッドにマッチします。登録済みパスに対して未対応のメソッドでリクエストが来た場合、`ServeMux` は自動的に `405 Method Not Allowed` を返します。

## パスワイルドカードと r.PathValue

`{name}` の形式でパスセグメントを変数として取り出せます。値は `r.PathValue` で参照します。

```go
mux.HandleFunc("GET /users/{id}", func(w http.ResponseWriter, r *http.Request) {
	// {id} にマッチした値を取得する
	id := r.PathValue("id")
	fmt.Fprintf(w, "user id: %s\n", id)
})
```

`r.PathValue` はパターンに存在しない名前を指定すると空文字列を返します。型変換（`strconv.Atoi` など）はハンドラー側で行う必要があり、フレームワークのような自動バリデーションは提供されません。

## {path...} による残余マッチ

セグメント名の末尾に `...` を付けると、以降のパス全体を1つの文字列としてキャプチャできます。

```go
mux.HandleFunc("GET /files/{path...}", func(w http.ResponseWriter, r *http.Request) {
	// {path...} はスラッシュを含む残りのパスすべてをまとめて取得する
	rest := r.PathValue("path")
	fmt.Fprintf(w, "requested file: %s\n", rest)
})
```

`/files/docs/2026/report.pdf` のようなリクエストでは `rest` に `"docs/2026/report.pdf"` が入ります。静的ファイル配信やプロキシのような「残りのパスをそのまま下流に渡す」用途に向いています。

## パターンの優先順位規則

複数のパターンが同じリクエストにマッチしうる場合、`ServeMux` は登録順ではなく「より具体的なパターン」を優先します。

- 固定セグメントはワイルドカードより優先される（`/posts/latest` は `/posts/{id}` より優先）
- メソッド指定ありのパターンはメソッド指定なしのパターンより優先される
- 曖昧な優先順位になる組み合わせ（どちらも一意に決められない場合）は `mux.Handle` 登録時にパニックになり、実行前に検出できる

```go
mux.HandleFunc("GET /posts/latest", handleLatestPost)
mux.HandleFunc("GET /posts/{id}", handlePostByID)
// 登録順に関わらず /posts/latest へのリクエストは常に handleLatestPost に届く
```

## サードパーティルーターからの移行判断

`ServeMux` の強化により、単純なREST APIであれば `gorilla/mux` や `chi` などを使わずに実装できるケースが増えました。ただし全面移行が常に正解とは限りません。

- **標準の ServeMux で十分な場面**:
  - CRUD中心のシンプルなAPI（メソッド分岐とID程度のパスパラメータのみ）
  - 依存関係を減らしたいマイクロサービスや小規模ツール
- **サードパーティを継続すべき場面**:
  - 正規表現によるパス制約や、パラメータの型検証をルーター側に持たせたい場合
  - サブルーターのマウントやミドルウェアチェーンの合成など、階層的な構成管理が必要な場合
  - 名前付きルートからのURL逆生成（`url_for` 的な機能）に依存している場合

移行する場合は、既存パターンに `{id:[0-9]+}` のような正規表現制約が含まれていないか確認し、必要なら `{id}` に単純化した上でハンドラー内でバリデーションを行うよう書き換えます。

## まとめ

Go 1.22の `ServeMux` は、メソッド指定・パスワイルドカード・残余マッチ・明確な優先順位規則を備え、標準ライブラリだけで実用的なルーティングを組めるようになりました。正規表現制約や階層的なルート管理が必要ない限り、まずは標準の `ServeMux` から検討する価値があります。
