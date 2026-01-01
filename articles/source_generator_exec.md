---
title: "Source Geratorsを使ってみた [テーブルモデル編]"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["csharp"]
published: false
---

## 概要

先日Source Generatorsについて調べてみた。
利用するのにどういった課題があるのか、試しに使ってみることで気づけることがあると思うので触ってみる。

## 環境

- .NET10
- ASP.NET Coreを使ったWeb APIのプロジェクト
- DBはSQLiteでEntityFramework Core
- EntityFramework Coreによるマイグレーションも使う

## 検証内容

どこまで自動生成できることが現実的なのかを見極めたいと思っています。
環境は標準的なASP.NET Coreを使ったAPIサーバーを想定していますので、FE周りはいったん検証対象外とさせてもらいます。
検証結果は個人的な意見が混ざるとは思いますが、ご了承ください。

## DBのテーブルクラスへの対応

### 使わないケース
テーブル設計として、よくあるケースとしては各テーブルに作成日、更新日、削除フラグ、削除日をもつことではないかと思います。（ログテーブルなどではつけないと思いますが）
このケースをを想定する。

Souce Generatorsを使わずにテーブルのモデルクラスを作成すると


``` csharp
public class Product : BaseModel
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
}

public abstract class BaseModel
{
    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }
    public bool IsDeleted { get; set; }
    public DateTime? DeletedAt { get; set; }

        // === ヘルパープロパティ ===
    /// <summary>
    /// エンティティがアクティブ（削除されていない）かどうか
    /// </summary>
    public bool IsActive => !IsDeleted;

    // === ヘルパーメソッド ===
    /// <summary>
    /// エンティティを論理削除としてマーク
    /// </summary>
    public void MarkAsDeleted()
    {
        IsDeleted = true;
        DeletedAt = System.DateTime.UtcNow;
    }

    /// <summary>
    /// 削除マークを解除
    /// </summary>
    public void Restore()
    {
        IsDeleted = false;
        DeletedAt = null;
    }

}
```

といった実装になると見込まれる。
必要なテーブルモデルクラスにBaseModelのクラスを継承することで作成日などの共通カラムを付与する方式である。

気になる点としては継承を利用しているところだと思います。
個人的にはあまり悪という印象はなく、使いどころ、ルールさえしっかりしていれば使うことは否定しません。
何でもかんでも継承したり、継承するものの選定ルールがあるといったケースはあまりよくないかと思っているくらいです。
DBのモデルクラスで共通カラムを外出しで継承する方式は個人的には問題ない使い方であると思っています。

### 使ってみたケース

#### どういった使い方をするか？

細かい話をする前に、まずどういった使い方をしたいかを決めたい。
さきほどのケースを書き換えるととなると毎回継承宣言するという手間はなくしたい。
一方で、継承する、しないで実現していた必要なときに付与するという仕様も残す。
もちろん、BaseModelにかいたヘルパーメソッドも引き継ぎたい。

#### 実現仕様

Souce Generatorsの制約として、追加のみが可能という点があります。
変更や削除といった機能はありません。
そこでpatial classを使います。
手で書くモデルクラスをすべてpatial class宣言に変更して追加する方針とする。
対象とするnamespaceのpatial classだけを対象とするとすべてについてしまうので属性指定することで付与する、しないを選択できるようにする。

#### 実装

Souce Generatorsを使うためにはSouce Generatorsのコードは今実装しているプロジェクトとは別のプロジェクトで記述する必要があります。

Souce Generators用の別プロジェクトを立てることはよいのですが、
このプロジェクトのターゲットフレームワークは「.NET Standard2.0」を使うことです。
最初調べときは馬鹿なと思ったのですが、ハンドブックでも推奨環境として挙げられています。

https://infinum.com/handbook/dotnet/best-practices/source-generators

いろいろ調べると.NET10でも動かすことは可能でした。しかし、困ったことに「ビルドは通るが、InteliSenseが効かない」という現象に見舞われてしまったことです。
これによって、実装上では、IDEが見つからないとエラーを表示してくれるが、ビルドすると成功するという矛盾する現象が起きてしました。
解消するすべがあまり思いつかないので、今回は推奨にしたがって「.NET Standard2.0」で作ります。

用意したGenerator側のコードは

``` csharp
#nullable enable
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp.Syntax;
using System.Collections.Immutable;
using System.Linq;
using System.Text;

namespace SourceAPI.Generators
{
    [Generator]
    public class AuditableEntityGenerator : IIncrementalGenerator
    {
        public void Initialize(IncrementalGeneratorInitializationContext context)
        {
            // [AuditableEntity]属性を持つpartial classをフィルタ
            var classDeclarations = context.SyntaxProvider
                .CreateSyntaxProvider(
                    predicate: static (s, _) => IsCandidateClass(s),
                    transform: static (ctx, _) => GetSemanticTargetForGeneration(ctx))
                .Where(static m => m is not null);

            // 監査フィールドとヘルパーメソッドを生成
            context.RegisterSourceOutput(classDeclarations, static (spc, source) => Execute(source!, spc));
        }

        private static bool IsCandidateClass(SyntaxNode node)
        {
            return node is ClassDeclarationSyntax classDeclaration &&
                   classDeclaration.Modifiers.Any(Microsoft.CodeAnalysis.CSharp.SyntaxKind.PartialKeyword) &&
                   classDeclaration.AttributeLists.Count > 0;
        }

        private static INamedTypeSymbol? GetSemanticTargetForGeneration(GeneratorSyntaxContext context)
        {
            var classDeclaration = (ClassDeclarationSyntax)context.Node;
            var symbol = context.SemanticModel.GetDeclaredSymbol(classDeclaration) as INamedTypeSymbol;

            if (symbol == null)
                return null;

            // [AuditableEntity]属性を持つか確認
            if (!HasAuditableEntityAttribute(symbol))
                return null;

            // SourceAPI.Models.DB namespaceのみを対象にする
            var namespaceName = symbol.ContainingNamespace.ToDisplayString();
            if (namespaceName != "SourceAPI.Models.DB")
                return null;

            return symbol;
        }

        private static bool HasAuditableEntityAttribute(INamedTypeSymbol classSymbol)
        {
            return classSymbol.GetAttributes().Any(attr =>
                attr.AttributeClass?.Name == "AuditableEntityAttribute");
        }

        private static void Execute(INamedTypeSymbol classSymbol, SourceProductionContext context)
        {
            var namespaceName = classSymbol.ContainingNamespace.ToDisplayString();
            var className = classSymbol.Name;

            // partial classで監査フィールドとヘルパーメソッドを生成
            var source = GeneratePartialClass(namespaceName, className);
            context.AddSource($"{className}.g.cs", source);
        }

        private static string GeneratePartialClass(string namespaceName, string className)
        {
            return $@"// <auto-generated/>
#nullable enable

namespace {namespaceName}
{{
    /// <summary>
    /// Source Generator により自動生成されたpartial class
    /// 監査フィールドとヘルパーメソッドを提供
    /// </summary>
    public partial class {className} : SourceAPI.Interfaces.IAuditableEntity
    {{
        // === 監査フィールド ===
        /// <summary>
        /// 作成日時
        /// </summary>
        public System.DateTime CreatedAt {{ get; set; }}

        /// <summary>
        /// 更新日時
        /// </summary>
        public System.DateTime UpdatedAt {{ get; set; }}

        /// <summary>
        /// 論理削除フラグ
        /// </summary>
        public bool IsDeleted {{ get; set; }}

        /// <summary>
        /// 削除日時
        /// </summary>
        public System.DateTime? DeletedAt {{ get; set; }}

        // === ヘルパープロパティ ===
        /// <summary>
        /// エンティティがアクティブ（削除されていない）かどうか
        /// </summary>
        public bool IsActive => !IsDeleted;

        // === ヘルパーメソッド ===
        /// <summary>
        /// エンティティを論理削除としてマーク
        /// </summary>
        public void MarkAsDeleted()
        {{
            IsDeleted = true;
            DeletedAt = System.DateTime.UtcNow;
        }}

        /// <summary>
        /// 削除マークを解除
        /// </summary>
        public void Restore()
        {{
            IsDeleted = false;
            DeletedAt = null;
        }}
    }}
}}
";
        }
    }
}
```

さらに
属性としては、`[AuditableEntity]`を採用したので、Attributeとして

``` csharp
namespace SourceAPI.Attributes;

/// <summary>
/// このクラスに監査フィールドとヘルパーメソッドを自動生成するマーカー属性
/// </summary>
[AttributeUsage(AttributeTargets.Class)]
public class AuditableEntityAttribute : Attribute
{
}
```

そして、EF Coreとのつなぎ込みのためにInterfacesを用意する。

``` csharp
namespace SourceAPI.Interfaces;

/// <summary>
/// 監査可能なエンティティのインターフェース
/// [AuditableEntity]属性を使用すると自動的に実装される
/// </summary>
public interface IAuditableEntity
{
    DateTime CreatedAt { get; set; }
    DateTime UpdatedAt { get; set; }
    bool IsDeleted { get; set; }
    DateTime? DeletedAt { get; set; }
}
```

さらに先ほどのモデルクラスを修正する。

``` csharp
[AuditableEntity]
public partial class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
}
```

DBContextは以下のようになる
``` csharp
using Microsoft.EntityFrameworkCore;
using SourceAPI.Models.DB;
using SourceAPI.Interfaces;

namespace SourceAPI.Data;

public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options)
    {
    }

    public DbSet<Product> Products { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        // IAuditableEntityを実装するすべてのエンティティに監査フィールドとクエリフィルターを設定
        ConfigureAuditableEntity<Product>(modelBuilder);
        ConfigureAuditableEntity<User>(modelBuilder);

        // Product固有の設定
        modelBuilder.Entity<Product>(entity =>
        {
            entity.HasKey(e => e.Id);
            entity.Property(e => e.Name).IsRequired().HasMaxLength(200);
            entity.Property(e => e.Price).HasPrecision(18, 2);
        });

        // シードデータ
        var seedDate = new DateTime(2025, 12, 7, 0, 0, 0, DateTimeKind.Utc);
        modelBuilder.Entity<Product>().HasData(
            new Product { Id = 1, Name = "Laptop", Price = 999.99m, CreatedAt = seedDate, UpdatedAt = seedDate, IsDeleted = false },
            new Product { Id = 2, Name = "Mouse", Price = 29.99m, CreatedAt = seedDate, UpdatedAt = seedDate, IsDeleted = false },
            new Product { Id = 3, Name = "Keyboard", Price = 79.99m, CreatedAt = seedDate, UpdatedAt = seedDate, IsDeleted = false }
        );
    }

    /// <summary>
    /// IAuditableEntityを実装するエンティティの共通設定
    /// </summary>
    private void ConfigureAuditableEntity<TEntity>(ModelBuilder modelBuilder) where TEntity : class, IAuditableEntity
    {
        modelBuilder.Entity<TEntity>(entity =>
        {
            // 監査フィールドの設定
            entity.Property(e => e.CreatedAt).IsRequired();
            entity.Property(e => e.UpdatedAt).IsRequired();
            entity.Property(e => e.IsDeleted).IsRequired().HasDefaultValue(false);
            entity.Property(e => e.DeletedAt);

            // グローバルクエリフィルター: 削除されていないデータのみを取得
            entity.HasQueryFilter(e => !e.IsDeleted);
        });
    }

    public override Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        // IAuditableEntityを実装するすべてのエンティティの監査フィールドを自動設定
        var entries = ChangeTracker.Entries<IAuditableEntity>();

        foreach (var entry in entries)
        {
            if (entry.State == EntityState.Added)
            {
                entry.Entity.CreatedAt = DateTime.UtcNow;
                entry.Entity.UpdatedAt = DateTime.UtcNow;
                entry.Entity.IsDeleted = false;
            }
            else if (entry.State == EntityState.Modified)
            {
                entry.Entity.UpdatedAt = DateTime.UtcNow;
            }
        }

        return base.SaveChangesAsync(cancellationToken);
    }
}
```

実装量を今を最小とするためでなく、長い目でみたときに最小とすることを考えたらこのような実装で落ち着きました。
一番のポイントは属性が付与されたときにDBContextです。
EF Coreにおいて、作成日などは自動で生成されず、開発者にゆだねられます。
それゆえに、付与したときの取り回しもこちらで制御する必要があります。
作成日などの設定処理をどこで取り込むべきなのか？というのは１つの課題ではあります。
今回は、プロパティの制約（必須であるかなど）と、作成、更新時の挙動に関しては属性付与時に統一ルールになるようにしました。
この規模感だけでみたら当初より何倍も実装していますが、サービスを作っていくという方針としては、かなり簡略化していけるものと思っています。
このあたりを対応するためにInterfaceが必要になりました。
カスタム属性の作成という面では不要ですが、EF Coreの挙動も踏まえた結果生まれたものです。


## 感想

Souce Generatorsを使ってテーブルクラスの自動生成をやってみましたが、
今回想定した使い方において、個人的な感想としては
最初からの導入なら問題ないかと思います。
途中から取り入れることは適応範囲によってはリスクだと思いました。
どういった面がリスクに感じたかというと
①カラム追加のみに絞ればそこまでリスクはないとは思う。
②共通カラムに対する制御の統一化は今の実装実態にも関係する部分なのでリスクだと思いました。

この辺りは実態と取り入れたい理由で別れる部分なんだと思いました。
デメリットとしては、Souce Generatorsで補っている部分は、開発者から認知されにくい要素になります。
チームとしての認知が必要になるので闇雲に導入することはチーム内のコーディングルールが増えていくことになるので管理が大変になりそうな印象でした。
作成日など、RDB系統のテーブルならあるよね？くらいなものなら取り入れやすいとは思いました。この辺りはどの言語でも自動生成まわりの課題かなとは感じました。



