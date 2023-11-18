---
title: "Swagger-mergerを使ってみた"
emoji: "🕌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["OpenAPI","Swagger", Redoc]
published: true
---

## 概要
APIのドキュメント管理でOpenAPIを採用したいが、
単純にswagger.yamlを記載する運用にすると1ファイルのサイズが肥大化して管理しにくい。
そこで、yamlファイルを分割して1ファイルのサイズを小さくし、運用しやすくする。
分割状態ではUIに表示されないので、swagger-mergerを使ってマージを行う。

## 構成
![](//images/swagger-merger/2023-11-19-00-30-18.png)

### 想定
- コンテナ起動時において、利用者がOpenAPIの管理フォルダ内のファイルを編集したときに自動でマージされる
- 利用者はlocalhost:8082でRedocにアクセスでき、閲覧できる。
  
## 設定手順
#### フォルダ構成
```
├── openapi/
│   ├── Dockerfile
│   ├── dist/
│   │   └── swagger.yaml
│   └── src/
│       ├── index.yaml
│       └── paths/
│           ├── index.yaml
│           └── metadata/
│               └── index.yaml
└── docker-compose.yml
```

#### docker-compose.ymlの用意
```
version: "3"
services:
  redoc:
    image: redocly/redoc
    container_name: "redoc"
    ports:
      - "8082:80"
    volumes:
      - ./openapi/dist:/usr/share/nginx/html/api
    environment:
      SPEC_URL: api/swagger.yaml

  swagger-merger:
    build:
      context: .
      dockerfile: ./openapi/Dockerfile
    command: >
      watch 'swagger-merger -i /swagger/src/index.yaml -o /swagger/dist/swagger.yaml' /swagger/src/
    volumes:
      - ./openapi:/swagger
```
大切なのはswager-mergerのコマンド設定です。
このコマンド宣言によってOpenAPIの管理ファルダで変更があった時にマージ処理が
自動で実行されるようになります。

#### Swagger-mergerのDockerfile作成
```
FROM node:alpine

RUN npm install -g swagger-merger watch

CMD ["swagger-merger"]
```

#### OpenAPIの用意
openapi/src/index.yaml
```
openapi: 3.0.1
info:
  title: Simple User API
  description: A simple API to retrieve user information.
  version: 1.0.0
paths:
  $ref: "./paths/index.yaml"
```

openapi/src/paths/index.yaml
```
/:
  $ref: "./metadata/index.yaml"
```

openapi/src/paths/index.yaml
```
get:
  tags:
    - metadata
  operationId: list-data-sets
  summary: test api
  responses:
    '200':
      description: test description
```

#### 表示確認
コンテナが起動していれば、
http://localhost:8082
にアクセスすれば表示されます。
起動していない場合は、起動時にもマージ処理が実行されますので、
追加の修正などは必要ありません。

## 統括
ファイル分割で利便性は向上したが、
マージファイルでないとエディターのプレビュー機能での表示確認はできなかった。
一人で管理する分ではどちらでもいいかもしれないが大きいファイルは
運用が煩雑になりがちな認識なので分割して運用したい。
チームで運用する場合は、分割されていることでコンフリクトの発生が低くなるので
余計な手間が減ると考える。
次回は、どんなフォルダ構成がいいかを考える。
