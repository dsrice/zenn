---
title: "クリーンアーキテクチャについて考える　前半"
emoji: "🕌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["CleanArchitecture","Architecture"]
published: false
---

## 概要
サービスのアーキテクチャを考える上で、様々な手法が存在している。
今回はクリーンアーキテクチャを取り入れることを考えている。
考えを理解したうえでフォルダ構成についてまとめる。

## 前提
クリーンアーキテクチャを採用するが、少し簡易的な構成を選択する。
理由としては、チームではなく個人運用であるサービスの構成案のため
複雑にしすぎても運用しきれないと考えているためである。
決まっていることとしては
- 言語はGo言語
- DataBaseはMySQL
- O/RマッパーはSQLBoilerの予定している（未確定だが、何かしらは使うつもり）

## クリーンアーキテクチャって？
![](/images/clean-arche/image.png)
引用　https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html

クリーンアーキテクチャといえば、この絵を見たことがある人は多いはず。
クリーンアーキテクチャのルールを表現するには非常に優れた絵だと思っています。
この絵が示しているルールは
__ルール１__ ソフトウェアはレイヤーに分割することで関心事を分離する
__ルール２__ ソースコードは円の内側の方向にのみ依存する
__ルール３__ 制御の流れと依存関係を逆転させて依存の方向を制御する

となっています。

### ソフトウェアはレイヤーに分割することで関心事を分離する
クリーンアーキテクチャに限らず、レイヤーを分割することは多くのアーキテクチャではよくある考え方です。
ですが、分割することで関心事を分離するとあることが重要な要素です。
関心事とは、プログラムの目的や役割ごとのクラスや関数などのアプリケーションを構成要素を指しています。
話を分かりやすくするためにIDとパスワードで認証するAPIを使って話を補足します。
IDとパスワードで認証するので認証成功時の処理の流れとしては、
1. リクエストを受け取る
2. リクエストパラメーターがすべてあること
3. IDのデータがあるか確認
4. IDに対応するパスワードとリクエストパラメータのパスワードが一致確認
5. 認証成功のレスポンスを返す

といった流れになります。
処理フローとして書いてしまうとすべて同等にみえてしましますが、
1,5はAPIとしてのInterfaceになります。指定されたリクエストと成功時のレスポンスです。
2,3,4は必要なビジネスロジックですし、3はより正確に分類するならDBへの問い合わせです。
このように、外の世界からみたときに役割を正しく分割することで密な依存関係を避ける狙いもあります。
APIはWebとして扱うので
1,5はInterce Adapter層のControllerとPresenterにあたります。
2,3,4はApplication Business RulesとしてUsecaseにあたります。
3に関しはDataBaseがかかわっているため、Interce Adapter層のGatewayが絡んでくる様子を含んでいます。

3をより正確に表現するならば
3-1. DataBaseにIDをつかって問い合わせ
3-2. 取得した情報をEntitiesに定めているクラスに格納
といった流れになりますので、3-1がGateway、3-2がUsecaseにあたります。


### ソースコードは円の内側の方向にのみ依存する
「依存する」とは、プログラムのビルドや実行時に別のプログラムが必要であることを指しています。
Go言語の世界観では、
```
package base

type PencilCase struct {
	Pen Pen
}

func New() PencilCase{
	pen := Pen{}
	return PencilCase{
		Pen: pen,
	}
}

type Pen struct {
	
}
```
の関係にある構造体があるときにPencilCaseはNewメソッド内でPenを利用しているため
Penが存在しないとビルドエラーになります。
そのためPencilCaseはPenに依存しているといいます。

依存関係はプログラムの修正、変更時の影響範囲に関係しており、
Penを変更した場合、
__依存しているPencilCaseにも影響があると考えるべきです。__
ですが、PencilCaseを変更した場合、
__Penは依存されているため影響が発生しないと考えるべきです。__

もっと日常的な例にたとえると
黒ボールペンがシャーペンに変われば、筆箱の中身としては別物ですが、
赤い筆箱が黒い筆箱に変わったとしても、筆箱の中身には影響がありません。
この変更は、内容によっては影響がないケースは存在していますが、
考え方としては、影響があるから確認した結果、影響がないと判断したです。

本来の話に戻しますが、
__ソースコードは円の内側方向にのみ依存する__
とは、どういうことを言っているのかに戻りますが、
最初のルールでソフトウェアはレイヤーに分割して関心事を分離しているます。
説明のために、
一番の外の円はWeb、UI
2番目の円はControllers
3番目の円がUsecase
を取り上げ、筆箱の中身の登録処理を考えてみます。

1. UI改善のためデザインを変更した場合
   1. 変更対象は1番外の円
   2. デザインのみなので筆箱の中身の登録内容には変更がないので2,3番目のレイヤーには影響がない
2. 筆箱の中身の登録時にバリデーションを追加する
   1. 変更対象はビジネスロジックになるのでUsecaseの3番目の円
   2. Controllerからビジネスロジックを呼び出す処理が必要になるので修正が必要
   3. UIはバリデーションの結果エラーになったときに表示などの対応が必要
3. 利用状況のためにリクエストパラメータに分析トークンを追加
   1. 変更対象はControllerになるとする
   2. UI側ではリクエストパラメータに追加する必要があるので変更が必要
   3. 筆箱の中身の登録には関係のない変更のため影響がなし

といった話になります。
依存関係がむちゃくちゃになっているとあっちもこっちも修正しなければならなくなり、テストコードを用意する際にも大きな影響を与えることになってしまうので依存関係を守る本ルールは非常に重要です。

### 制御の流れと依存関係を逆転させて依存の方向を制御する
このルールは単独で意味があるというよりルール２である円の内側方向にのみ依存することを守るためのルールになっている。
例として、ControllerとビジネスロジックのServiceの２つを考える。
実行順としては、Controllerから始まり、ビジネスロジックであるServiceを呼び出す。結果を受けてControllerの処理を完結する。

```
type Controller struct{}

func(c *Controoler) Start() error {
   s := Service{}
   err := s.Logic("test")

   return err
}

type Service struct{}

func(s *Service) Logic(target string) error{
   log.Println(target)

   return nil
}
```

この内容では、依存関係は
ControolerはServiceに依存しています。
また、制御の流れも
ControllerからServiceに流れています。

これでは、
Serviceに変更があったときにControllerに影響があると考えられ、
内側の円の内容が変更があった時に外側の円に影響がでやすい構造となってしまいます。
ルール1で関心事でレイヤーを分割し、ルール２に依存する方向を定めていたのに
ソースコードの影響範囲が逆点してしまっているのでルールを設定した意味がなくなってしまっています。

こうならないために制御の流れを反転させたいのです。
今回制御の流れを反転させるためには、Interfaceを利用します。

```
type Controller struct{}

func(c *Controoler) Start() error {
   var s IService
   s = Service{}
   err := s.Logic("test")

   return err
}

type IServece interface{
   Login(target string) error
}

type Service struct{}

func(s *Service) Logic(target string) error{
   log.Println(target)

   return nil
}
```

実装量が増えてしまっていますが、ControllerはIServiceというInterfaceに依存するようになりました。
また、ServiceはIServiceに依存しています。
ですが、制御の流れは変わっていません。
非常にわかりにくい話ではあるのですが、
最初の実装では、入口にあたる部分からServiceの実装に含まれていました。
しかし、今回の実装では入口がIServiceが担っているためにServiceはその中の実装のみを対応するようになっています。

これによって影響範囲はどう変わるのでしょうか？
Serviceに変更があったときにはControllerに影響がでなくなっています。
これは入口が別の担当になったためにビジネスロジックのみの変更ならばControllerには影響がでなくなっています。もちろん、ビジネスロジックの結果が変わるようになり、Controller側でその結果を活用しているといったケースは別です。

現実的なたとえるなら配達です。
配達は住所で場所を特定し、氏名で受取人か確認して荷物を渡します。
住所は建物がある限り、住人が変わっていようが固定のものです。
住所がInterfaceになっています。
ですが、住人は会うまで本人であるかは不明のままです。
入口という住所は同じでも住んでいる人次第で結果は異なります。
住人に関する情報がServiceとしてロジックになります。
少し強引な部分もありますが、理解する足がかりとしては想像しやすい話だと思います。

元来コーディングは具象に依存するべきされてきてInterfaceが実装されてきたのは仕事でプログラムをやっている人には経験があることだと思います。
私もよく見てきました。
ですが、世の中は不思議でInterfaceを用意するという事象のみが残ってしまってInterfaceになぜかメソッドがないという実装を見たこともある方もきっといるでしょう。（筆者はあるし、メソッドちゃんと書いたらなぜか怒られたこともあります。今でも怒られた理由は謎のままです）
クリーンアーキテクチャの実現では、ちゃんとInterfaceを活用しましょうということですので
皆さん注意しましょう。

## 最後に
長くなったのでどういう構成にするかは次回に回します。
クリーンアーキテクチャの基本的な思想に振れた内容でした。
実装するうえでのメリットやデメリットも構成検討時に触れていきたいと思います。

もっと真面目にクリーンアーキテクチャついて考えると考えるべきことや思想的なところはたくさんあります。
もっと深堀りするときは何かの書籍を読みながら書いていこうかと思います。