---
title: <span>context.Value</span> の正しい使い所とアンチパターン
title_safe: context.Valueの正しい使い所とアンチパターン
highlight: indigo
pack: duotone
icon: code-pull-request
draft: false
toc: true
comments: false
date: 2026-06-30
category: Go
tags:
  - go
  - context
  - clean-code
---
Goの `context` パッケージは、処理のキャンセル信号の伝播に加えて、リクエストスコープの値を格納する `context.WithValue` を提供しています。しかし、この機能は型安全性が失われやすいため、乱用すると「アンチパターン」になりがちです。今回は正しい使い所と代替案について整理します。

<!--more-->

## 1. 避けるべきアンチパターン（関数の隠れた依存関係）

最も避けるべきは、必須のパラメータ（データベース接続、設定値、省略できない引数など）を Context に埋め込んで引き回す設計です。

```go
// アンチパターン: データベースの接続情報をContextに埋めて引き回す
func GetUser(ctx context.Context, id string) (*User, error) {
	db := ctx.Value("database").(*sql.DB) // 型チェックが実行時まで行われないため危険
	return db.QueryUser(id)
}
```

このアプローチでは、関数のシグネチャ（引数定義）からデータベースが必要であることが読み取れなくなり、またテストの際に適切なモックを渡すのが困難になります。データベース接続などは、明示的に構造体のフィールドや関数の引数として渡すべきです。

## 2. 正しい使い所（リクエストスコープの付随的データ）

`context.Value` は、ビジネスロジックに影響を与えない、リクエストに関連する**「付随的かつ一時的なデータ」**を伝播させるために使用します。

- **リクエストID / トレースID**: 複数のマイクロサービスをまたぐ分散トレーシングや構造化ログの相関ID。
- **認証トークン / ログインユーザー情報**: ミドルウェアで検証済みのユーザー識別子（権限チェックやログ記録のみに使うもの）。

## 3. 型安全性を保つための「プライベート型キー」

Context はグローバルに共有される可能性があるため、キーの名前衝突を防ぐために**「エクスポートしない（プライベートな）独自型」**をキーとして定義するのが基本規約です。

```go
package request

import "context"

// 外部から見えない独自の型を定義
type contextKey struct{}

var userKey contextKey

// ゲッターとセッターを関数としてカプセル化して公開する
func WithUser(ctx context.Context, userID string) context.Context {
	return context.WithValue(ctx, userKey, userID)
}

func GetUser(ctx context.Context) (string, bool) {
	userID, ok := ctx.Value(userKey).(string)
	return userID, ok
}
```

こうすることで、他のパッケージが `context.WithValue(ctx, "userID", ...)` と書いてキーが衝突してしまう事故を防ぎ、型キャストの手間もカプセル化できます。

## まとめ

`context.Value` は「リクエストごとの付随的なコンテキスト情報」を伝えるためのものであり、関数の主要な引数を置き換えるものではありません。プライベート型キーを用いて安全にカプセル化し、正しい範囲で活用しましょう。
