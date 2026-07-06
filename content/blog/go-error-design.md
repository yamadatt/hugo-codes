---
title: Goの <span>error</span> をAPIとして設計する
title_safe: GoのerrorをAPIとして設計する
highlight: emerald
pack: duotone
icon: bug
draft: false
toc: true
comments: false
date: 2026-07-05
category: Go
tags:
  - go
  - errors
  - api
---
Goの`error`は単なる失敗メッセージではなく、呼び出し元に判断材料を返すAPIです。ログに残す文字列と、分岐に使う意味を分けて考えると、`errors.Is`や`errors.As`の使いどころがはっきりします。

<!--more-->

## 分岐させたい失敗だけ公開する

呼び出し元が再試行、404変換、入力エラー表示のような判断をするなら、比較可能なエラーを公開します。

```go
var ErrUserNotFound = errors.New("user not found")

func (r *UserRepository) Find(ctx context.Context, id string) (*User, error) {
	row := r.db.QueryRowContext(ctx, `select id, name from users where id = ?`, id)

	var user User
	if err := row.Scan(&user.ID, &user.Name); err != nil {
		if errors.Is(err, sql.ErrNoRows) {
			return nil, ErrUserNotFound
		}
		return nil, fmt.Errorf("scan user %s: %w", id, err)
	}

	return &user, nil
}
```

ここで`sql.ErrNoRows`をそのまま外へ漏らすと、保存先の詳細が上位層に染み出します。アプリケーションとしての意味に変換してから返します。

## ラップは原因を残したいときに使う

`fmt.Errorf("...: %w", err)`でラップすると、文脈を足しつつ`errors.Is`や`errors.As`で原因をたどれます。

```go
func loadConfig(path string) (*Config, error) {
	b, err := os.ReadFile(path)
	if err != nil {
		return nil, fmt.Errorf("read config %q: %w", path, err)
	}

	var cfg Config
	if err := yaml.Unmarshal(b, &cfg); err != nil {
		return nil, fmt.Errorf("parse config %q: %w", path, err)
	}
	return &cfg, nil
}
```

文字列だけを結合すると、機械的な判定ができなくなります。運用ログに必要な文脈を足しながら、プログラムが扱える原因も残すのが基本です。

## 型付きエラーは追加情報が必要なときだけ

HTTPステータスやフィールド名など、呼び出し元が構造化された情報を必要とするなら型を定義します。

```go
type ValidationError struct {
	Field string
	Msg   string
}

func (e *ValidationError) Error() string {
	return e.Field + ": " + e.Msg
}

func statusCode(err error) int {
	var v *ValidationError
	if errors.As(err, &v) {
		return http.StatusBadRequest
	}
	if errors.Is(err, ErrUserNotFound) {
		return http.StatusNotFound
	}
	return http.StatusInternalServerError
}
```

型を増やすほどAPI面は広がります。分岐に必要な情報がないなら、無理に型を作らずラップで十分です。

## まとめ

公開するエラーは、呼び出し元に約束する契約です。分岐させたい失敗だけを明示し、内部実装のエラーは意味に変換し、原因を残したいところでは`%w`でラップします。
