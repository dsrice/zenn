---
title: "クリーンアーキテクチャについて考える　自動生成編"
emoji: "🕌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["CleanArchitecture", "Golang", "dig", "jennifer", "echo"]
published: false
---

## 目的

フレームワークにEchoを使いながら、クリーンアーキテクチャの実現を簡易的に目指すためにdigを使った依存性逆転まで実現させる方法まで考えた。しかし、単純に開発者が守るべきルールとしては決まった作業が多く感じる。
そこで、Golangには自動生成する文化があるのでデフォルトの部分は自動生成して開発者が実装部分に注力しやすい環境を用意する。

## 実現に向けて

今回はあくまでも自動生成にのみできることを目指す。作成済みのファイルをコマンド１つで修正できるといったことは目指さない。

自動生成は[Jennifer](https://github.com/dave/jennifer)を使います。
また、作成にあたりdigの取りまわしも考えます。なぜ考えるのかは自動生成で作成したあとの取りまわしをよくするためです。詳細は後述させていただきます。

## Jennifer

### 採用の経緯

JenniferはGolangのための自動生成モジュールになります。
そもそもGolangにはgo generateという自動生成コマンドが存在しています。
生成コードの管理が大変そうな印象を受けて、避けてJenniferを見つけてきました。
また、今回自動生成で補いたかった内容がInterfaceとその注入先の関係を一定にしたかったという点です。詳細に箇条書きすると

- 注入関数はNewから始めたい。
- Interfaceはインターフェイスとわかる名称をつけないが、注入するほうは末尾にImpをつけたい
- Interfaceと注入側は別のフォルダにしたい

の3点です。
Jenniferならこの3点に関してそれほど複雑な対応が求められていないので実装が大変ではありませんでした。ただ、生成される内容がコードを読まなければいけないという点に関してはデメリットではあります。ですが、生成されるコードが単純なものなので大きなマイナスではないとう認識で採用に至っています。

### 使ってみる

試してみるステージにはあるモジュールではあるので実際に使ってみました。
インストールはこちらのコマンド

``` golang
go get -u github.com/dave/jennifer/jen
```

自動作成時にgo runコマンドを利用するのでアプリケーションとは別のmainファイルを用意します。

``` golang
package main

import (
    "fmt"

    "github.com/dave/jennifer/jen"
)

func main() {
  f := jen.NewFile("ci")

  f.ImportName("app/usecases/ui", "ui")

  f.Type().Id("testGenerateImp").Struct()

  f.Func().Id("NewTestGenerate").Params().Qual(
    "app/controllers/ci", "TestGenerate"
  ).Block(
    jen.Return(jen.Op("&").Id("testGenerateImp").Values()),
  )

  fmt.Printf("%#v", f)
}
```

main関数の説明をしいくと

- 1行目　f:= jen.NewFile("ci")
  - 紛らわしい関数名なのですが、ファイル宣言なのですが、引数に入る文字列はパッケージ名になります。
- 3行目 f.ImportName("app/usecases/ui", "ui")
  - インポート宣言なのですが、書かなくてもjenniferが保管してくれます。
  - インポートに対してエイリアスを指定したい場合は必ず利用しましょう。
  - 保管してくれた場合は、自動でエリアスを付けられるので回避した場合も宣言する必要があります。
- 5行目　f.Type().Id("testGenerateImp").Struct()
  - 構造体宣言になります。
  - 構造体に限らずIdメソッドで変数などを宣言します。
- 7行目以降
  - 関数宣言です。関数は、関数名、引数、返り値、中身の順番で宣言します。

こちらの例ではファイルは作成されず、コンソールに出力結果が表示されます。
結果はこちらになります。

``` golang
package ci

import (
        ci "app/controllers/ci"
        "app/usecases/ui"
)

type testGenerateImp struct{}

func NewTestGenerate() ci.TestGenerate {
        return &testGenerateImp{}
}
```

## 今考えている構成に組み込む

### 利用想定の整理

インターフェイスを使う際の関係を自動作成で保管したいのでレイヤーで話すなら

- Controller
- Usecase
- Repository

を作成対象とします。
出力先のフォルダも事前に用意してファイルが作成される想定です。

Interfaceと注入の関係はInterfaceを「LoginController」注入側は「loginControllerImp」とします。
注入側はプライベート宣言にすることでパッケージが異なるときにInterfaceしか使えない状況を作り出すことでInterfaceを介さないと利用できない状態にしてクリーンアーキテクチャの実現できる環境にします。

ここまでの話を整理すると実行コマンドは

``` golang
go run generate/generator.go <作成種別>　<Interface名>
```

として、作成種別は数字で
1: Controller
2: Usecase
3: Repository
とします。
Interface名のみを宣言しますが、注入側はInterface名の戦闘を小文字に変換して末尾にImpを追加する処理を行います。

作成種別に対しては基本的な構造は似ているので、今回はControllerの作成処理のみとりあげます。
現在、私が考えているアーキテクチャでは、ControllerからはUsecaseを呼び出します。この場合、Controllerの構造体には利用するUsecaseのInterfaceを要素に追加し、注入関数でも対応する必要があります。LoginUsercaseが必要な場合を考えると

``` golang
package ci

import (
        "app/controllers/ci"
        "app/usecases/ui"
)

type testGenerateImp struct{
  login ui.LoginUsecase
}

func NewTestGenerate(login ui.LoginUsecase) ci.TestGenerate {
        return &testGenerateImp{
          login: login,
        }
}
```

となります。
1つのControllerで大量のUsecaseを使う場合に、注入関数が膨れ上がってしまう現状が起きてしまします。また、せっかく自動生成したのに、膨れ上がってしまうことは避けたい思いもあります。
そこでdig.Inをつかって引数を１つのみにします。
用意する構造体は、

``` golang
package ui

import "go.uber.org/dig"

type InUsecase struct {
  dig.In
  Login LoginUsecase
}
```

を用意し、新しいUsecaseを作成するたびにここに追加していきます。
実現したい出力内容は、

``` golang
package ci

import (
        "app/controllers/ci"
        "app/usecases/ui"
)

type testGenerateImp struct{
  login ui.LoginUsecase
}

func NewTestGenerate(uc ui.InUsecase) ci.TestGenerate {
        return &testGenerateImp{
          login: uc.Login,
        }
}
```

### 使ってみた

#### フォルダ構成

前回クリーンアーキテクチャの実現のときに考えたフォルダ構成から以下のように手を入れる
generatorフォルダを追加して、自動生成のための実行ファイルを置きます。
Usecase用の構造体はuiフォルダ（usecaseのInterface置き場）に配置します。

```
.
└── app/
    ├── consts
    ├── controllers/
    │   ├── ci
    │   └── cg/
    │       └── controllerGenerator.go
    ├── di/
    │   ├── controller.go
    │   ├── di.go
    │   ├── usecase.go
    │   └── repository.go
    ├── entities
    ├── generator/
    │   └── generator.go
    ├── infra/
    │   ├── database
    │   ├── logger
    │   └── server/
    │       └── server.go
    ├── repositories/
    │   └── ri
    └── usecases/
        └── ui/
            └── inusecase.go
```

Usecase用の構造体は

``` golang: app/usecase/ui/inusecase.go
package ui

import "go.uber.org/dig"

type InUsecase struct {
  dig.In
  Login LoginUsecase
}
```

実行するgenerator.goは

``` golang: app/generator/generator.go
package main

import (
  "app/controllers/cg"
  "app/infra/genarator"
  "app/repositories/rg"
  "app/usecases/ug"
  "log"
  "os"
  "strings"
)

const (
  controller = "1"
  usecase    = "2"
  repo       = "3"
)

func main() {
  if len(os.Args) < 3 {
    log.Println("引数がたりません")
    os.Exit(9)
  }

  cgs := genarator.CreateGenerator{
    In: os.Args[2],
  }

  cgs.Fn = createImpName(cgs.In)

  ts := os.Args[1]
  var err error

  switch ts {
  case controller:
    err = cg.CreateController(&cgs)
  case usecase:
    err = ug.CreateUsecase(&cgs)
  case repo:
    err = rg.CreateRepository(&cgs)
  }

  if err != nil {
    log.Fatal(err.Error())
  }
}

func createImpName(name string) string {
  sName := strings.Split(name, "")

  sName[0] = strings.ToLower(sName[0])

  return strings.Join(sName, "") + "Imp"
}

```

Controllerの自動生成は

``` golang: app/controllers/cg/controllerGenerator.go
package cg

import (
  "app/infra/genarator"
  "github.com/dave/jennifer/jen"
  "path"
)

func CreateController(cg *genarator.CreateGenerator) error {
  cg.BasePath = "/go/src/app/controllers/"

  err := createCi(cg)
  if err != nil {
    return err
  }

  err = createImp(cg)
  if err != nil {
    return err
  }

  return nil
}

func createCi(cg *genarator.CreateGenerator) error {
  f := jen.NewFile("ci")

  f.Type().Id(cg.In).Interface()

  f.Save(path.Join(cg.BasePath, "ci", cg.Fn+".go"))

  return nil
}

func createImp(cg *genarator.CreateGenerator) error {
  f := jen.NewFile("controllers")

  f.ImportName("app/controllers/ci", "ci")
  f.ImportName("app/usecases/ui", "ui")

  f.Type().Id(cg.Fn + "Imp").Struct()

  f.Func().Id("New"+cg.In).Params(
    jen.Id("uc").Qual("app/usecases/ui", "InUsecase"),
  ).
    Qual("app/controllers/ci", cg.In).Block(
    jen.Return(jen.Op("&").Id(cg.Fn + "Imp").Values()),
  )

  f.Save(path.Join(cg.BasePath, cg.Fn+".go"))

  return nil
}
```

ここまで実装できれば

``` 
go run generator/generator/go 1 LoginController
```

で自動生成されるようになります。


## 統括

Jennifer自体は、本来書く順番に対して同じ順番に宣言していく形式なので、
個人的には好きな利用方法でした。
コードになってしまうことで出力結果がイメージできないと用意するときやほかの開発者が修正したりするときに大変なので、宣言順と書く順番が一致していることで特別な解釈も少なくてよいので使いやすいモジュールだと思っています。

Interfaceと注入側との実装時の注意事項はコマンド１つで済むようになったので効率面では向上したと思っています。ですが、ルールが決まっている実装を自動化する観点ではいくつか課題も残っています。

- Interfaceに追加した関数の対応
  - Interfaceに追加したときに注入側にも追加する必要がある。現状では手動で追加する必要がある。
- dig.Inで対応した構造体への追加
  - Usecase作成は自動生成しているのだが、InUsecaseへは手動追加している。
  - 半自動化してしまっているので見逃しやすい可能性があるのでコマンド実行時にInUsecaseにも追加されると効率面ではかなり向上しそう。

どちらも既存ファイルを修正するパターンなのでJenniferでは対応できないので別の手法を考える必要があるのでいろいろ調べます。
