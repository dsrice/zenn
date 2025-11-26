---
title: ".NET10がリリースされたから調べてみた[EntityFramework 編]"
emoji: "🕌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["dotnet","csharp", "entityframework"]
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

今回はEntityFramework編です。

## 変更内容

一部特定のDBに特化したものがあるが、箇条書きで並べると

- LINQ演算子の強化
- ベクターデータ型のサポート
- ExecuteUpdateの簡素化
- Complex Typesのサポート強化
- Azure Cosmos DBの改善
- パラメータ化されたコレクションの処理最適化
- SQLログのセキュリティ強化
- SQL Server2025の新しいJSON型のネイティブサポート

となっている。
以下では、各項目を細かく見ていく。

### LINQ演算子の強化

個人的には一番ありがいたい強化点だと思っています。

DB構造がサービスの成長とともに複雑になっていくのは仕方のないことだが、
その事象に対してEntityFrameworkはLeftJoinなどが複雑な記載をする必要性が求められることが多くあり、
コードの難読化が進んでしまう要因に感じていたからです。

RDBを使う以上、Joinは避けれない行為なのでこの辺りの簡素化は非常に恩恵があると思っています。

#### Before

メソッド構文

``` csharp
var query = context.Customers
    .GroupJoin(
        context.Orders,
        customer => customer.Id,
        order => order.CustomerId,
        (customer, orders) => new { Customer = customer, Orders = orders })
    .SelectMany(
        x => x.Orders.DefaultIfEmpty(),
        (customer, order) => new 
        { 
            CustomerName = customer.Customer.Name,
            OrderNumber = order?.OrderNumber ?? "N/A"
        });
```

クエリ構文

``` csharp
var query = from customer in context.Customers
            join order in context.Orders 
            on customer.Id equals order.CustomerId into joinedGroup
            from joined in joinedGroup.DefaultIfEmpty()
            select new 
            {
                CustomerName = customer.Name,
                OrderNumber = joined?.OrderNumber ?? "N/A"
            };
```

クエリ構文はいいですが、やはりメソッド構文は積極的に使いたいとは思えない構造になっています。

#### After

LeftJoinがファーストクラスのLINQメソッドに採用されたことで、
簡潔に記述できるようになりました。

``` csharp
var query = context.Customers
    .LeftJoin(
        context.Orders,              // 結合するテーブル
        customer => customer.Id,     // 左側のキー
        order => order.CustomerId,   // 右側のキー
        (customer, order) => new     // 結果のプロジェクション
        { 
            CustomerName = customer.Name,
            OrderNumber = order?.OrderNumber ?? "N/A"
        });
```

同様にRightJoinも実装されています。

``` csharp
var query = context.Reviews
    .RightJoin(
        context.Products,
        review => review.ProductId,
        product => product.Id,
        (review, product) => new 
        {
            ProductId = product.Id,
            ProductName = product.Name,
            ReviewRating = review?.Rating ?? 0
        });
```

#### 注意事項

メソッド構文ではLeft,RightJoinが採用されたことで簡素化が進みましたが、
クエリ構文には採用されていません。

このためクエリ構文で書いている場合には、注意してください。

おそらくですが、クエリ構文で書いている人が多いと思うのでこの点だけは残念だと感じる点でした。
ただ、基本的な記載をメソッド構文で書いていて、複雑なケースのときのクエリ構文としていたのを
メソッド構文に統一できる可能性があるのはうれしく思います。

### ベクターデータ型のサポート

ベクターデータは埋め込みを保存することで類似性検索を効率的に実行できるため、
セマンティック検索やRAGなどのAIワークロードを支援するとされています。

ベクターは最適化されたバイナリ形式で保存されますが、利便性のためJSON配列として公開されます。

#### 対応環境

SQL Server2025以降か、Azure SQL Databeaseでのみ利用可能となっている。

#### 利用方法

エンティティクラスの定義は以下のようになる

``` csharp
using Microsoft.Data.SqlTypes;
using System.ComponentModel.DataAnnotations.Schema;

public class Blog
{
    public int Id { get; set; }
    public string Name { get; set; }
    
    [Column(TypeName = "vector(1536)")]
    public SqlVector<float> Embedding { get; set; }
}
```

類似性検索の実行は

``` csharp
// ユーザークエリをベクトル化
var queryEmbedding = await embeddingGenerator
    .GenerateVectorAsync("検索したいクエリ");
var sqlVector = new SqlVector<float>(queryEmbedding);

// 最も類似した上位3件のブログを取得
var topSimilarBlogs = await context.Blogs
    .OrderBy(b => EF.Functions.VectorDistance("cosine", b.Embedding, sqlVector))
    .Take(3)
    .ToListAsync();
```

といった感じで実行でき、ベクターとして扱うことで他の検索なども行うことができる。

時代の流れに乗り遅れないため対応にも感じるが、必要なことに感じる。複合代入演算子の対応もふくめると狙いがあると感じる。

### ExecuteUpdateの簡素化

#### Before

そんなに以前から使いが手が悪いわけではない。

``` csharp
// 単一プロパティの更新
await context.Blogs
    .Where(b => b.Rating < 3)
    .ExecuteUpdateAsync(setters => 
        setters.SetProperty(b => b.IsVisible, false));

// 複数プロパティの更新
await context.Books
    .Where(b => b.PublishedOn.Year < 2018)
    .ExecuteUpdateAsync(s => s
        .SetProperty(b => b.Title, b => b.Title + " (Classic)")
        .SetProperty(b => b.UpdatedAt, DateTime.UtcNow));
```

だが、条件付更新の際には式ツリーを手動構築する必要があった。

``` csharp
// EF Core 7-9での条件付き更新 - 非常に複雑
Expression<Func<SetPropertyCalls<Blog>, SetPropertyCalls<Blog>>> setters = 
    s => s.SetProperty(b => b.Views, 8);

if (nameChanged) 
{
    // 式ツリーを手動で構築
    var blogParameter = Expression.Parameter(typeof(Blog), "b");
    
    setters = Expression.Lambda<Func<SetPropertyCalls<Blog>, SetPropertyCalls<Blog>>>(
        Expression.Call(
            instance: setters.Body,
            methodName: nameof(SetPropertyCalls<Blog>.SetProperty),
            typeArguments: [typeof(string)],
            arguments: [
                Expression.Lambda<Func<Blog, string>>(
                    Expression.Property(blogParameter, nameof(Blog.Name)), 
                    blogParameter),
                Expression.Constant("foo")
            ]
        ),
        setters.Parameters
    );
}

await context.Blogs.ExecuteUpdateAsync(setters);
```

#### After

列セッターに式ツリー引数ではなく、非式引数を受け入れるようになりました。
その結果コードが非常にシンプルになる。

``` csharp
// EF Core 10での条件付き更新 - シンプルで読みやすい
await context.Blogs.ExecuteUpdateAsync(s => 
{
    s.SetProperty(b => b.Views, 8);
    
    if (nameChanged) 
    {
        s.SetProperty(b => b.Name, "foo");
    }
});
```

### Complex Typesのサポート強化

#### Complex Typesとは

キー値によって識別または追跡されず、エンティティ型の一部として定義される必要がある構造化オブジェクトである。

以前にはOwned Entity Typesがあり、こちらはキー値をもつことが特徴である。

``` csharp
// Owned Entity Typesでの問題例
Address address = new() 
{ 
    Street = "123 Main St", 
    City = "Springfield", 
    PostCode = "12345" 
};

Company company = new() 
{ 
    Name = "Contoso", 
    Address = address 
};

company.Offices.Add(new Office 
{ 
    Name = "HQ", 
    Location = address,      // 同じインスタンスを再利用
    Headquarters = address   // エラー発生！
});

context.Companies.Add(company);
await context.SaveChangesAsync();  
// ❌ InvalidOperationException: オーナーへの参照なしでOwned Entityを保存できない
```

この問題をComplex Typesでは

``` csharp
// Complex Typesでは動作する
Address address = new() 
{ 
    Street = "123 Main St", 
    City = "Springfield", 
    PostCode = "12345" 
};

var customer = new Customer 
{ 
    Name = "Willow", 
    Address = address 
};

customer.Orders.Add(new Order 
{ 
    Contents = "Tesco Tasty Treats",
    BillingAddress = customer.Address,   // ✅ 同じインスタンスを使用可能
    ShippingAddress = customer.Address   // ✅ 問題なし
});

await context.SaveChangesAsync();
```

Complex TypesはJSON列とのマッピングの相性がよいとされていて

``` csharp
public class Order
{
    public int Id { get; set; }
    public string OrderNumber { get; set; }
    
    // JSONとしてマッピング
    public OrderDetails Details { get; set; }
}

public class OrderDetails  // Complex Type
{
    public Address ShippingAddress { get; set; }
    public Address BillingAddress { get; set; }
    public List<string> Notes { get; set; }
}

// 設定
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Order>()
        .ComplexProperty(o => o.Details)
        .ToJson();
}
```

### Azure Cosmos DBの改善

EF Core 10のCosmos DBプロバイダーは、AI/ML時代のモダンアプリケーションに最適化されていました。

#### フルテキスト検索のサポート

Azure Cosmos DBはフルテキスト検索のサポートしており、効率的なテキスト検索と検索クエリに対するドキュメントの関連性評価を可能にしている。

**モデル設定：**

``` csharp
public class Blog 
{
    public string Id { get; set; }
    public string Contents { get; set; }
}

public class BloggingContext : DbContext 
{
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Blog>(b => 
        {
            // プロパティでフルテキスト検索を有効化
            b.Property(x => x.Contents).EnableFullTextSearch();
            
            // フルテキストインデックスを作成
            b.HasIndex(x => x.Contents).IsFullTextIndex();
        });
    }
}
```

**クエリでの使用**

``` csharp
// 基本的なフルテキスト検索
var results = await context.Blogs
    .Where(x => EF.Functions.FullTextContains(x.Contents, "cosmos"))
    .ToListAsync();

// 複数のキーワード検索
var multiResults = await context.Blogs
    .Where(x => EF.Functions.FullTextContainsAll(x.Contents, 
        new[] { "database", "performance" }))
    .ToListAsync();

// いずれかのキーワード
var anyResults = await context.Blogs
    .Where(x => EF.Functions.FullTextContainsAny(x.Contents, 
        new[] { "EF Core", "Entity Framework" }))
    .ToListAsync();
```

#### ハイブリット検索

EF10はRRF（Reciprocal Rank Fusion）を使用してハイブリッド検索を可能にし、フルテキスト検索とベクター検索を組み合わせてより正確な結果を提供する。

**RRFの利点:**

- フルテキスト検索とベクター検索の結果を統合
- AI駆動のアプリケーションで関連性と検索最適化に有用
- より正確な検索結果のランキング

``` csharp
public class Article 
{
    public string Id { get; set; }
    public string Title { get; set; }
    public string Content { get; set; }
    public float[] Embedding { get; set; }  // ベクター埋め込み
}

// ハイブリッド検索の設定
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Article>(e => 
    {
        // フルテキスト検索
        e.Property(x => x.Content).EnableFullTextSearch();
        e.HasIndex(x => x.Content).IsFullTextIndex();
        
        // ベクター検索
        e.Property(x => x.Embedding)
            .IsVectorProperty(VectorDistanceFunction.Cosine, dimensions: 1536);
        e.HasIndex(x => x.Embedding).IsVectorIndex();
    });
}

// ハイブリッド検索クエリ
float[] queryVector = GetQueryEmbedding("machine learning best practices");

var hybridResults = await context.Articles
    .OrderBy(x => EF.Functions.Rrf(
        // フルテキスト検索スコア
        EF.Functions.FullTextScore(x.Content, "machine learning"),
        // ベクター類似度
        EF.Functions.VectorDistance(x.Embedding, queryVector)
    ))
    .Take(10)
    .ToListAsync();
```

#### ベクター検索の成熟

EF Core9では実験的機能であったが、EF Core10で正式サポートされ機能も拡張された。

owned reference entitiesで定義されたベクタープロパティを持つコンテナを生成できるようになった。

``` csharp
// 1. Owned reference entitiesでのベクタープロパティサポート
public class Blog 
{
    public string Id { get; set; }
    public string Name { get; set; }
    public BlogMetadata Metadata { get; set; }  // Owned entity
}

public class BlogMetadata 
{
    public string Description { get; set; }
    public float[] DescriptionEmbedding { get; set; }  // ✅ EF10で対応
}

protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Blog>(b => 
    {
        b.OwnsOne(x => x.Metadata, metadata => 
        {
            // Owned entity内でベクター設定可能
            metadata.Property(m => m.DescriptionEmbedding)
                .IsVectorProperty(VectorDistanceFunction.Cosine, dimensions: 768);
            metadata.HasIndex(m => m.DescriptionEmbedding)
                .IsVectorIndex();
        });
    });
}

// 2. APIの改名（より明確に）
// 旧: .HasVectorProperty() / .HasVectorIndex()
// 新: .IsVectorProperty() / .IsVectorIndex()

// 3. クエリでの使用
var queryEmbedding = new float[] { /* ... */ };

var results = await context.Blogs
    .OrderBy(b => EF.Functions.VectorDistance(
        b.Metadata.DescriptionEmbedding, 
        queryEmbedding))
    .Take(5)
    .ToListAsync();
```

他にも細かい点でも改善されている。

### パラメータ化されたコレクションの処理最適化

SQLでいうとIN句を使う検索などを行う際の話。

``` csharp
// よくあるクエリパターン
int[] ids = [1, 2, 3];
var blogs = await context.Blogs
    .Where(b => ids.Contains(b.Id))
    .ToListAsync();
```

EF Core8ではこの場合のSQLはJSON配列を使用してパラメータ化されたコレクション翻訳を変更されていた。

``` sql
-- EF Core 8-9で生成されるSQL
@__ids_0='[1,2,3]'

SELECT [b].[Id], [b].[Name]
FROM [Blogs] AS [b]
WHERE [b].[Id] IN (
    SELECT [i].[value]
    FROM OPENJSON(@__ids_0) WITH ([value] int '$') AS [i]
)
```

と書かれる。
メリットとしては

- 単一SQLでキャッシュ可能である
- プランキャッシュ汚染が解消されている。

デメリットしては

- データベースクエリプランナーがコレクションのカーディナリティ（長さ）に関する重要な情報を失い、少数または大量の要素に対して適切に機能するプランを選択できる
- パフォーマンス回帰の可能性がある

EF Core10では、パラメータ化されたコレクションは複数のスカラーパラメータを使用して翻訳されるようになった。

``` csharp
-- EF Core 10で生成されるSQL
SELECT [b].[Id], [b].[Name]
FROM [Blogs] AS [b]
WHERE [b].[Id] IN (@ids1, @ids2, @ids3)
```

特徴としては以下のものが挙げられる

- カーディナリティ情報の提供: クエリプランナーが要素数を把握できる
- パラメータ化: 値が変わってもSQL構造は同じ
- バケット化/パディング: SQL変動を削減

#### バケット化の仕組み

あくまでも仕組みを理解するための例として見てほしいが、
SQLのパラメータ数が実際のコレクションの数と毎回一致するのではなく、
一定数用意するようになることでSQLを流用している。

``` csharp
// 8個の値を持つコレクション
int[] ids = [1, 2, 3, 4, 5, 6, 7, 8];

var blogs = await context.Blogs
    .Where(b => ids.Contains(b.Id))
    .ToListAsync();

// 生成されるSQL（10個のパラメータにパディング）
// SELECT [b].[Id], [b].[Name]
// FROM [Blogs] AS [b]
// WHERE [b].[Id] IN (@ids1, @ids2, @ids3, @ids4, @ids5, 
//                    @ids6, @ids7, @ids8, @ids9, @ids10)
// 
// @ids9と@ids10は@ids8と同じ値を含む
```

この方式によって、サンプルのSQLならコレクションが10個までなら同じプランで実行されるようになり、コレクション数にプランが左右されにくくなる。

制御方法を設定することが可能で、グローバル設定なら

``` csharp
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder.UseSqlServer("<CONNECTION_STRING>", options =>
    {
        // オプション1: 複数パラメータ（デフォルト、EF10）
        options.UseParameterizedCollectionMode(
            ParameterTranslationMode.MultipleParameters);
        
        // オプション2: JSON配列パラメータ（EF8-9の動作）
        options.UseParameterizedCollectionMode(
            ParameterTranslationMode.Parameter);
        
        // オプション3: 定数インライン化（EF7以前の動作）
        options.UseParameterizedCollectionMode(
            ParameterTranslationMode.Constant);
    });
}
```

クエリごとの制御も可能で

``` csharp
int[] ids = [1, 2, 3];

// オプション1: 複数パラメータを明示的に指定
var blogs1 = await context.Blogs
    .Where(b => EF.MultipleParameters(ids).Contains(b.Id))
    .ToListAsync();
// SQL: WHERE Id IN (@ids1, @ids2, @ids3)

// オプション2: 単一パラメータ（JSON配列）を強制
var blogs2 = await context.Blogs
    .Where(b => EF.Parameter(ids).Contains(b.Id))
    .ToListAsync();
// SQL: WHERE Id IN (SELECT v FROM OPENJSON(@__ids_0))

// オプション3: 定数を強制
var blogs3 = await context.Blogs
    .Where(b => EF.Constant(ids).Contains(b.Id))
    .ToListAsync();
// SQL: WHERE Id IN (1, 2, 3)
```

なぜ、こういった制御顔存在しているかというと
実がコレクション数によってパフォーマンスに影響があるためである。

| シナリオ | 推奨モード | 理由 |
| ---- | ---- | ---- |
| 小規模コレクション（1-20個）| MultipleParameters |カーディナリティ情報で最適なプラン |
| 中規模コレクション（20-100個）| MultipleParameters | バランスが取れている |
| 大規模コレクション（100+個）| Parameter (JSON) | パラメータ数が多すぎる場合の回避 |
| 頻繁に変わるコレクション | MultipleParameters |バケット化でプラン再利用 |
| 固定サイズのコレクション | Constant | 最適なプランが選択される |
| ワンオフクエリ | Constant | プランキャッシュを気にしない |

### SQLログのセキュリティ強化

EF Core 10では、機密ロギングがオフの場合、SQLログ内のリテラル定数値が編集されるようになった。

#### Before

``` csharp
// ユーザーのロールで検索する関数
Task<List<User>> GetUsersByRoles(BlogContext context, string[] roles)
    => context.Users
        .Where(u => roles.Contains(u.Role))
        .ToListAsync();

// 実行
var users = await GetUsersByRoles(context, new[] { "Admin", "SuperUser" });
```

ログ出力の内容は

``` sql
-- 機密情報が露出！
SELECT [u].[Id], [u].[Role]
FROM [Users] AS [u]
WHERE [u].[Role] IN ('Admin', 'SuperUser')
```

#### After

``` sql
-- EF Core 10のログ出力（編集済み）
SELECT [b].[Id], [b].[Role]
FROM [Blogs] AS [b]
WHERE [b].[Role] IN (?, ?)
```

機密情報となる部分が？になった。

本番では特に出したくない要素ではあるので隠されるようになったのは非常大きい。
しかし、開発時は出力してほしいので設定で切り替えられる。

``` csharp
public class AppDbContext : DbContext
{
    private readonly IWebHostEnvironment _env;
    
    public AppDbContext(
        DbContextOptions<AppDbContext> options,
        IWebHostEnvironment env) : base(options)
    {
        _env = env;
    }
    
    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        // デフォルト: 機密データは編集（本番環境で安全）
        optionsBuilder.LogTo(Console.WriteLine, LogLevel.Information);
        
        // 開発環境のみ: 機密データを表示
        if (_env.IsDevelopment())
        {
            optionsBuilder.EnableSensitiveDataLogging();
        }
    }
}
```

#### SQLインジェクション保護の強化

同様の観点でSQLインジェクションに対しても一部対応された。

``` csharp
public class ProductService
{
    private readonly AppDbContext _context;
    
    // ケース1: 安全なパラメータ化クエリ
    public async Task<List<Product>> SearchProductsSafe(string searchTerm)
    {
        // EF Coreが自動的にパラメータ化
        var products = await _context.Products
            .Where(p => p.Name.Contains(searchTerm))
            .ToListAsync();
        
        // ログ（EF Core 10）: 
        // WHERE [Name] LIKE '%' + @__searchTerm_0 + '%'
        // ← searchTermの値は編集される
        
        return products;
    }
    
    // ケース2: FromSqlRaw（注意が必要）
    public async Task<List<Product>> SearchProductsWithRawSql(string category)
    {
        // ✅ 安全: パラメータ化されている
        var products = await _context.Products
            .FromSqlRaw(
                "SELECT * FROM Products WHERE Category = {0}", 
                category)
            .ToListAsync();
        
        // ログ（EF Core 10）:
        // SELECT * FROM Products WHERE Category = ?
        // ← categoryの値は編集される
        
        return products;
    }
    
    // ⚠️ 危険: 文字列補間（使用しないでください）
    public async Task<List<Product>> SearchProductsUnsafe(string category)
    {
        // ❌ SQL Injectionの脆弱性あり
        // 絶対に使用しないでください！
        var products = await _context.Products
            .FromSqlRaw($"SELECT * FROM Products WHERE Category = '{category}'")
            .ToListAsync();
        
        return products;
    }
}
```

#### 設定のおすすめ

本番、開発で以下のように使い分けれればよいと思います。

**本番**

``` csharp
// ✅ 推奨: 本番環境
services.AddDbContext<AppDbContext>(options =>
{
    options.UseSqlServer(connectionString);
    
    // 機密データロギングは無効（デフォルト）
    // options.EnableSensitiveDataLogging(); ← 呼び出さない
    
    // 最小限のロギング
    options.LogTo(
        logger.LogInformation, 
        LogLevel.Warning); // 警告以上のみ
});
```

**開発**

``` csharp
// ✅ 推奨: 開発環境
services.AddDbContext<AppDbContext>(options =>
{
    options.UseSqlServer(connectionString);
    
    // 詳細なデバッグ情報
    options.EnableSensitiveDataLogging();
    options.EnableDetailedErrors();
    
    // すべてのログを出力
    options.LogTo(
        Console.WriteLine, 
        LogLevel.Debug);
});
```

### SQL Server2025の新しいJSON型のネイティブサポート

SQL Server2022以前は

``` sql
CREATE TABLE [Blogs] (
    [Id] int NOT NULL,
    [Name] nvarchar(max),
    [Tags] nvarchar(max),  -- ← JSONは文字列として保存
    [Details] nvarchar(max) -- ← JSONは文字列として保存
);
```

SQL Server 2025 / Azure SQL Database（EF Core 10）:
SQL Serverは数バージョンにわたってJSON機能を含んでいたが、データ自体はプレーンなテキスト列に保存されていた。新しいデータ型は大幅な効率改善とJSONとの安全な対話方法を提供する。

``` sql
CREATE TABLE [Blogs] (
    [Id] int NOT NULL,
    [Name] nvarchar(max),
    [Tags] json,       -- ← 新しいネイティブJSON型
    [Details] json     -- ← バイナリ形式で効率的に保存
);
```

#### EF Core10での自動サポート

EF 10でUseAzureSql()または互換性レベル170以上で設定すると、EFは自動的に新しいJSONデータ型を使用する。

``` csharp
// モデル定義
public class Blog
{
    public int Id { get; set; }
    public string Name { get; set; }
    
    // プリミティブコレクション → JSON列にマップ
    public string[] Tags { get; set; }
    
    // Complex Type → JSON列にマップ
    public required BlogDetails Details { get; set; }
}

public class BlogDetails
{
    public string? Description { get; set; }
    public int Viewers { get; set; }
}

// DbContext設定
public class BlogContext : DbContext
{
    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        // オプション1: Azure SQL Database
        optionsBuilder.UseAzureSql("<connection_string>");
        // → 自動的にjson型を使用
        
        // オプション2: SQL Server 2025（互換性レベル170）
        optionsBuilder.UseSqlServer(
            "<connection_string>",
            options => options.UseCompatibilityLevel(170));
        // → json型を使用
        
        // オプション3: SQL Server 2022以前（互換性レベル160）
        optionsBuilder.UseSqlServer(
            "<connection_string>",
            options => options.UseCompatibilityLevel(160));
        // → nvarchar(max)を使用（従来通り）
    }
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Blog>(entity =>
        {
            // Tagsは自動的にJSON列にマップされる
            
            // DetailsをComplex TypeとしてJSON列にマップ
            entity.ComplexProperty(b => b.Details);
        });
    }
}
```

Complex Typesとの統合も行われた

``` csharp
public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; }
    
    // Complex Typesを使用
    public required Address ShippingAddress { get; set; }
    public Address? BillingAddress { get; set; }
}

public class Address
{
    public required string Street { get; set; }
    public required string City { get; set; }
    public required string State { get; set; }
    public required string ZipCode { get; set; }
}

// モデル設定
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Customer>(entity =>
    {
        // 各AddressをJSON列にマップ
        entity.ComplexProperty(c => c.ShippingAddress);
        entity.ComplexProperty(c => c.BillingAddress);
    });
}
```

生成されるテーブルは以下のもの

``` sql
CREATE TABLE [Customers] (
    [Id] int NOT NULL IDENTITY,
    [Name] nvarchar(max) NOT NULL,
    [ShippingAddress] json NOT NULL,
    [BillingAddress] json NULL,
    CONSTRAINT [PK_Customers] PRIMARY KEY ([Id])
);
```

#### JSON列の部分更新（ExecuteUpdate）

EF10はExecuteUpdate内でJSON列とそのプロパティを参照できるようになり、リレーショナルデータベース内のドキュメントモデル化されたデータの効率的な一括更新が可能になった。

``` csharp
public class Blog
{
    public int Id { get; set; }
    public BlogDetails Details { get; set; }
}

public class BlogDetails
{
    public string Title { get; set; }
    public int Views { get; set; }
    public DateTime LastUpdated { get; set; }
}

protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Blog>()
        .ComplexProperty(b => b.Details, bd => bd.ToJson());
}

// JSON列の部分更新
await context.Blogs
    .ExecuteUpdateAsync(s => 
        s.SetProperty(b => b.Details.Views, b => b.Details.Views + 1));
```

生成されるSQLは

``` sql
-- 新しいmodify関数を使用（効率的）
UPDATE [b]
SET [Details].modify('$.Views', JSON_VALUE([b].[Details], '$.Views') + 1)
FROM [Blogs] AS [b]
```

以前は

``` sql
-- JSON_MODIFY関数を使用
UPDATE [b]
SET [b].[Details] = JSON_MODIFY([b].[Details], '$.Views', JSON_VALUE([b].[Details], '$.Views') + 1)
FROM [Blogs] AS [b]
```
