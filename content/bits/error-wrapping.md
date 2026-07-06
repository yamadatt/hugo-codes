---
type: blurb
date: 2026-07-05T18:00:00+09:00
tags:
  - go
  - errors
---

`fmt.Errorf("read config: %w", err)`の文脈はログのため、`%w`は呼び出し元の分岐のため。両方の読み手を意識してラップします。
