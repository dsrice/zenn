---
title: "SQLBuilderの仕様を考える　目標設定編"
emoji: "🕌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Go","SQLBuilder", "OSS"]
published: false
---

# 目的
GolangのSQLBuilerを作ってみたくて、仕様を考える。
最終系をみつつ、Version1にするゴールを考える

# なんでやってみたいのか
GolangにもSQLBuilderはあるが、unionに対応していないことが多い。（もしかしたらやり方を知らないだけかも）
現在業務では、unionを使うことが非常に多いのでどうするべきかに困っていた。
それならやってみようと思ったことと、OSS活動にもならないかと思ってやってみようと思い立った。

# 目標
主要な手段である
- Select
- insert
- Update
- Delete(truncateを含める)
に対応できるようにしたい。
最初は特定のコマンドを実行することでSQL文が取得できる形を目指す。
他のORMなどの連携に関しては形ができたときに検討することとして一旦は保留とする。
最初はMySQL対応をベースにする。
Postgreseに関しては別途対応を考えるものとする。

## Select
Select文では主要な機能をあげると
- Where句
- Order By
- Limit
- Offset
- Distinct
- Group By
- Having
- Join
- Union(Union ALLも含む)
の対応を目指す。

## Insert
InsertはBuik Insertまでできることを目指す。

## Update
UpdateはWhere句による対処ウ絞り込みまでできることを目指す。

## Delete
DeleteはWhere句による対象絞り込みまでできることを目指す
TruncateはTable指定でできるようにする。

## Function
SQL関数に関しては、今回実装しない。
これは風呂敷を広げすぎないための対応でVersion2での対応を考えている。