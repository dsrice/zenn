---
title: "SQLBuilderの仕様を考える　SELECT仕様検討編"
emoji: "🖋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Go","SQLBuilder", "OSS"]
published: false
---

## 目的

GolangのSQLBuilderを作ってみたくて、仕様を考える。
今回はSELECT文の仕様を考えるが、サブクエリまで考えると複雑になるので
一旦保留にする

## 目指す形

ここではusersというTableに対応したUserというTableモデル構造体がある場合とする。
宣言としては、

```sql
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

## Select

パッケージ名は仮でpkとして

```go
pk.Select().From(pk.Table("users"))

// SELECT * FROM users;
```

とすることを考えている。

カラムを限定する場合は

```go
pk.Select("id").From(pk.Table("users"))

// SELECT id FROM users;
```

### Where句

Where句は例ととして挙げると

```go
pk.Select()
  .From(pk.Table("users"))
  .Where(
    pk.Eq("id", 1)
  )

// SELECT * FROM users WHERE id = 1
```

Where句に複数条件がある場合は

```go
pk.Select()
  .From(pk.Table("users"))
  .Where(
    pk.Gt("id", 1)
    .AND(pk.Eq("name", "test"))
  )

// SELECT * FROM users WHERE id > 1 AND name = 'test'
```

Where句でOR条件を利用する場合は、

```go
pk.Select()
  .From(pk.Table("users"))
  .Where(
    pk.Gt("id", 1)
    .AND(
      pk.Eq("name", "test")
      .OR(pk.Eq("name", "test2"))
    )
  )

// SELECT id, name FROM users WHERE id > 1 AND (name = 'test' OR name = "test2")
```

### Inner Join

Inner Joinを使う場合は、

```go
pk.Select(
    "users.id",
    "users.name",
    "tokens.token",
  )
  .From(pk.Table("users"))
  .InnerJoin(pk.Table("tokens"), pk.Eq("tokens.user_id", "users.id"))
  

/*
  SELECT 
    users.id,
    users.name,
    tokens.token
  FROM users
  INNER JOIN tokens on
    tokens.user_id = users.id
*/
```

同じSQLになるが、次の書き方もできるようにする

```go
u := pk.Table("users")
t := pk.Table("tokens")

pk.Select(
    u.Col("id"),
    u.Col("name"),
    t.Col("token")
  )
  .From(u)
  .InnerJoin(t, k.Eq(t.Col("user_id"), u.Col("id")))

/*
  SELECT 
    users.id,
    users.name,
    tokens.token
  FROM users
  INNER JOIN tokens on
    tokens.user_id = users.id
*/
```

一見無駄な労力に見えるが、結合が長くなったり、Table名が長いときに、
AS句を利用して別名を割り当てることは珍しくないので便利な形としては
Table宣言時にAS句の指定をいれることで後の取り回しをよくすることを狙っている。

```go
u := pk.Table("users").As("u")
t := pk.Table("tokens").As("t")

pk.Select(
    u.Col("id"),
    u.Col("name"),
    t.Col("token")
  )
  .From(u)
  .InnerJoin(t, k.Eq(t.Col("user_id"), u.Col("id")))

/*
  SELECT 
    u.id,
    u.name,
    t.token
  FROM users AS u
  INNER JOIN tokens AS t on
    t.user_id = u.id
*/
```

最初の書き方では、Tableに付けられた別名が結合条件時には不明なので
記載時に別名で書く必要がでてくる。

### Joinに関して

SQLのJoinの種類に応じて

- INNER JOIN
  - InnerJoin()
- LEFT (OUTER) JOIN
  - LeftJoin()
- RIGHT (OUTER) JOIN
  - RightJoin()
- FULL (OUTER) JOIN
  - FullJoin()
- CROSS JOIN
  - CrossJoin()
  - CROSS JOINは結合条件が不要なので設定できないようにする。

で用意する。
利用し方は基本同じとするため詳細はここでは省略する。

### Order by

Order byの利用は以下を想定している。
デフォルトはASCでの並び替えを想定している。

```go
pk.Select()
  .From(pk.Table("users")).
  Order("id")

pk.Select()
  .From(pk.Table("users")).
  Order("id").ASC()

// SELECT * FROM users ORDER BY id ASC
```

DESCで並べたい場合は、

```go
pk.Select()
  .From(pk.Table("users")).
  Order("id").DESC()

// SELECT * FROM users ORDER BY id DESC
```

一見不便そうなのだが、一覧表示で降べき、昇べきが変更できたりするので
この仕様でも需要がある気と考える。
面倒なケースも当然ありそうなので短縮も用意する。

```go
pk.Select()
  .From(pk.Table("users")).
  OrderA("id")

// SELECT * FROM users ORDER BY id ASC

pk.Select()
  .From(pk.Table("users")).
  OrderDe("id")

// SELECT * FROM users ORDER BY id DESC
```

複数宣言に関しては、

```go
pk.Select()
  .From(pk.Table("users")).
  Order("id", "name").ASC()

// SELECT * FROM users ORDER BY id, name ASC
```

を想定、並び順がそのまま指定順となることを考えている。
複雑なケースは以下の利用を想定

```go
pk.Select()
  .From(pk.Table("users")).
  Order("id").ASC().Order("name").Desc()

pk.Select()
  .From(pk.Table("users")).
  OrderA("id").OrderDe("name")

// SELECT * FROM users ORDER BY id　ASC, name DESC
```

Table名指定も有効とすることも考えており

```go
u := pk.Table("users").As("u")
t := pk.Table("tokens").As("t")

pk.Select(
    u.Col("id"),
    u.Col("name"),
    t.Col("token")
  )
  .From(u)
  .InnerJoin(t, k.Eq(t.Col("user_id"), u.Col("id")))
  .Order(u.Col("name"), t.Col("token"))

/*
  SELECT 
    u.id,
    u.name,
    t.token
  FROM users AS u
  INNER JOIN tokens AS t on
    t.user_id = u.id
  ORDER BY u.name, t.token
*/
```

とテーブル名のAS句利用への対応は行う。
しかし、今回カラム名の継承は見送る。
見送るSQLは以下のパターン

```sql
SELECT id AS i, name FROM users ORDER BY i ASC
```

### Limit

Limitの利用は以下を想定している。
Limitのデフォルト数値は設けなず、指定を必須とする。

```go
pk.Select()
  .From(pk.Table("users")).
  Limit(5)


// SELECT * FROM users LIMIT 5
```

### Offset

Offsetの利用は以下を想定している。
Offsetのデフォルト数値は設けず、指定を必須とする。

```go
pk.Select()
  .From(pk.Table("users")).
  Offset(5)

// SELECT * FROM users OFFSET 5
```

### Group By

Group byの利用は以下を想定している。

```go
pk.Select("name")
  .From(pk.Table("users")).
  GroupBy("name")

// SELECT name FROM users GROUP BY name
```

複数の場合は、

```go
pk.Select("id", "name")
  .From(pk.Table("users")).
  GroupBy("name", "id")

// SELECT id, name FROM users GROUP BY name, id
```

となることを想定しており、宣言順がグルーピングする順番に反映される。
Table名指定も有効とすることも考えており

```go
u := pk.Table("users").As("u")
t := pk.Table("tokens").As("t")

pk.Select(
    u.Col("name"),
    t.Col("token")
  )
  .From(u)
  .InnerJoin(t, k.Eq(t.Col("user_id"), u.Col("id")))
  .GroupBy(u.Col("name"), t.Col("token"))

/*
  SELECT 
    u.id,
    u.name,
    t.token
  FROM users AS u
  INNER JOIN tokens AS t on
    t.user_id = u.id
  GROUP BY u.name, t.token
*/
```

### Having

Havingの利用は以下を想定している。

```go
pk.Select("name")
  .From(pk.Table("users")).
  Group("name").
  Having(pk.Like("name", "%t%"))

// SELECT name FROM users GROUP BY name HAVING name LIKE '%t%'
```

HavingがGroup Byがなくてもつかえるので

```go
pk.Select()
  .From(pk.Table("users")).
  Having(pk.Like("name", "%t%"))

// SELECT * FROM users HAVING name LIKE '%t%'
```

も利用可能とする。

## Expressions

Expressionsとして不等号などの表現を用意する。

- Eq(A, B)
  - 等号で A = Bとなる。
  - 文字列の場合はSQL発行時に''で囲む
- Neq(A, B)
  - 等号で A != Bとなる。
  - 文字列の場合はSQL発行時に''で囲む
- Gt(A, B)
  - A > Bとなる
  - Bの文字列は利用可能とする
- Gte(A, B)
  - A ≧ Bとなる
  - Bの文字列は利用可能とする
- Lt(A, B)
  - A <> Bとなる
  - Bの文字列は利用可能とする
- Lte(A, B)
  - A ≦ Bとなる
  - Bの文字列は利用可能とする
- LIKE(A, B)
  - A LIKE Bとなる
  - Bに関してはLIKE検索のための%は補填しない
- NliKE(A, B)
  - A NOT LIKE Bとなる
  - Bに関してはLIKE検索のための%は補填しない
- Pm(A, B)
  - A LIKE 'B%' となる
  - 前方一致するための%はSQL発行時に補填する
- Npm(A, B)
  - A NOT LIKE 'B%' となる
  - 前方一致するための%はSQL発行時に補填する
- Sm(A, B)
  - A LIKE '%B'　となる
  - 後方一致するための%はSQL発行時に補填する
- Nsm(A, B)
  - A NOT LIKE '%B'　となる
  - 後方一致するための%はSQL発行時に補填する
- Psm(A, B)
  - A LIKE '%B%'　となる
  - 必要な％はSQL発行時に補填する
- Npsm(A, B)
  - A NOT LIKE '%B%'　となる
  - 必要な％はSQL発行時に補填する
- Between(A, B , C)
  - A BETWEEN B TO C となる
- Nbetween(A, B , C)
  - A NOT BETWEEN B TO C となる
- In(A, B)
  - A IN (B) となる
  - Bに関してはSliceでも、複数宣言も可とする。
- Nin(A, B)
  - A NOT IN (B) となる
  - Bに関してはSliceでも、複数宣言も可とする。
- IsNull(A)
  - A IS NULL となる
- IsNotNull(A)
  - A IS NOT NULL となる
- IsTrue(A)
  - A = true　となる
- IsNotTrue(A)
  - A != true となる
- IsFalse(A)
  - A = false　となる
- IsNotFalse(A)
  - A != false となる
