---
title: Goの <span>Fuzzing</span> による入力バリエーションの自動テスト
title_safe: GoのFuzzingによる入力バリエーションの自動テスト
highlight: cyan
pack: duotone
icon: bomb
draft: false
toc: true
comments: false
date: 2026-07-06T14:00:00+09:00
category: Go
tags:
  - go
  - testing
---
Go 1.18からは `go test -fuzz` によるファジングテストが標準ツールチェインに統合され、外部ライブラリなしでランダムな入力を生成して境界値やパニックを探索できます。今回は、FuzzXxx関数の書き方からクラッシュの再現、CIでの運用方法までを整理します。

<!--more-->

## FuzzXxx関数とf.Addによるシードコーパス

ファジングテストは `_test.go` ファイルの中に `FuzzXxx(f *testing.F)` という形式で定義します。`f.Add` で渡す値は「シードコーパス」と呼ばれ、ファジングエンジンが変異を加える出発点になります。

```go
// reverse_test.go
package stringutil

import "testing"

func FuzzReverse(f *testing.F) {
	// シードコーパス: 代表的な入力をいくつか登録しておく
	testcases := []string{"Hello, world", " ", "!12345", "日本語"}
	for _, tc := range testcases {
		f.Add(tc)
	}

	f.Fuzz(func(t *testing.T, orig string) {
		rev := Reverse(Reverse(orig))
		if orig != rev {
			t.Errorf("Before: %q, after: %q", orig, rev)
		}
	})
}
```

シードは境界値（空文字、マルチバイト文字など）を意識して選ぶと、探索の初動が効率的になります。

## f.Fuzzの書き方とテスト対象の実装

`f.Fuzz` に渡す関数の第2引数以降がファジング対象のパラメータで、`string` `[]byte` `int` などの限られた型のみ指定できます。テスト対象は次のような素朴な文字列反転関数です。

```go
// reverse.go
package stringutil

// Reverse は文字列をルーン単位で反転します。
// バイト単位で反転すると不正なUTF-8を生成してしまうため注意が必要です。
func Reverse(s string) string {
	r := []rune(s)
	for i, j := 0, len(r)-1; i < j; i, j = i+1, j-1 {
		r[i], r[j] = r[j], r[i]
	}
	return string(r)
}
```

`go test -fuzz=FuzzReverse -fuzztime=30s` を実行すると、シードコーパスを起点に変異させた入力で `f.Fuzz` 内の関数を繰り返し呼び出し、パニックやテスト失敗を探します。

## クラッシュ発見時のtestdata/fuzzへの保存と再現

失敗を引き起こす入力が見つかると、Goは自動的に `testdata/fuzz/FuzzReverse/` 配下にファイルとして保存します。

```
testdata/fuzz/FuzzReverse/3a7f9c1e2b0d4f5a
```

```
go test fuzz v1
string("\xc0")
```

このファイルは通常の `go test`（`-fuzz` フラグなし）を実行するだけで自動的に読み込まれ、リグレッションテストとして再実行されます。つまりクラッシュを一度発見すれば、修正後も同じ入力で回帰していないことを継続的に検証できます。このファイルはリポジトリにコミットして共有すべき資産です。

## ユニットテストとの使い分け

- **ユニットテスト（table driven test）を使う場面**:
  - 期待する入出力の組み合わせが明確に決まっている仕様確認
  - 正常系・既知の異常系を網羅的に固定パターンで検証したい場合
- **ファジングを使う場面**:
  - パーサーやエンコーダ/デコーダなど、想定していない入力によるパニックやクラッシュを洗い出したい場合
  - `Reverse(Reverse(x)) == x` のように、入出力の関係性（プロパティ）でしか検証できない処理

ファジングはユニットテストの代替ではなく、見落としがちな入力を機械的に補完する位置づけです。

## CIでの運用と注意点

- 通常の `go test ./...` はシードコーパスと保存済みのリグレッションコーパスのみを実行するため決定的で高速です。CIの通常パイプラインにはこちらを組み込みます。
- `-fuzz` を付けた探索的な実行は終了時刻が不定なため、通常のPRチェックには向きません。`-fuzztime` で上限を設定し、ナイトリービルドなど専用ジョブで定期実行するのが実用的です。
- 見つかったクラッシュの再現ファイルは必ずコミットし、レビューを経てから修正することでリグレッションを防げます。
- 長時間のファジングはCPUを大きく消費するため、専用のジョブランナーやタイムアウト設定で他のCIジョブへの影響を抑える必要があります。

## まとめ

Go標準のファジングは、`FuzzXxx` 関数と `f.Add` によるシードコーパス、`f.Fuzz` によるプロパティ検証という最小限の仕組みで、想定外の入力によるバグを継続的に発見できます。通常のCIでは保存済みコーパスの再実行に留め、探索的なファジングは別ジョブで運用するのが実務上のバランスです。
