---
title: "OpenAPIの分割ファイルの構成を考えてみた"
emoji: "🕌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["OpenAPI","Swagger", Redoc]
published: false
---

## 概要
APIのドキュメントをOpenAPIを使って管理するために分割できるようにしたが、
フォルダ構成を決めないと運用できない。
今回は、使いやすいと思われる構成を考える。

#### 前提条件
今回実装しようと考えているAPIはREST構成を考えている。
その点を考慮した構成とする。

## 構成
```
.
└── openapi/
    ├── dist/
    │   └── swagger.yaml
    └── src/
        ├── index.yaml
        ├── components/
        │   ├── components.yaml
        │   ├── parameters/
        │   │   └── parameter.yaml
        │   ├── responses/
        │   │   └── notFound.yaml
        │   └── schemas/
        │       └── auth.yaml
        └── paths/
            ├── paths.yaml
            └── login/
                ├── login.yaml
                └── post/
                    ├── login_post.yaml
                    └── login_post_200.yaml
```

#### ルール
- 同じファイル名は基本的に避ける
- 要素単位で分割するようにする。
- pathsは以下の構成とする
  EP > method
- REST構成想定なので同じEPでもmethodがことなるケースが存在するのでこの方法を採用した。
- methodフォルダ内はリクエストとレスポンスを分ける。
  - レスポンスはステータスコード別にファイルを用意する。
  - 共通エラーに関してはcomponentsに定義する。
- componentsに共通要素を定義することが重要になる
  - 特にparametersを活用することが非常に重要な要素になる。
  - parametersは重複定義を回避するために1ファイルのみとする。


