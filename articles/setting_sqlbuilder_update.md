---
title: "SQLBuilderの仕様を考える　INSERT仕様検討編"
emoji: "🖋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Go","SQLBuilder", "OSS"]
published: false
---

# 目的
GolangのSQLBuilerを作ってみたくて、仕様を考える。
今回はUPDATE文の仕様を考える。

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

## Update

パッケージ名は仮でpkとして

```go
pk.Update(pk.Table("users")).Set("name", "test")

// UPDATE users set name = 'test'
```

とすることを考えている。

複数カラムに関しては

```go
pk.Update(pk.Table("users")).Set("id", 2).Set("name", "test")

// UPDATE users set id = 2, name = 'test'
```

また、以下のような連想配列も許可する

```go
um := map[string]interface{}
um["id"] = 2
um["name"] = "test"

pk.Update(pk.Table("users")).SetMap(um)

// UPDATE users set id = 2, name = 'test'
```

## Where句の利用
利用としては以下を想定

```go
pk.Update(pk.Table("users")).Set("name", "test").Where(pk.Eq("id", 1))

// UPDATE users set name = 'test' where id = 1
```

また、以下のようにも利用できるようにする。

```go
user := pk.Table("users")

pk.Update(users).Set(user.Col("name"), "test").Where(pk.Eq(user.Col("id"), 1))

// UPDATE users set users.name = 'test' where users.id = 1
```

AS句が利用されても適応される想定はSELECT文の時と同じ

```go
user := pk.Table("users").AS("u")

pk.Update(users).Set(user.Col("name"), "test").Where(pk.Eq(user.Col("id"), 1))

// UPDATE users AS u set u.name = 'test' where u.id = 1
```

