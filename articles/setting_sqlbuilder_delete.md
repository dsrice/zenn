---
title: "SQLBuilderの仕様を考える　DELETE仕様検討編"
emoji: "🖋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Go","SQLBuilder", "OSS"]
published: false
---

# 目的
GolangのSQLBuilerを作ってみたくて、仕様を考える。
今回はDELETE文の仕様を考える。また、TRUNCATEも併せて考える。

# 目指す形
ここではusersというTableに対応したUserというTableモデル構造体がある場合とする。
宣言としては、
```
CREATE TABLE users (
  id int,
  name varchar(10)
)

CREATE TABLE tokens (
  id int,
  user_id int REFERENCES users.id,
  token varchar(10)
)
```
を想定する

## DELETE

パッケージ名は仮でpkとして

```go
pk.DELETE(pk.Table("users"))

// DELETE FROM users
```

とすることを考えている。

WHERE句の利用は

```go
pk.DELETE(pk.Table("users")).WHERE(pk.Eq("users.id"), 1)

// DELETE FROM users WHERE users.id = 1
```
を考えており、WHERE句としてはSELECT文と同じ

## TRUNCATE
パッケージ名は仮でpkとして

```go
pk.TRUNCATE(pk.Table("users"))

// TRUNCATE TABLE users
```
