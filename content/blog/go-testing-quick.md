---
title: Goの <span>testing</span> パッケージの知られざる便利機能
title_safe: Goのtestingパッケージの知られざる便利機能
highlight: blue
pack: duotone
icon: vial
draft: false
toc: true
comments: false
date: 2026-07-06
category: Go
tags:
  - go
  - testing
---
Goの標準 `testing` パッケージは非常にシンプルですが、テストコードの品質を高め、実行を最適化するための強力な機能が多数組み込まれています。今回は、意外と知られていないテスティングのテクニックを紹介します。

<!--more-->

## 1. `t.Helper()` によるエラー位置の修正

ヘルパー関数を作成してテストコードを共通化する場合、`t.Helper()` を呼び出すことで、テストが失敗した際のエラー報告位置を「ヘルパー関数内」ではなく「ヘルパー関数を呼び出したテスト側」に偽装できます。

```go
func assertEqual(t *testing.T, expected, actual string) {
	t.Helper() // これにより、失敗した行がこの関数内ではなく、呼び出し元の行になります
	if expected != actual {
		t.Errorf("expected %q but got %q", expected, actual)
	}
}
```

## 2. `t.Cleanup()` によるクリーンアップ処理の登録

テスト内のセットアップで一時ファイルやデータベース接続を作成した際、`defer` の代わりに `t.Cleanup()` を使うことで、サブテストも含めたテスト終了時に確実にリソースをクリーンアップできます。

```go
func TestDatabase(t *testing.T) {
	db := setupDB()
	t.Cleanup(func() {
		db.Close() // テスト終了時に実行される
	})
	
	t.Run("SubTest", func(t *testing.T) {
		// db を使ったテスト
	})
}
```

`defer` は定義された関数を抜けるときに実行されますが、`t.Cleanup` はテストフレームワーク全体がそのテストの完了を検知したタイミングで実行されるため、より安全です。

## 3. `t.Parallel()` によるテストの並行実行

I/O待ちが発生する統合テストなどでは、`t.Parallel()` を呼び出すことで、他の並行テストと同時にテストを実行し、全体の実行時間を大幅に短縮できます。

```go
func TestSlowAPI(t *testing.T) {
	t.Parallel() // 他の t.Parallel() が指定されたテストと並行に走る
	// テスト処理
}
```

## まとめ

`testing` パッケージは、`t.Helper()` や `t.Cleanup()`、`t.Parallel()` などの組み合わせによって、標準機能だけでメンテナンス性の高いテストスイートを構築できるよう設計されています。
