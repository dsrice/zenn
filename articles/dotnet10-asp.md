---
title: ".NET10がリリースされたから調べてみた[ASP.NET編]"
emoji: "🕌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["dotNET10","csharp", "ASP"]
published: false
---

## 概要

2025年11月に.NET10がリリースされました。
偶数バージョンなのでLTSリリースとなるのでサポートも2028年11月までの3年間サポートとなっています。
.NET8を使っていてサポート終了前に切り替えようとする方も多いのではないのでしょうか？
何が新しくなったのかという観点でいろいろ書いていこうと思いますが、
.NET9からというより.NET8からどう変わるのかで書きたいと思います。
また、個人的な感想も書かせてもらいます。こればかりは、人によると思いますので
むしろ、私はこう思うなどがあれば視野を広げるためにも教えていただきたいです。

今回はASP.NET編です。

## 変更内容

ASP.NETの変更点は以下のものがある。

- OpenAPI 3.1のサポート
- Blazorの改善
- ルート属性の改善
- ファイルベースアプリのサポート

詳細は以下に記載します。

### OpenAPI 3.1のサポート

#### .NET8まで

Swashbuckleがデフォルトで使用され、OpenAPI仕様に基づいたドキュメント生成とSwagger UIの提供されていた。

#### .NET9

ASP.NET Coreに組み込みのOpenAPIドキュメント生成サポートが導入され、Microsoft.AspNetCore.OpenApiパッケージが提供されるようになりました。

対応理由としては、
Swashbuckleライブラリのメンテナンス不足と、外部ツールへの依存を減らし、.NETの迅速なリリースサイクルをサポートするためとされています。

このときの特徴としては、

- デフォルトでOpenAPI 3.1ドキュメントの生成をサポートし、JSON Schema draft 2020-12に完全対応 
- ランタイムおよびビルド時のドキュメント生成に対応
- トランスフォーマーAPIによるドキュメントのカスタマイズが可能
- Native AOTと完全互換

#### .NET10

OpenAPI 3.1のサポートには、基盤となるOpenAPI.NETライブラリのバージョン2.0へのアップデートが必要で、いくつかの破壊的変更が含まれます。

主な徘徊的変更は

- 型の変更
  - ドキュメント内のエンティティ（操作やパラメータなど）がインターフェース型として定義されるようになり、インライン版と参照版の具体的な実装が存
  - 例: IOpenApiSchemaはOpenApiSchema（インライン）またはOpenApiSchemaReference（参照）のいずれか
- Nullableプロパティの削除
  - OpenApiSchema型からNullableプロパティが削除され、型がnullableかどうかはOpenApiSchema.TypeプロパティがJsonSchemaType.Nullを含むかで判断
- OpenApiAnyクラスの廃止
  - OpenApiAnyクラスが廃止され、代わりにJsonNodeを直接使用するように変更

コードの移行内容

``` csharp
// .NET 9まで
schema.Example = new OpenApiObject
{
    ["date"] = new OpenApiString(DateTime.Now.AddDays(1).ToString("yyyy-MM-dd")),
    ["temperatureC"] = new OpenApiInteger(0),
    ["temperatureF"] = new OpenApiInteger(32),
    ["summary"] = new OpenApiString("Bracing"),
};

// .NET 10以降
schema.Example = new JsonObject
{
    ["date"] = DateTime.Now.AddDays(1).ToString("yyyy-MM-dd"),
    ["temperatureC"] = 0,
    ["temperatureF"] = 32,
    ["summary"] = "Bracing",
};
```

他にも以下の点が改善されている。

- OpenAPI 3.1では、`$ref`と一緒に説明（description）をプロパティとして含めることが可能になった。
- 生成されたOpenAPIドキュメントをYAML形式で提供可能に
- XMLコメントからのメタデータ抽出が改善
- JSON Patchオペレーションのメディアタイプが正しく適用されるように


この手の推奨ライブラリが変わるというのはよくある話ですが、
現状は中途半端です。
というのもMicrosoft.AspNetCore.OpenApiが担っているのはJsonもしくはYamlファイルの生成までです。ASP.NETの開発環境でUIが必要ならばSwashbuckleのUIライブラリが必要になります。
この辺りも改善されてくると切り替えてくる人たちが増えるんだと思います。
Swashbuckleも.NET10対応もされたので切り替えずに使う人も多いのではないでしょうか？（記事書いてたら5分前に新しいVersion出してるくらい微修正しているけど）

### Blazorの改善

#### Blazorとは

Blazorは.NETのフロントエンドのWebフレームワークで、単一のプログラミングモデルでサーバー側とレンダリングとクライアント対話機能の両方をサポートしている。

従来の開発なら
BE： C#、Jave、Python
FE： TypeScript

Blazorなら
BE、FEともにC#

と、C#のみで実装することが可能となる。

#### .NET8での転換期

そもそもBlazorは.NET8で大きな変革をもたらされた。
フルスタックWebUIフレームワークとなり、コンポーネントまたはページレベルで、静的サーバーレンダリングとインタラクティブサーバーレンダリングの両方のコンテンツをレンダリングできるようになった。

レンダーモードの導入

- **Static SSR**: サーバーで静的HTMLを生成（高速、軽量）
- **Interactive Server**: SignalRでサーバーと接続してインタラクティブに
- **Interactive WebAssembly**: クライアント側でWebAssembly実行
- **Interactive Auto**: 最初はServer、バックグラウンドでWASMをダウンロードして次回以降はWASM

#### .NET10で起きたこと

1. 永続状態管理の簡素化
  PersistentState属性を使用して、コンポーネントやサービスから永続化する状態を宣言的に指定できるようになりました。この属性を持つプロパティは、プリレンダリング中にPersistentComponentStateサービスを使用して自動的に永続化されます。

``` csharp
[PersistentState]
public List<Contact>? Contacts { get; set; }
```
 **利点:**

- プリレンダリング時にデータを取得し、インタラクティブレンダリング時に再利用
- 「フラッシュ」効果（データが一瞬消えて再表示される現象）の解消
- データベースへのリクエスト回数が半減
- Enhanced Navigation中の永続的なコンポーネント状態の処理をサポート

2. Blazorスクリプトの配信改善
 .NET 10以降、Blazorスクリプトは埋め込みリソースではなく、自動圧縮とフィンガープリントを備えた静的Webアセットとして提供されます。
 blazor.web.jsスクリプトが静的アセットとして提供されるようになり、自動圧縮とフィンガープリントの恩恵を受け、ファイルサイズが183KBから約43KBへと76%削減されました

3. 検証システムの強化
 検証システムが更新され、ネストされたオブジェクトのプロパティやコレクションアイテムの検証をサポートするようになりました。

 有効化方法

 ``` csharp
// Program.cs
builder.Services.AddValidation();

// モデルクラス
[ValidatableType]
public class NewContactForm 
{
    public Contact Contact { get; set; }
    public string? Honeypot { get; set; }
}
 ```

4. ナビゲーションの改善
 以前は、NavigationManager.NavigateToが同一ページへのナビゲーション時にページトップへスクロールしていましたが、.NET 10ではこの動作が変更され、同一ページへのナビゲーション時にページトップへスクロールしなくなりました。
 これにより、クエリ文字列やフラグメントを変更する際のユーザー体験が向上します。

5. 再接続UIの改善
 Blazor Web Appプロジェクトテンプレートに、クライアントがWebSocket接続を失った際の再接続UIを改善するためのReconnectModalコンポーネント（CSS、JavaScriptファイル含む）が含まれるようになりました。

6. WebAssemblyプリロード制御
 `<LinkPreload />`などのコンポーネントを使用して静的アセットとWASMのプリロード制御が可能になり、起動パフォーマンスが向上します。

7. QuickGridの機能強化
 RowClassパラメータにより、条件付きで行をスタイリングでき（例：アーカイブされた行に異なるクラスを適用）、QuickGridマークアップに直接記述できます。また、HideColumnOptionsAsync()を使用してプログラムで列オプションを閉じることができます。

8. パスキー認証のサポート
 ASP.NET Core Identityがパスキー認証（WebAuthn/FIDO2標準）に正式対応し、Blazor Web Appのプロジェクトテンプレートには標準でパスキーの管理とログイン機能が組み込まれました。

### ルート属性の改善

ルーティングは、受信したリクエストのURLのコントろーたのアクションメソッドにマッピングする機能です。ルート属性を使用することでURLパターンを定義できます。

#### .NET8で導入された機能

- 構文ハイライト
  - ルートの異なる部分がIDEがハイライト表示されるようになった。
- リアルタイムのルート検証
  - 構文ハイライトとビルド時にルート構文エラーを警告するアナライザーにより、開発者がルートを使いやすくなりました
- オプショナルパラメータの型チャック
  - ルーティングはオプショナルパラメータをサポートし、例えば `/blog/archive/{date?}` は `/blog/archive` と `/blog/archive/2023-4-1` の両方にマッチします。オプショナルパラメータがnull非許容のメソッドパラメータにバインドされている場合、アナライザーが検出して警告します

``` csharp
// 構文ハイライトとリアルタイム検証
app.MapGet("/product/{id:int}", (int id) => ...);

// オプショナルパラメータの検証
app.MapGet("/blog/archive/{date?}", (DateTime? date) => ...);
// ↑ DateTime? (nullable) に修正することをfixerが提案
```

#### .NET10での改善

**ルート構文ハイライトの強化**
[Route]属性がルート構文ハイライトをサポートするようになり、ルートテンプレートの構造を視覚的に確認できるようになりました。

``` csharp
[Route("api/[controller]")]
public class ProductsController : Controller
{
    // GET: api/products
    [HttpGet]
    public IActionResult GetAll() { }
    
    // GET: api/products/5
    [HttpGet("{id:int}")]
    public IActionResult Get(int id) { }
    
    // GET: api/products/search?name=test
    [HttpGet("search")]
    public IActionResult Search(string name) { }
}
```

### ファイルベースアプリのサポート

今まではプロジェクトをたて、決まった構成ファイルを用意していた。
従来なら

- `Program.cs`
- `MyApp.csproj`

とコードとプロジェクトファイルで最低限構成されていた。
しかし、.NET10では

- `app.cs`

のみでよくなり、`dotnet run app.cs` で実行できるようなった。

NuGetを使う場合は以下のように宣言します。

``` csharp
// app.cs
#:package Newtonsoft.Json@13.0.3

using Newtonsoft.Json;

var data = new { 
    date = DateTime.Now, 
    message = "Hello from file-based app!" 
};

string json = JsonConvert.SerializeObject(data, Formatting.Indented);
Console.WriteLine(json);
```

WebAPIのアプリケーションの場合は

``` csharp
// HelloHeroku.cs
#:sdk Microsoft.NET.Sdk.Web

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/", () => "Hello from .NET 10 on Heroku!");

app.Run();
```

プロジェクトへの変換も可能となっており、
`dotnet project convert app.cs`を実行すると、プロジェクトファイルが自動的にセットアップされ、ファイル用のフォルダーが作成され、.csprojが追加され、app.csの#:ディレクティブが削除され、これらのディレクティブが適切なMSBuild設定と参照に変換されます

一見便利ですが、用途に応じた使い分けは必要です。
向いているケースとしては、
✅ クイックスクリプト・ユーティリティ
✅ データ移行の一度きりのタスク
✅ API テスト
✅ プロトタイピング
✅ 学習・教育目的
✅ ビルドスクリプト
✅ サンプルコード

向いていない場合は
❌ 300行以上のコード
❌ テストフレームワークが必要
❌ 多数のクラスがある
❌ エンタープライズグレードの構造が必要
❌ ソースジェネレーターを使用

この場合分けは、現在の制限事項から生まれています。
現時点では、厳密には単一ファイルのみとなっていおり、複数ファイルが必要な場合は
次バージョン以降に期待するしかありません。
とはいっても、データ移行みたいな一度きりの処理をプロジェクトを作らずに行ける点は便利です。

ファイルベースアプリは、C#をPythonやJavaScriptのようなスクリプト言語と同じくらい手軽に使えるようにしながら、型安全性、パフォーマンス、.NETの豊富なエコシステムの恩恵を受けられる画期的な機能になる可能性を感じました。



