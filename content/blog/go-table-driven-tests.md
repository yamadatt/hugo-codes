---
title: Goの <span>table driven test</span> を読みやすく保つ
title_safe: Goのtable driven testを読みやすく保つ
highlight: orange
pack: duotone
icon: list-check
draft: false
toc: true
comments: false
date: 2026-06-13
category: Go
tags:
  - go
  - testing
---
テーブル駆動テストはGoらしい書き方ですが、ケースを詰め込みすぎると読みづらくなります。テスト名、入力、期待値、検証方法の粒度を揃えることが大切です。

<!--more-->

## ケース名は失敗理由になる

`t.Run`の名前は、CIで落ちたときの最初の手がかりです。入力値の説明ではなく、仕様の違いが伝わる名前にします。

```go
func TestNormalizeEmail(t *testing.T) {
	tests := []struct {
		name string
		in   string
		want string
		err  bool
	}{
		{name: "trims spaces", in: "  USER@example.com ", want: "user@example.com"},
		{name: "rejects missing at mark", in: "example.com", err: true},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got, err := NormalizeEmail(tt.in)
			if tt.err {
				if err == nil {
					t.Fatal("expected error")
				}
				return
			}
			if err != nil {
				t.Fatalf("unexpected error: %v", err)
			}
			if got != tt.want {
				t.Fatalf("got %q, want %q", got, tt.want)
			}
		})
	}
}
```

失敗時に`rejects missing at mark`と出れば、どの仕様が壊れたのかすぐ分かります。

## 比較関数を先に決める

構造体やスライスの比較では、`reflect.DeepEqual`だけに寄せるより、差分が読める比較を選ぶほうが調査時間を減らせます。標準ライブラリだけで済むなら`cmp`相当の依存は不要ですが、差分表示が重要な領域では専用の比較関数を用意します。

```go
func assertUser(t *testing.T, got, want User) {
	t.Helper()

	if got.ID != want.ID {
		t.Fatalf("ID: got %q, want %q", got.ID, want.ID)
	}
	if got.Email != want.Email {
		t.Fatalf("Email: got %q, want %q", got.Email, want.Email)
	}
}
```

`t.Helper()`を付けると、失敗行がヘルパー内部ではなく呼び出し元に寄ります。

## テーブルにロジックを入れすぎない

ケースごとにセットアップが大きく違うなら、テーブル駆動にこだわらないほうが読みやすいことがあります。テーブルは「同じ操作を、違う入力で確認する」ための道具です。

複数の外部依存、非同期処理、時計の差し替えが絡む場合は、テスト関数を分けて意図を明確にします。

## まとめ

テーブル駆動テストの価値は、網羅性よりも読みやすい反復にあります。ケース名を仕様として書き、比較方法を揃え、セットアップが複雑になったら無理に表へ押し込まないのが実践的です。
