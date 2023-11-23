---
title: "OpenAPIの分割ファイルの構成を考えてみた"
emoji: "🕌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["OpenAPI","Swagger", Redoc]
published: true
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
        │   ├── component.yaml
        │   ├── parameters/
        │   │   └── parameter.yaml
        │   ├── requestBodies/
        │   │   ├── requestBody.yaml
        │   │   └── login_post.yaml
        │   ├── responses/
        │   │   ├── response.yaml
        │   │   ├── 404.yaml
        │   │   └── 500.yaml
        │   └── schemas/
        │       ├── auth.yaml
        │       ├── internalError.yaml
        │       ├── notFoundError.yaml
        │       └── schema.yaml
        └── paths/
            ├── path.yaml
            └── login/
                ├── login.yaml
                └── post/
                    ├── login_post.yaml
                    └── login_post_200.yaml
```

#### ルール
- 同じファイル名は基本的に避ける
  - 現状endpointとrequestBodyの関連付けがわかるように同名ファイルにしている。
- 要素単位で分割するようにする。
- pathsは以下の構成とする
  EP > method
- REST構成想定なので同じEPでもmethodがことなるケースが存在するのでこの方法を採用した。
- methodフォルダ内はリクエストとレスポンスを分ける。
  - レスポンスはステータスコード別にファイルを用意する。
  - 共通エラーに関してはcomponentsに定義する。

#### 各フォルダ説明
##### distファルダ
マージ後のyamlファイル置き場にしているので、人の手で編集することはない想定
VS Codeなどでマージ後の内容を確認したい場合はこのファイルに対して表示を確認すればよい。

##### srcフォルダ
実装フォルダにしている。
OpenAPIのyamlファイルを3分割している。
- index.yaml
  - 一番のベースファイルになる
  - infoの情報などは分割せずに記載しているのでVersion変更時などは編集が必要
- pathsフォルダ
  - paths要素をまとめているフォルダ
- componentsフォルダ
  - components要素をまとめているフォルダ

###### pathsフォルダ
paths要素を記載する。
- path.yaml
  - pathsの管理ファイルになっている。
  - エンドポイント追加時はまずこのファイルに追加する。
- エンドポイントフォルダ
  - メソッドフォルダ
    - 今回はRESTを採用しているためこのようなフォルダ構成を採用している。
    - RESTでないならエンドポイントフォルダでもよいと考えている。
    - メソッドフォルダ内には、リクエストとレスポンスのファイルは分けるようにしている。

###### componentsフォルダ
components要素を記載する
- component.yaml
  - componentsの管理ファイル
  - 追加でcomponentsの要素を追加する場合は編集が必要
- parametersフォルダ
  - parametersの管理フォルダだが、配下にはparameter.yamlのみを置くようにしている。
  - parametersにはheaderなど共通に使用するparameterを記載している。
  - そのため重複宣言の回避のため1ファイル運用としている
- requestBodiesフォルダ
  - Json形式を想定しているのでEP毎のrequestBodyをまとめている。
  - 1ファイルで運用すべきかは悩む問題点ではあるが、requestBodyの種類ごとにファイルを作成する運用としている。
    - 悩んでいる背景としては、例えばユーザーIDに対応するkey値はどこrequestBodyでも「user_id」としたいときに共通にするためにはどの運用が保守しやすいかという点である。
    - このあたりは微妙に手が届かない要素な気がするので命名規則でカバーしていくしかないと考えている。
- responsesフォルダ
  - responsesの管理フォルダだが、共通のresponseのみ管理している。
  - 該当するのはエラー時のレスポンスが主な管理範囲。
  - そのため、ファイル名にステータスコードを採用している。
  - EP固有のエラーに関してはここでは管理しない想定としている。
- schemasフォルダ
  - schemaの管理フォルダ
  - 現在は、schema毎にファイル作成する方針ではあるが、運用してみてルールを変更する可能性がある。
  - EPにはタグをつける予定ではあるので、タグごとにファイル分けをすることで同じ種類のschemaが1ファイルにまとめることで、key値の命名が異なることを避けやすくなるのでは？と考えているため。
  - 考えすぎな気もするのである程度運用してから見直す。
  

## 総括
一旦今考えられる運用を想定してフォルダ分けを実施してみた。
いろいろ作るうえで運用しづらい点などがあったら構成も含めて見直していきたいと考える。
合間みて作っていくものだから半年後とかかな・・・・

