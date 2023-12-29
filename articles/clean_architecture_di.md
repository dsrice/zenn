---
title: "クリーンアーキテクチャについて考える　DI編"
emoji: "🕌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["CleanArchitecture","Architecture", "Golang", "dig"]
published: false
---

## 目的

GolangのAPIのみのバックエンドプログラムを考えるときのクリーンアーキテクチャ実現のための依存性の注入をdigを使って実現する。

## 要約

クリーンアーキテクチャの実現において依存性の注入は必須になる。
Golangにおいて依存性の注入には有名な手法として[wire](https://github.com/google/wire)がある。
とても有名な手段であるのだが、現在更新が止まっている（最終が2021年）。
使っている状態であったとしてもこれだけ更新が止まってしまっていると継続利用に躊躇するレベルだと思っている。
今回は、[dig](https://github.com/uber-go/dig)を使って依存性の注入を行います。

## digのメリットとデメリット

digはDIコンテナを提供するモジュールとなっており、コンストラクタをコンテナに登録することで必要なインスタンスを提供します。
メリットとしては、引数などの情報なしにコンストラクタを登録するのみなので依存関係の解決が柔軟になっています。
デメリットとしては、暗黙的に依存関係を解決しようとするため依存関係を解決できなかったときにエラーになります。ですが、エラーはインスタンス取得時のため同じ場所で様々なエラーが発生してしまい、Goっぽいくない挙動をしてしまします。

## 依存性の注入

そもそも依存性の注入とはInterfaceに具象を入れることである。
もう少しわかりやすい表現をするならInterfaceの宣言に対して中身を定義することになります。
Golangではモジュールなどを使わなくても以下のような記載をします

```　golang
type UserInterface interface {
  GetUser() error
}

type User struct{
  UserID string
}

func (u *User) GetUser() error {
  省略
}

func NewUser() UserInterface {
  return &User{}
}
```

なくても運用することは可能です。その場合、使われ方をレビューなどでチェックする必要があります。個人的な感覚ではありますが、この内容をレビューする運用は結構大変だと思っています。レビュー側への負担が重すぎて何かしらの仕組みを使った依存性の注入の実現が必要だと思っています。

## クリーンアーキテクチャと依存性の注入

クリーンアーキテクチャでは制御方向に対して依存関係を反転させてレイヤーの外側から内側へのみに依存する関係を作りだすことを可能としています。
今回考えているアプリケーションでは

- Controller
- Usecase
- Repository

の３つが登場します。
依存関係としては

- Controller
  - Usecaseを呼び出します
- Usecase
  - 別のUsecaseを呼び出すこともありえる
  - Repositoryを呼び出す
- Repository
  - 別のRepositoryを呼び出すこともある
  - Connectionの持ち回りがある。

といった関係にある。
例として、
とあるControllerを宣言するのにUsecaseを１つ宣言する必要があるとし、
そのUsecaseの宣言に１つのRepositoryの宣言が必要があるケースを考えると
これらの構造体内にする宣言はすべてInterfaceであるため、宣言時に依存性の注入を行う必要があります。
この依存性注入は、最下層にあるRepositoryから行っていく必要もあるため、
Controller１つを宣言するだけRepositoryとUsercaseの宣言が必要になり、かなりの労力になってしまします。
Interfaceの特性的にも関数として用意するのも限界があるので悩ましい問題ではあります。
この悩ましい問題をDIモジュールである[dig](https://github.com/uber-go/dig)を使って労力を減らすことを考えます。

## 前提

今回、WebアプリケーションのフレームワークにEchoを使うものとしています。
digはこういったアプリケーションのフレームワークを使う場合に向いているモジュールです。
フレームワークを使わない場合は、[fx](https://github.com/uber-go/fx)というモジュールを使うことをお勧めします。
digとかなり使い勝手が近いのですが、アプリケーションの起動までもこちらに任せる形になります。

## digの使い方

簡単な使い方は以下のようになる。

``` golang
func main(){
  c := dig.New()
  c.Provide(server.NewServer)

  err := c.Invoke(func(s *server.Server) {
    s.Start()
  })

  if err != nil{
    panic(err)
  }
}
```

### 1行目 c := dig.New()

DIモジュールにはよくあるコンテナの宣言になります。

### 2行目 c.Provide(server.NewServer)

DIコンテナにあるProvideはコンストラクタをコンテナに登録します。
serverはアプリケーション（今回だとEcho）の起動宣言など入っているパッケージになっており、Serverという構造体を用意しています。
SereverはStart()関数を持っており、こちらがEchoの初期設定から起動までの処理を用意しています。

### 4行目　err := c.Invoke(func(){})

インスタンスを取得して関数を実行しています。
DIコンテナはあくまでの依存性の注入の関係を宣言しているだけなので、どこかできっかけ（最初の宣言）が必要になります。
今回は、Start()はEchoの起動まで含まれているのでこちらを実行してWebアプリケーションを実行しています。

## クリーンアーキテクチャとEchoとdig

ここまでパズルのピースとして独立しているので融合を考えます。
状況を整理すると

- Echo
  - ルーティングにControllerの宣言が必要のため、Server構造体にControllerの宣言が必要
- dig
  - Controller, Usecase, Repositoryの依存関係の宣言が必要
  - Invoke関数でアプリケーションが起草するまでに依存関係の宣言が終わっている必要がある。

先ほどの簡単なdigの使い方だとmain関数にどんどん詰め込む形になってしますのでパッケージを分けていったほうが運用面では簡潔になる。
また、main関数に追記する形は、実行時にはかならず通る処理なのでmain関数が修正されることはアプリケーションすべてに影響がある可能性を疑う作りになってしまいます。
内容が依存関係の注入なのでEPの追加でも影響があるし、Repositoryも含まれているのでTable追加でも影響を受けるので、改修頻度が多いのにメインの処理に直接手を入れるという現象が起きているので、テストなどの確認作業においては大きな影響があるかというそうではない。
コードの改修内容に対して、テスト内容が伴わない実態が生まれてしまうので個人的にはお勧めしない。DI周りは別パッケージにして処理をまとめる仕組みを提供する。

### フォルダ構成

フォルダ構成は以下のようにすることした。
DIの宣言に関しては非常にどこにあるべきか悩ましい問題ではあるが、一番外に設けた。

``` golang
.
└── app/
    ├── consts
    ├── controllers/
    │   └── ci
    ├── di/
    │   ├── controller.go
    │   ├── di.go
    │   ├── usecase.go
    │   └── repository.go
    ├── entities
    ├── infra/
    │   ├── database
    │   ├── logger
    │   └── server/
    │       └── server.go
    ├── repositories/
    │   └── ri
    └── usecases/
        └── ui        
```

### diパッケージ

diパッケージにあるファイルは

- di.go
  - メインの処理を記載するファイル
  - 一度用意した場合、このファイルは触らない想定
- controller.go
  - Controllerの依存関係を宣言するファイル
- usecase.go
  - Usecaseの依存関係を宣言するファイル
- repository.go
  - Repositoryの依存関係を宣言するファイル

としました。

#### di.go

``` golang:dig.go
package di

import (
  "app/infra/database/connection"
  "app/infra/server"
  "go.uber.org/dig"
)

func BuildContainer(c *dig.Container) {
  setProvide(c, server.NewServer)
  setProvide(c, connection.NewConnection)
  provideController(c)
  provideUsecase(c)
  provideRepository(c)
}

func setProvide(c *dig.Container, i interface{}) {
  if err := c.Provide(i); err != nil {
    log.Fatal(err.Error())
  }
}

```

- BuildContainer
  - DIコンテナ作成時に呼び出すメイン関数になります。
  - DIコンテナ作成後、この処理を呼び出すだけで依存関係の宣言が終わる想定です。
- setProvide
  - Provide関数は本来返り値にerrorを持っている。毎回このエラーハンドリングを記載することはコード量も増えてしまうのでその負担を減らすために処理を用意した。
  - 依存性宣言する際は常にこの関数を利用する。

#### controller.go

``` golang: controller.go
package di

import (
  "app/controllers"
  "go.uber.org/dig"
)

func provideController(c *dig.Container) {
  setProvide(c, controllers.NewLoginController)
}
```

- provideController
  - Controllerの依存関係を記載する関数
  - 各Controllerには注入用の関数を用意しておき、DIコンテナに関数を注入していく

Usecase,Repositoryは同様の宣言をするので省略します。

### serverパッケージ

Echoの起動までの宣言をまとめているパッケージ

#### server.go

```golang: server.go
package server

import (
  "app/controllers/ci"
  "github.com/labstack/echo/v4"
  "go.uber.org/dig"
)

type Server struct {
  echo  *echo.Echo
  Login ci.LoginController
}

type inServer struct {
  dig.In
  Login ci.LoginController
}

func NewServer(s inServer) *Server {
  return &Server{
    Login: s.Login,
  }
}

func (s *Server) Start() {
  s.echo = echo.New()
  s.routing()

  s.echo.Logger.Fatal(s.echo.Start(":1323"))
}

func (s *Server) routing() {
  s.echo.GET("/", s.Login.Get)
}
```

ポイントになるのはServer構造体とinServer構造体です。
見るからに無駄な感じしかない宣言なのだが、Echoと同居するには必要な形になっている。
将来的にコード量を減らすためにdig.Inを採用しているためにこの形になっています。
今は１つのControllerしかないので、恩恵がありませんが、仮で５つのControllerがあるケースを考えます。
dig.Inを使わない場合は、

``` golang
package server

import (
  "app/controllers/ci"
  "github.com/labstack/echo/v4"
  "go.uber.org/dig"
)

type Server struct {
  echo  *echo.Echo
  con1 ci.Con1Controller
  con2 ci.Con2Controller
  con3 ci.Con3Controller
  con4 ci.Con4Controller
  con5 ci.Con5Controller
}

func NewServer(
  c1 ci.Con1Controller,
  c2 ci.Con2Controller,
  c3 ci.Con3Controller,
  c4 ci.Con4Controller,
  c5 ci.Con5Controller,
) *Server {
  return &Server{
    con1: c1,
    con2: c2,
    con3: c3,
    con4: c4,
    con5: c5,
  }
}

func (s *Server) Start() {
  s.echo = echo.New()
  s.routing()

  s.echo.Logger.Fatal(s.echo.Start(":1323"))
}

func (s *Server) routing() {
  s.echo.GET("/", s.Login.Get)
}
```

dig.Inを使う場合は

``` golang
package server

import (
  "app/controllers/ci"
  "github.com/labstack/echo/v4"
  "go.uber.org/dig"
)

type Server struct {
  echo  *echo.Echo
  con1 ci.Con1Controller
  con2 ci.Con2Controller
  con3 ci.Con3Controller
  con4 ci.Con4Controller
  con5 ci.Con5Controller
}

type inServer struct {
  dig.In
  con1 ci.Con1Controller
  con2 ci.Con2Controller
  con3 ci.Con3Controller
  con4 ci.Con4Controller
  con5 ci.Con5Controller
}

func NewServer(s inServer) *Server {
  return &Server{
    con1: s.con1,
    con2: s.con2,
    con3: s.con3,
    con4: s.con4,
    con5: s.con5,
  }
}

func (s *Server) Start() {
  s.echo = echo.New()
  s.routing()

  s.echo.Logger.Fatal(s.echo.Start(":1323"))
}

func (s *Server) routing() {
  s.echo.GET("/", s.Login.Get)
}
```

NewServer関数に大きな違いが生まれるようなっています。
違いがあるかといわれると非常に難しい問題ではあると思います。
inServer構造体の宣言があるので実際のコーディング量はほぼ同じです。
判断ポイントとしては、NewServer関数の簡潔さにあると思います。
今回のケースではControllerが増えるたびに引数が増えていくのでNewServerの実態がわかりにくくなっていきます。こちらを嫌う場合はdig.Inを使ったほうが、読みやすいコーディングだと思います。

### main.go

用意したパッケージの実装を利用するとmain.goは以下のようになります。

``` golang
package main

import (
  "app/infra/di"
  "app/infra/server"
  "go.uber.org/dig"
)

func main() {
  c := dig.New()

  di.BuildContainer(c)
  err := c.Invoke(func(s *server.Server) {
    s.Start()
  })

  if err != nil {
    panic(err)
  }
}
```

実現したかった、main.goの毎回の修正はこの実装によってなくなりました。
Controllerが増えたら、diパッケージのcontroller.goを修正する。
追加したControllerが正しく使えているかは当然のテスト対象ですが、アプリケーション自体の起動に関しては影響を与えにくくなっています。

## 最後に

今回は、Echoを使いながらdigを使う方法を記事にしました。
最適解なのかは、個人的にはまだまだ追及してもいいかもしれないと思っています。
digにはほかにも機能があるのでdigをもっと調べて検討する価値があると感じています。
手法の１つとして実現させてもの程度に読んでいただけたらと思います。
（もっといい方法があるよ！は募集します！！！）
