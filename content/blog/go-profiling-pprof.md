---
title: <span>pprof</span> を使ったGoアプリケーションのプロファイリング入門
title_safe: pprofを使ったGoアプリケーションのプロファイリング入門
highlight: red
pack: duotone
icon: timeline
draft: false
toc: true
comments: false
date: 2026-06-19
category: Go
tags:
  - go
  - performance
  - profiling
---
Goには、CPU使用率やメモリ割り当てのボトルネックを可視化・分析するための標準プロファイラ `pprof` が用意されています。今回は、Webサーバーなどのアプリケーションで `pprof` を有効化し、パフォーマンスを改善する手順を解説します。

<!--more-->

## 1. Webサーバーでの pprof の有効化

`net/http/pprof` をインポートするだけで、`/debug/pprof` エンドポイントが自動的に登録されます。

```go
package main

import (
	"log"
	"net/http"
	_ "net/http/pprof" // インポートするだけでエンドポイントが有効化される
)

func main() {
	log.Println("Starting server on :8080...")
	// 標準の http.ListenAndServe を使用する場合
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

## 2. プロファイルデータの収集

サーバーが起動している状態で、`go tool pprof` コマンドを使用してデータを収集します。

### CPUプロファイルの取得 (30秒間)
```sh
go tool pprof http://localhost:8080/debug/pprof/profile?seconds=30
```

### メモリプロファイル（ヒープアロケーション）の取得
```sh
go tool pprof http://localhost:8080/debug/pprof/heap
```

## 3. 対話モードでの分析

コマンドを実行すると、pprof のインタラクティブシェルが起動します。以下のコマンドが特に有用です。

- `top10`: CPUやメモリを最も消費している上位10関数を表示します。
- `flat`: その関数自体で消費された量。
- `cum`: その関数から呼び出された関数も含めた累積の消費量。
- `list <関数名>`: 指定した関数のソースコードを1行ずつ解析し、どの行で負荷が発生しているかを表示します。
- `web`: コールグラフをWebブラウザで画像として表示します（Graphvizのインストールが必要）。

## 4. Web UI の起動

`-http` オプションを付けて実行すると、ブラウザ上でグラフやフレームグラフ（Flame Graph）を表示するWeb UIが起動します。

```sh
go tool pprof -http=:8081 http://localhost:8080/debug/pprof/profile?seconds=10
```

これにより、ボトルネックとなっている処理の親子関係を視覚的に一目で確認できます。

## まとめ

`pprof` を使えば、推測ではなく事実に基づいてパフォーマンスチューニングが行えます。負荷の高い本番環境でも、オーバーヘッドを最小限に抑えながらプロファイルを取得できるため、Go運用において必須の知識と言えます。
