---
title: Goの標準構造化ロギング <span>slog</span> の使い方
title_safe: Goの標準構造化ロギングslogの使い方
highlight: indigo
pack: duotone
icon: list-check
draft: false
toc: true
comments: false
date: 2026-07-06
category: Go
tags:
  - go
  - logging
---
Go 1.21 から、標準ライブラリに待望の構造化ロギングパッケージ `log/slog` が導入されました。サードパーティのライブラリ（ZapやZerologなど）を導入することなく、キーバリュー形式やJSON形式の構造化ログを高速かつ標準的なAPIで出力できます。今回は `slog` の基本的な使い方とカスタマイズ方法を解説します。

<!--more-->

## 構造化ログとは？

従来のテキストログは人間に読みやすい反面、プログラムによる自動パースや検索（DatadogやCloudWatch Logsなどでの集計）が困難でした。構造化ログは、情報をJSONなどの形式で出力し、機械判読性を高める手法です。

## slogの基本出力

`slog` を使って、テキスト形式とJSON形式でログを出力してみます。

```go
package main

import (
	"log/slog"
	"os"
)

func main() {
	// デフォルトハンドラ（テキスト形式）で出力
	slog.Info("hello, world", "user", "alice", "role", "admin")

	// JSON形式で標準出力に出力するロガーを設定
	jsonLogger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
	
	// アプリ全体のグローバルロガーとして設定
	slog.SetDefault(jsonLogger)

	// JSON形式での出力
	slog.Info("user logged in", "userID", 42, "status", "success")
}
```

JSONの出力結果は以下のようになります。
```json
{"time":"2026-07-06T12:00:00.000Z","level":"INFO","msg":"user logged in","userID":42,"status":"success"}
```

## より型安全なキーバリュー指定

`slog.Info` に渡す引数は `any`（可変長引数）のため、キーと値のペアがずれてしまうミスが発生する可能性があります。これを防ぎ、さらに処理を高速化するために `slog.Attr` を使った指定方法が提供されています。

```go
slog.LogAttrs(
	context.Background(),
	slog.LevelInfo,
	"database query execution",
	slog.String("query", "SELECT * FROM users"),
	slog.Int("duration_ms", 15),
)
```

## ロググループの利用

関連するフィールドをJSONのネストしたオブジェクトにまとめるために `slog.Group` を使います。

```go
slog.Info("api request completed",
	slog.Group("request",
		slog.String("method", "POST"),
		slog.String("path", "/users"),
	),
	slog.Int("status", 201),
)
```

**出力JSON:**
```json
{
  "time": "2026-07-06T12:00:00.000Z",
  "level": "INFO",
  "msg": "api request completed",
  "request": {
    "method": "POST",
    "path": "/users"
  },
  "status": 201
}
```

## ハンドラのカスタマイズ (LogLevelの制御など)

特定のログレベル（例: Debug）を表示するかどうかの設定は、ハンドラのオプションを定義して指定します。

```go
opts := &slog.HandlerOptions{
	Level: slog.LevelDebug, // Debug以上のログを出力
}
logger := slog.New(slog.NewJSONHandler(os.Stdout, opts))
```

## まとめ

`log/slog` パッケージの登場により、Goの標準ライブラリだけでハイパフォーマンスかつ高度な構造化ログを扱えるようになりました。サードパーティ製ロガーとの相互運用性も考慮されているため、新規プロジェクトでは積極的に `slog` を採用すると良いでしょう。
