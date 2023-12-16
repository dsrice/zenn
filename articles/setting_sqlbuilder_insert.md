---
title: "SQLBuilderの仕様を考える　INSERT仕様検討編"
emoji: "🖋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Go","SQLBuilder", "OSS"]
published: false
---

# 目的
GolangのSQLBuilerを作ってみたくて、仕様を考える。
今回はINSERT文の仕様を考えるが、SELECT+INSERTまで考えると複雑になるので
一旦保留にする

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

## Insert

パッケージ名は仮でpkとして

```go
pk.Insert().Into(pk.Table("users")).Value(1, "test")

// INSERT INTO users VALUES(1, 'test')
```

とすることを考えている。

カラムを限定する場合は

```go
pk.Insert("id").Into(pk.Table("users")).Value(1)

// INSERT INTO users(id) VALUES(1)
```

## バルクインサートの実現に向けて
バルクインサートのためにValuesを何で受け取るべきかを考える。
現在、モデルを定義していないので構造体では難しい問題がある。
またSliceで受け取る場合もデータ型の問題でユーザに難しい作成を強いることになる。

#### 案１
Valueメソッドを連結するパターンとして
```go
pk.Insert("id").Into(pk.Table("users")).Value(1).Value(2)

// INSERT INTO users(id) VALUES(1), (2)
```

をあげられる。ユーザにfor文を強制してしまうが手法の１つしてはありだと考えているので
一応採用する。

#### 案２（保留案）
pk.TableのSliceで受け取るパターン
手間は必要になるが、案１との違いが重要になる。

まずはテーブルに対してのカラム宣言
```go
user := pk.Table("users")

user.SetValue("id", 1)
user.SetValue("name", "test")
```

ここから

```go
pk.Insert().Into(pk.Table("users")).Value(user)

// INSERT INTO users VALUES(1, 'name')
```

といった感じ・・・・
使わない気がするな。
なぜなら、サービスでの利用を考えたときになにかしら構造体でもっていることが一般的。
これはその構造体からの変換処理を求めているので、
こんな面倒なことをせずに案１を使えばいいだけな感覚。
それだったら構造体にタグ設定したほうが扱いが簡単なので、
案２に関しては現時点では保留とする。モデルの対応自体を保留にしている状況でここで定義してしまうと全体適応時に弊害になる懸念があるため。
