---
title: <span>database/sql</span> 接続プールの適切なチューニング
title_safe: database/sql接続プールの適切なチューニング
highlight: fuchsia
pack: duotone
icon: database
draft: false
toc: true
comments: false
date: 2026-06-29
category: Go
tags:
  - go
  - database
  - performance
---
Goの標準パッケージ `database/sql` は、裏側でデータベースの接続プール（Connection Pool）を自動的に管理しています。しかし、デフォルト設定のまま高負荷な本番環境で運用すると、過剰な接続生成によるパフォーマンス低下や接続エラーが発生することがあります。今回はプール設定の適切なチューニング方法を解説します。

<!--more-->

## 1. 接続プールの主要な設定項目

`*sql.DB` 構造体には、プール動作を制御するための4つの主要メソッドがあります。

```go
db, err := sql.Open("postgres", dsn)
if err != nil {
	log.Fatal(err)
}

// 1. 同時接続数の上限
db.SetMaxOpenConns(25)

// 2. アイドル（待機）状態の最大接続数
db.SetMaxIdleConns(25)

// 3. 接続の最大寿命（再利用可能な期間）
db.SetConnMaxLifetime(5 * time.Minute)

// 4. アイドル状態の最大時間
db.SetConnMaxIdleTime(3 * time.Minute)
```

## 2. 各設定のチューニングガイド

### SetMaxOpenConns
同時に開くことができるデータベース接続の上限です。これを無限（デフォルトの `0`）にしておくと、急激なアクセス増の際にデータベース側の最大接続数を超えてしまい、接続拒否エラー（`too many connections`）が発生します。
データベースサーバーのスペックに応じて適切な値を制限する必要があります。

### SetMaxIdleConns
プール内に保持するアイドル接続の最大数です。
**非常に重要**: この値を `SetMaxOpenConns` と同じか、それに近い値に設定することが推奨されます。
デフォルトは `2` ですが、もし同時接続制限を `25` にしてアイドル最大数を `2` のままにすると、高頻度で接続が作成されては破棄されるため、TCPコネクションのハンドシェイクコストが頻発しパフォーマンスが大幅に低下します。

### SetConnMaxLifetime
接続の最大寿命です。データベース側の接続タイムアウトや、ロードバランサ等による切断を避けるため、接続が作られてから一定時間経過したものは自動的に廃棄・再作成されるようにします。一般的には数分から十数分程度に設定します。

## まとめ

`database/sql` をチューニングする第一歩は、`SetMaxOpenConns` と `SetMaxIdleConns` を適切な同一値に揃え、接続の頻繁なクローズとオープンを防ぐことです。アプリケーションの並行処理数とデータベースの処理能力を計測しながら調整しましょう。
