---
title: "Source Geratorsを調べてみた"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["csharp"]
published: false
---

## 概要

先日.NET10についていろいろ調べている中で「Source Generators」というワードが自動生成的な要素ででてきた。
C#でそんな機能あったっけ？と思ったので調べたので理解度を高めるために記事にすることにした。

こういったのはどういった課題を疑問視していたのか？という観点から始めると理解しやすいと思っているので歴史から始める。

## C#におけるコード生成の歴史

### 黎明期：手動コード生成（2000年台初頭）

C#1.0の時代のコード生成は、

``` csharp
// 手書きでボイラープレートを量産
public class Person
{
    private string name;
    public string Name 
    { 
        get { return name; } 
        set { name = value; } 
    }
    
    private int age;
    public int Age 
    { 
        get { return age; } 
        set { age = value; } 
    }
}
```

私も昔の状態から回収されていないクラスで見たことがあります。
当時はJava、Rudy、Pythonとやってきた中でC#のこのコードを見たときに意味不明すぎて固まった記憶があります。

問題点としたは、

- 機械的な作業が多い
- ミスが発生しやすい
- リファクタリングが困難
  
とくにリファクタリングが困難なのはその通りだと思います。
上記のものなら特に問題がないのですが、set時に謎の処理が入っていると簡単に回収できなくなってしまうので容易に進められなかった記憶があります。

### 自動実装プロパティの登場（C#3.0 2007年）

``` csharp
public class Person
{
    public string Name { get; set; }
    public int Age { get; set; }
}
```

普通に実装されていればこの状態なのではないかと思います。
画期的だった理由としては以下のものがあげれます。

- コンパイラが自動でパッキングフィールドを生成
- コード量が劇的に減少
- 初めての「コンバイラによる自動生成」の成功例
  
先ほどのプログラムをリファクタリングしたときも利用全クラスを回収してすっきりさせました。
コード量が減少する量が異常でPRだしたときのメンバーからこんなに減っていいのか？
と再確認されたのも記憶に残っています。

### T4テンプレートの時代(2008年～)

Visual Studio 2008でT4（Text Template Transformation Toolkit）が導入されました。

``` csharp
<#@ template language="C#" #>
<#@ output extension=".cs" #>
<#
    string[] properties = { "Name", "Age", "Email" };
#>
public class Person
{
<# foreach(var prop in properties) { #>
    public string <#= prop #> { get; set; }
<# } #>
}
```

利点としては以下の内容が挙げれる。

- データベーススキーマからモデルクラスを生成
- 繰り返しパターンの自動化

データベーススキーマから取得するケースは以下のような実装で実現できる。

T4テンプレート

``` csharp
<#@ template debug="true" hostspecific="true" language="C#" #>
<#@ assembly name="System.Core" #>
<#@ assembly name="System.Data" #>
<#@ import namespace="System.Data" #>
<#@ import namespace="System.Data.SqlClient" #>
<#@ output extension=".cs" #>
<#
    string connectionString = "Server=localhost;Database=MyDb;Integrated Security=true;";
    var tables = GetTables(connectionString);
#>
using System;

namespace MyApp.Models
{
<# foreach(var table in tables) { #>
    public class <#= table.Name #>
    {
<#     foreach(var column in table.Columns) { #>
        public <#= column.CSharpType #> <#= column.Name #> { get; set; }
<#     } #>
    }

<# } #>
}

<#+
    // データベースからテーブル情報を取得
    List<TableInfo> GetTables(string connectionString)
    {
        var tables = new List<TableInfo>();
        
        using (var connection = new SqlConnection(connectionString))
        {
            connection.Open();
            
            // テーブル一覧取得
            var tableQuery = @"
                SELECT TABLE_NAME 
                FROM INFORMATION_SCHEMA.TABLES 
                WHERE TABLE_TYPE = 'BASE TABLE'";
                
            using (var cmd = new SqlCommand(tableQuery, connection))
            using (var reader = cmd.ExecuteReader())
            {
                while (reader.Read())
                {
                    var tableName = reader.GetString(0);
                    tables.Add(new TableInfo { Name = tableName });
                }
            }
            
            // 各テーブルの列情報取得
            foreach (var table in tables)
            {
                var columnQuery = @"
                    SELECT COLUMN_NAME, DATA_TYPE, IS_NULLABLE
                    FROM INFORMATION_SCHEMA.COLUMNS
                    WHERE TABLE_NAME = @tableName";
                    
                using (var cmd = new SqlCommand(columnQuery, connection))
                {
                    cmd.Parameters.AddWithValue("@tableName", table.Name);
                    using (var reader = cmd.ExecuteReader())
                    {
                        while (reader.Read())
                        {
                            var column = new ColumnInfo
                            {
                                Name = reader.GetString(0),
                                SqlType = reader.GetString(1),
                                IsNullable = reader.GetString(2) == "YES"
                            };
                            column.CSharpType = MapSqlTypeToCSharp(column.SqlType, column.IsNullable);
                            table.Columns.Add(column);
                        }
                    }
                }
            }
        }
        
        return tables;
    }
    
    string MapSqlTypeToCSharp(string sqlType, bool isNullable)
    {
        var csharpType = sqlType.ToLower() switch
        {
            "int" => "int",
            "bigint" => "long",
            "smallint" => "short",
            "tinyint" => "byte",
            "bit" => "bool",
            "decimal" or "numeric" or "money" => "decimal",
            "float" => "double",
            "real" => "float",
            "datetime" or "datetime2" or "date" => "DateTime",
            "uniqueidentifier" => "Guid",
            "nvarchar" or "varchar" or "nchar" or "char" or "text" => "string",
            _ => "object"
        };
        
        // 値型で nullable の場合
        if (isNullable && csharpType != "string")
        {
            return csharpType + "?";
        }
        
        return csharpType;
    }
    
    class TableInfo
    {
        public string Name { get; set; }
        public List<ColumnInfo> Columns { get; set; } = new List<ColumnInfo>();
    }
    
    class ColumnInfo
    {
        public string Name { get; set; }
        public string SqlType { get; set; }
        public bool IsNullable { get; set; }
        public string CSharpType { get; set; }
    }
#>
```

得られる結果

``` csharp
using System;

namespace MyApp.Models
{
    public class Users
    {
        public int Id { get; set; }
        public string Name { get; set; }
        public string Email { get; set; }
        public DateTime? CreatedAt { get; set; }
        public bool? IsActive { get; set; }
    }

    public class Orders
    {
        public int Id { get; set; }
        public int UserId { get; set; }
        public DateTime? OrderDate { get; set; }
        public decimal? TotalAmount { get; set; }
    }
}
```

C#関係なしにORMでスキーマ優先でモデルを作れるものは似たような実装になっています。
こちらの実装はまだ中途半端で、FKなどの対応が入っていません。
そういった意味では可能だけど用意するのは非常に難易度が高いものでした。

T4テンプレートの問題点は以下のものがあります。

- IDE統合が弱い
- デバックが困難
- ビルドプロセスとの統合が不完全
- 生成タイミングが不明確

### リフレクション全盛期（2010年代）

ASP.NET MVCやEntity Frameworkの普及とともに、リフレクションが多用されるようになりました。

``` csharp
// 実行時にプロパティ情報を取得
var properties = typeof(Person).GetProperties();
foreach (var prop in properties)
{
    var value = prop.GetValue(person);
    // シリアライズなどの処理
}
```

メリットとしては

- 柔軟で強力
- コードがシンプル
- 型を問わず汎用的

致命的な課題としては

- パフォーマンスが悪い（特にホットパス）
- 実行時エラーのリスク
- トリミング（AOT）との相性が悪い

### .NET Coreとパフォーマンス重視の流れ(2016年～)

.NET Coreの登場により、マイクロソフトはパフォーマンスを最重要視するようになりました。
TechEmpower Benchmarksでの競争が激化し、.NETのランキング向上が目標となりました。

``` csharp
// System.Text.Json (2019) の初期実装
// リフレクションベース → 遅い
JsonSerializer.Serialize(person);
```

課題として

- リフレクションがボトルネック
- AOT（Ahead-of-Time）コンパイルとの非互換
- コールドスタートの遅さ（サーバーレス環境で問題）

確かに、このころのC#は遅い印象がありました。
Windowsサーバーという制約から脱却しようとしているが他言語の処理速度比較してもまったく劣っているというのも新規サービスを立てるときの不採用理由になりがちでした。

これらを抱えている中でSource Generatorsが誕生する。

## Source Generators誕生の背景

### Roslynプロジェクトの成果

2011年から開始された**Roslyn**（C#コンパイラのオープンソース化）プロジェクトが基盤となりました。

**Roslynの革新:**

- コンパイラがAPIとして公開
- 構文木（Syntax Tree）への直接アクセス
- コンパイルパイプラインへの介入が可能

RoslynはC#で動いているなら導入しているLinterだと思います。
これらの内容を細かく書いてもいいのですが、これだけで記事ができそうなので別の機会にします。

### 具体的な問題：System.Text.Jsonのケース

2019年、.NET Core 3.0でSystem.Text.Jsonが登場しましたが、初期バージョンはリフレクションベースでした。

**パフォーマンステスト結果:**

```
Newtonsoft.Json:    500 MB/s
System.Text.Json:   800 MB/s (リフレクション)
期待される性能:    2000+ MB/s
```

当時はあまりSystem.Text.Jsonが推奨されてなく、サービス内両方混在してしまっていて取りまわしが面倒になっていたりすることもあったようです（経験談）。

この状況は以下のジレンマに陥りました。

- 手がりでシリアライザを書けが速いが、現実的ではない
- リフレクション使用では遅い
- T4テンプレートは使いにくい
  
そこで、他言語のアプローチを参考に認識を改めて
「コンパイル時のコード生成」を重要視するようになりました。

.NET5以降では、AOT（Ahead of Time）が重要になりました。

## Source Generatorsの登場（2020年）

### リリースタイムライン

- **2020年4月**: プレビュー発表
- **2020年11月**: .NET 5 / C# 9.0で正式リリース
- **2021年11月**: .NET 6 / C# 10でIncremental Generators導入
- **2022年〜**: 主要ライブラリが続々採用

### 設計思想

**原則:**

1. **加算のみ（Additive）**: 既存コードを変更せず、新しいコードを追加
2. **決定論的（Deterministic）**: 同じ入力に対して常に同じ出力
3. **高速（Fast）**: インクリメンタルビルドに対応
4. **デバッグ可能（Debuggable）**: 生成コードが確認できる

### 技術的な仕組み

```
ソースコード (.cs)
    ↓
[構文解析]
    ↓
構文木 (Syntax Tree)
    ↓
[Source Generator 実行] ← ここで新しいコード生成
    ↓
拡張された構文木
    ↓
[セマンティック解析]
    ↓
[IL生成]
    ↓
アセンブリ (.dll)
```

### 初期実装例

``` charp
// Generator実装（簡略版）
[Generator]
public class NotifyPropertyChangedGenerator : ISourceGenerator
{
    public void Execute(GeneratorExecutionContext context)
    {
        // 属性を持つクラスを探す
        var classesToGenerate = FindClassesWithAttribute(context);
        
        foreach (var classSymbol in classesToGenerate)
        {
            // プロパティ変更通知コードを生成
            var code = GenerateNotifyCode(classSymbol);
            context.AddSource($"{classSymbol.Name}.g.cs", code);
        }
    }
}

// 使用例
public partial class Person
{
    [Notify]
    private string _name;
    
    // 以下が自動生成される:
    // public string Name 
    // { 
    //     get => _name;
    //     set { _name = value; OnPropertyChanged(); }
    // }
}
```

まだ実感しないが、先ほどのSystem.Text.Jsonも含めた改善例で話を補足する

## 採用事例

### System.Text.Json

**.NET6以降**

``` csharp
[JsonSerializable(typeof(Person))]
public partial class MyJsonContext : JsonSerializerContext { }

// 使用
var json = JsonSerializer.Serialize(person, MyJsonContext.Default.Person);
```

**パフォーマンス改善:**
```
リフレクション版:        800 MB/s
Source Generator版:    2,500 MB/s
改善率: 約3倍
```

**メモリ使用量:**
```
リフレクション版:   大量のメタデータをキャッシュ
Generator版:        ほぼゼロオーバーヘッド
```

### Entity Framework Core

``` csharp
[DbContext(typeof(MyDbContext))]
public partial class MyDbContextFactory { }

// コンパイル時にクエリが最適化される
```

**効果:**

- クエリのコンパイル時最適化
- AOT互換性の向上
- 起動時間の短縮

### パフォーマンスへの影響

**ASP.NET Core APIのベンチマーク:**
```
シナリオ: JSON APIの応答時間

リフレクションベース:
- 平均応答時間: 15ms
- メモリ使用量: 50MB
- 起動時間: 3秒

Source Generator使用:
- 平均応答時間: 8ms (約47%改善)
- メモリ使用量: 30MB (40%削減)
- 起動時間: 1秒 (67%削減)
```

こちらは元の実装にもよるところなので、一つの参考例程度に見てほしいです。
ですが、当時C#サービスがNewtonsoft.Jsonを使っていた中で、
Microsoftが自身で作っているSystem.Text.Jsonを推奨し始めたのは記憶に残っています。調査したら劇的に早くなったし、メモリ使用率も下がったので印象的でした。
それで、いろいろあってVersionをさげてみたら恐ろしく遅くなった経験があります。

そういった話もありますが、ASP.NET CoreでAPIサーバーを構築するとSystem.Text.Jsonを使っているので処理速度が改善されるようになっています。
実装時に使っていなくてもフレームワーク内で使われている技術であることがわかります。
今回記事にしようと思った最大の理由はこれです。
確かに知らなくてもフレームワークは使えます。
Json周りもルールを知らなくても設定可能です。
ですが、裏ではこういった技術によって支えられていることは知っておくべきことだと思ったからです。
現に、Microsoftは自作ライブラリに積極的に導入しています。
ASP.NET CoreでEntity Framework Coreという組み合わせは少なくないと思います。
両者で使われている技術になっているので、重要な位置づけでになります。

## 課題と批判

一見積極的に取り入れてたいSource Generatorsですが、完璧ではありません。
様々な観点の課題、問題点をまとめます。

### 開発者からの懸念

#### 1. 可視性の問題

**Stack Overflowでの議論（2021年）:**

- 「生成されたコードがどこにあるか分からない」
- 「obj/フォルダを探す必要がある」
- 「Visual Studioでは見やすいが、Riderやコマンドラインでは不便」

**Visual Studio 2022での改善:**

- Solution Explorerに「Dependencies > Analyzers」ノードが追加
- 生成されたファイルが表示されるように

#### 2. ビルド時間の増加

実際のプロジェクトでの報告例:

```
中規模プロジェクト (50,000行):
Generator導入前: ビルド時間 8秒
Generator導入後: ビルド時間 12秒 (50%増加)
```

#### 3. デバッグの困難さ

``` csharp
// エラーメッセージが分かりにくい
error CS0246: The type or namespace name 'GeneratedClass' could not be found
// → Generatorがエラーで生成に失敗している
```

### 学習曲線の高さ

**必要な知識:**

- Roslyn API
- 構文木の理解
- セマンティックモデル
- ISymbol体系
- インクリメンタルな設計

**コミュニティの声:**

- 「Generatorを書くのは難しい」（GitHub Issues）
- 「ドキュメントが不十分」
- 「サンプルが少ない」

### ツールチェーンの問題

**IDE対応:**

- Visual Studio: 良好
- Rider: 対応済みだが一部問題あり
- VS Code: 限定的なサポート

**CI/CD:**

- キャッシュの扱いが複雑
- インクリメンタルビルドの不具合

## 使い分け

導入には、ハードルが非常に高いという問題は抱えているものの
導入に成功できれば受けれる恩恵があるというのも事実です。
開発現場で生成したコードがどこにあるのか？というのも開発時に気になる問題ではあります。

私個人が考える使い分けに関しては以下にまとめます。
様々な意見があると思いおますので、意見があればコメントしていただければト思います。

### 推奨事項

**成熟したGeneratorを使う場合**
すでに成熟しているGeneratorを採用されたライブラリを使うケースです。
似た機能を有しているライブラリが2つあるときの比較条件としてみればよいと思います。
採用されているほうが処理は早いことがあるので積極的に採用していただければと思います。

- System.Text.Json
- Entity Framework Core

など

### 検討事項

- DTO/モデルのマッピング
  - 共通プロパティなどがあれば有用だと思います。
- バリデーションコード
  - 共通のバリデーション処理があればと思います。

### 避けるべき事項

- ルールが複雑すぎる
  - ビジネスロジックが絡んだり、条件分岐が多い
- 数回しか使わない場合
  - 手書きのほうが早い
  - オーバーエンジニアリング
- チーム全体の理解が得られない場合
  - 属人化のリスク
  - 保守が困難

### 実践的な内容

``` csharp
// Good: シンプルで明確なパターン
[AutoMapper]
public partial class PersonDto { }

// Bad: 複雑なビジネスロジック
[ComplexValidation(Rules = "If Age > 18 AND Country = 'US' THEN ...")]
public partial class Person { }
```

## 今後の展望

### .NET 8/9での進化

**Incremental Generatorsの改善:**

- さらなるパフォーマンス最適化
- より詳細なキャッシュ制御

**Native AOTの普及:**

- リフレクション完全排除の方向性
- Source Generatorsがより重要に

### コミュニティの動向

**人気が高まっているライブラリ:**

- **Mapperly**: AutoMapperの代替
- **StronglyTypedId**: 型安全なID
- **Vogen**: 値オブジェクト生成

**GitHubでの統計（2024年）:**

```
"Source Generator"を含むリポジトリ: 5,000+
関連NuGetパッケージ: 500+
成長率: 年間50%増
```

## 結論：コード生成の新時代へ

### Source Generatorsが変えたこと

**技術的:**

- リフレクションからコンパイル時生成へのパラダイムシフト
- AOT互換性の実現
- パフォーマンスの大幅改善

**開発文化:**

- 「ボイラープレートは自動生成するもの」という認識
- コンパイラを拡張可能なツールとして活用
- 宣言的プログラミングの促進

### 現実的な評価

**成功している領域:**

- ライブラリ開発（System.Text.Json, EF Core）
- パフォーマンスクリティカルなアプリケーション
- 大規模プロジェクトでの定型コード削減

**まだ課題がある領域:**

- 小規模プロジェクトでの採用
- カスタムGeneratorの開発ハードル
- ツールサポートの統一

### 最後に：銀の弾丸ではない

Source Generatorsは強力なツールですが、**適材適所**が重要です。

**推奨アプローチ:**

```
1. まずは既存の成熟したGeneratorを使う
2. 効果を測定する
3. 本当に必要な場合のみカスタムGeneratorを検討
4. チーム全体で理解を共有する
```

.NET10でのSource Generatorsを使った自動生成の実現に関しては別の記事で書こうと思います。
