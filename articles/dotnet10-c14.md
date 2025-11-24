---
title: ".NET10がリリースされた[C#14編]"
emoji: "🕌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["dotNET10","csharp", "AI", "AIAssistant"]
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

今回はC#14編です。

## 主要な新機能

C#14で追加された機能は以下のものがあります。

- Extension Members
- field キーワード
- Null-Conditional Assignment
- User-Defined Compound Assignment
- Span<T> First-Class Support
- Partial Constructors & Events
- nameof改善

### Extension Members

#### そもそもExtension Membersって？

既存のクラスに対してメソッドだけを追加できる機能としてあった。
対象クラスをthisを使って宣言して固有メソッドを増やす使い方ができました。

``` csharp
public static class StringExtensions
{
    // thisキーワードで拡張対象を指定
    public static bool IsNullOrEmpty(this string value)
    {
        return string.IsNullOrEmpty(value);
    }
    
    public static string Truncate(this string value, int maxLength)
    {
        if (string.IsNullOrEmpty(value) || value.Length <= maxLength)
            return value;
        return value.Substring(0, maxLength);
    }
}

// 使用例
string text = "Hello World";
bool isEmpty = text.IsNullOrEmpty();  // インスタンスメソッドのように呼べる
string short = text.Truncate(5);      // "Hello"

```

ただ、制約が非常に多く、

- メソッドした追加できない
- プロパティは追加できない
- 静的メンバーは追加できない
- 演算子オーバーロードは追加でき兄
- 各メソッドで毎回`this`を書く必要がある

といった具体で少々使い勝手が悪かった。

#### どういった進化をしたのか？

変更点を箇条書きすると

- 拡張プロパティが設定できるようになった
- 静的拡張メンバーが設定できるようになった
- 演算子オーバーロードも定義できるようになった

新しい構文で書くと

``` charp
public static class StringExtensions
{
    // extension ブロックで対象型を一度だけ指定
    extension(string value)
    {
        // ✅ 拡張プロパティ
        public bool IsNullOrEmpty => string.IsNullOrEmpty(value);
        
        // ✅ 拡張メソッド（thisが不要）
        public string Truncate(int maxLength)
        {
            if (string.IsNullOrEmpty(value) || value.Length <= maxLength)
                return value;
            return value[..maxLength];
        }
        
        public int WordCount() => value?.Split(' ').Length ?? 0;
        
        // ✅ 静的拡張メンバー（型自体の拡張）
        public static bool IsValidEmail(string email)
        {
            return email?.Contains("@") ?? false;
        }
    }
}

// 使用例
string text = "Hello World";

// インスタンスメンバーとして使用
bool isEmpty = text.IsNullOrEmpty;     // プロパティ！
string short = text.Truncate(5);
int words = text.WordCount();

// 静的メンバーとして使用
bool valid = string.IsValidEmail("test@example.com");

```

といった感じにかなり使いやすくなりました。

ライブラリのクラスなどに紐づくメソッドなどを追加したいと感じるケースは少なくないので今後はうまく扱っていきたいと思える機能でした。
今までは、継承する方法をとることが多かく、コンストラクタなどをうまく合わせるといったことが必要で手間だと思っていました。

### field キーワード

#### Before

自動実装プロパティから、カスタムget/setアクセサーへスムーズに移行可能にする機能である。
今までの問題点は、

- バリデーションの内容とわず、フィールド宣言が必要
- 自動実装プロパティから書き直しが必要
- コードが冗長になる

``` charp

// 最初は自動実装プロパティで十分
public class Student
{
    public string Name { get; set; }
}

// しかし、後で「空文字を許可したくない」という要件が追加
public class Student
{
    // 😫 バッキングフィールドを明示的に宣言が必要
    private string _name = "";
    
    public string Name
    {
        get => _name;
        set => _name = string.IsNullOrWhiteSpace(value) 
            ? throw new ArgumentException("名前は必須です") 
            : value;
    }
}

```

個人的な今までの仕様だと
やり方は把握していましたが、やりたい手法ではありませんでした。
フィールド宣言が必要になるということがネックすぎてクラスがわかりにくくなるという印象が強くfieldキーワードではなく、バリデーション時に用意する方針が主でした。

#### After

新しくなると以下のようになって取り扱いやすくなりました。
フィールド宣言がふようになったのは大きいと感じています。

``` charp
public class Student
{
    // getは自動実装のまま、setだけカスタマイズ
    public string Name
    {
        get;  // 自動実装
        set => field = string.IsNullOrWhiteSpace(value) 
            ? throw new ArgumentException("名前は必須です") 
            : value;
    }
}
```

積極的に使うかといわれると個人的にはたぶん使わないです。
fieldキーワードの欠点としては、返り値が設定できないので
プロパティセッターでのバリデーションはExceptionをthorowすることになります。
つまり、このセッターを使い際はExceptionが起きる可能性が生まれ、
使用時にはtry-catchが必要になってしまいます。
これでは、インデントが深くなっていく傾向にあるし、変数のセットくらいで例外が起きるわけがないという実装をされてしまうリスクを考えると利用は危険なリスクが伴うと感じてしまいます。
これを回避するとなるとメソッドを用意する、先ほどのExtention Membersで紐づけておくような形になるのかと思います。

### Null-Conditional Assignment

null条件メンバーアクセス演算子を左辺で使用可能となった。

#### Before 

今まではnull確認が必要なケースが散在していた。
クラスのnull確認も含まれるが、プロパティがull許可されているときのnull確認の必要性は大事だった。

```charp
// ネストしたnullチェック
Student student = GetStudent();

if (student != null && student.Address != null)
{
    student.Address.PostalCode = "100-0001";
}

// 配列要素の代入
Examination[] examinations = GetExaminations();

if (examinations != null && examinations.Length > 0)
{
    examinations[0] = newExamination;
}

```

#### After

左辺でnull条件演算子である?.と?[]を使用可能になったことでシンプルになった。

```charp
// nullでなければ代入（超シンプル！）
Customer customer = GetCustomer();
customer?.Order = GetCurrentOrder();

// ネストしたプロパティもOK
Student student = GetStudent();
student?.Address?.PostalCode = "100-0001";

// 配列要素もOK
Examination[] examinations = GetExaminations();
examinations?[0] = newExamination;
```

これは正直ありがたい変更点だと思っている。
既存の確認処理をわざわざ無理に変えるほどではないが、
コードがシンプルになることは間違いないので積極的に取り入れていきたい。
仕方ないけどnullのときに別の処理にするケースもあるから思ったより利用頻度は高くないかも・・・・

### User-Defined Compound Assignment

復号代入演算子をオーバーロード可能になり、方での不要なコピーを削減できる。

#### Before

```charp
public struct Vector3
{
    public double X, Y, Z;
    
    public Vector3(double x, double y, double z)
    {
        X = x; Y = y; Z = z;
    }
    
    // +演算子をオーバーロード
    public static Vector3 operator +(Vector3 a, Vector3 b)
    {
        return new Vector3(a.X + b.X, a.Y + b.Y, a.Z + b.Z);
    }
}

// 使用例
Vector3 position = new Vector3(10, 20, 30);
Vector3 velocity = new Vector3(1, 2, 3);

// += は自動的に以下のように展開される
position += velocity;
// ↓ コンパイラが自動変換
// Vector3 temp = position + velocity;  // 新しいインスタンス作成（コピー）
// position = temp;                     // 代入（コピー）
```

コンパイラでの実際の処理の問題ではあるが、ものとメソッドがnewしているので
違和感のない動きであった。

#### After

できなかった複合代入演算子をオーバーロードできるようになったので
newする必要がなくなっている。

```charp
public struct Vector3
{
    public double X, Y, Z;
    
    public Vector3(double x, double y, double z)
    {
        X = x; Y = y; Z = z;
    }
    
    // 従来の+演算子
    public static Vector3 operator +(Vector3 a, Vector3 b)
    {
        return new Vector3(a.X + b.X, a.Y + b.Y, a.Z + b.Z);
    }
    
    // ✨ NEW: +=演算子を直接オーバーロード（インプレース更新）
    public void operator +=(Vector3 other)
    {
        X += other.X;  // コピーなし！
        Y += other.Y;
        Z += other.Z;
    }
}

// 使用例
Vector3 position = new Vector3(10, 20, 30);
Vector3 velocity = new Vector3(1, 2, 3);

position += velocity;  // インプレース更新（コピーなし！）
```

これは非常に便利な変更なイメージではある。
今までの実装なら、オーバーロードができないから計算メソッドを用意していたと思うが、これならメソッドいらなくなるからコードの可読性があがると考えられる。

### Span<T> First-Class Support

そもそも使ったことがないのでSpan<T>について調べるところから

#### Span<T>とは

連続したメモリ領域への参照を表す型になっている。
このメモリは、配列やスタックメモリ、アンマネージドメモリなどどこにあるメモリでも
統一的に使うことができる。
・・・・よくわからないので実装例を調べて考える。

``` charp
// 問題1: 部分配列を処理するとコピーが発生
byte[] imageData = new byte[1_000_000];  // 1MBの画像データ

// 最初の100KBだけ処理したい
byte[] subset = new byte[100_000];
Array.Copy(imageData, 0, subset, 100_000);  // 😫 100KBコピー
ProcessData(subset);

// 問題2: 文字列の部分処理でもコピー
string studentData = "PT12345678|山田太郎|1980-01-01";
string studentId = studentData.Substring(0, 10);  // 😫 新しい文字列作成
string name = studentData.Substring(11, 12);      // 😫 また作成

// 問題3: スタックメモリを使いたいが型が異なる
unsafe
{
    byte* buffer = stackalloc byte[256];  // ポインタ（unsafeが必要）
    // 通常のメソッドに渡せない
}
```

調べてみて納得する3例をかいた。
問題１はたしかにやりたいことはに対して一度別の変数にいれないと
一括で処理しづらいからメモリ的には2重にある。

問題２は、文字列から各要素を取り出すさいには似たようなことをする。
配列にするにしても文字列と配列の２つが存在するようになるので
いいたことは同じ

問題３に関してはスタックメモリを積極的につかうことがないが、
よく聞く話な気がする。

```charp
// ✨ 解決1: 部分配列を参照のみで処理
byte[] imageData = new byte[1_000_000];
Span<byte> subset = imageData.AsSpan(0, 100_000);  // コピーなし！
ProcessData(subset);

// ✨ 解決2: 文字列の部分処理もコピーなし
string studentData = "PT12345678|山田太郎|1980-01-01";
ReadOnlySpan<char> studentIdSpan = studentData.AsSpan(0, 10);  // コピーなし
ReadOnlySpan<char> nameSpan = studentData.AsSpan(11, 12);      // コピーなし

// ✨ 解決3: スタックメモリを安全に使える
Span<byte> buffer = stackalloc byte[256];  // unsafe不要！
ProcessData(buffer);  // 同じメソッドに渡せる
```

Spanを活用することで、メモリ参照することができるのでメモリを効率的に扱えるようになる。

負荷がかかったときにこういった処理の効率性は大事になってくるのでメモリ不足などのときの解消方法としては大事な機能だった。
これらを踏まえて本題になる変更点に移る。

#### Before

問題点としては以下のものがあった。

- 明示的な変換宣言が必要であること
- メソッドの呼び出しが冗長になりやす
- 型パラメータの推論が効かない


``` charp
public void ProcessData(byte[] data)
{
    // 配列をSpanに変換（明示的）
    Span<byte> span = data.AsSpan();
    ProcessSpan(span);
}

public void ProcessSpan(Span<byte> span)
{
    // 処理
}

// メソッド呼び出しが冗長
byte[] imageData = LoadImage();
ProcessData(imageData);

// または
ProcessSpan(imageData.AsSpan());  // 明示的な変換が必要
```

#### After

C#14から暗黙的な変換がサポートされるようになったので自然に扱えるようになった。

``` charp
public void ProcessData(Span<byte> span)
{
    // 処理
}

// 配列を直接渡せる（暗黙的変換）
byte[] imageData = LoadImage();
ProcessData(imageData);  // ✨ AsSpan()不要！

// ReadOnlySpan<T>も同様
public void ReadData(ReadOnlySpan<byte> span) { }

ReadData(imageData);  // ✨ 自動変換！
```

たしかにBeforeとAfterを比較すると扱いやすくなった印象。
画像などの大きいデータを扱いときなどうまく活用したいところではある。
CSVの処理にも活用できたりするので、サイズの大きいファイルなどには効果的だと思う。

### Partial Constructors & Events

この項目はC#13での手が入っているのでいろいろ補足する。

#### C#13で実装されたこと

C#13でpartial propertiesとindexersが追加されたこと、以下のようなClassが作れるようになった。
これによりクラス定義を複数ファイルに分割する際に柔軟性が向上した。

```charp
// Student.cs
public partial class Student
{
    public string StudentId { get; set; }
    public string Name { get; set; }
}

// Student.Generated.cs（自動生成コード）
public partial class Student
{
    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }
}
```

できたこととしては

- プロパティの分割
- メソッドの分割
- フィールドの分割
  
できないこととしては

- コンストラクタ宣言と実装の分割
- イベントの宣言と実装の分割

自動生成コードで常に必要なプロパティだけ別に宣言
追加分は人の手で追加するといったケースではよい機能である。
EF Coreのモデルクラスとかだと作成日などを原則いれるようにしているDB設計は
珍しくないと思うので、使いこなせると便利かもしれない。

#### Partial Constructors

コンストラクタが抱えていた、重複宣言を解消している。

``` charp
// Student.cs（手書きコード）
public partial class Student
{
    public string StudentId { get; set; }
    public string Name { get; set; }
    
    // コンストラクタの宣言のみ
    partial Student();
    partial Student(string StudentId, string name);
}

// Student.Generated.cs（自動生成コード）
public partial class Student
{
    // コンストラクタの実装
    partial Student()
    {
        CreatedAt = DateTime.UtcNow;
        AuditLog = new List<string>();
        InitializeDefaults();
    }
    
    partial Student(string StudentId, string name)
    {
        StudentId = StudentId;
        Name = name;
        CreatedAt = DateTime.UtcNow;
        AuditLog = new List<string>();
        InitializeDefaults();
    }
    
    private DateTime CreatedAt { get; set; }
    private List<string> AuditLog { get; set; }
    
    private void InitializeDefaults()
    {
        // 初期化ロジック
    }
}
```

#### Partial Events

Eventに関して抱えていた問題を解消している。

``` charp
// Student.cs（手書きコード）
public partial class Student
{
    public string StudentId { get; set; }
    public string Name { get; set; }
    
    // イベントの宣言のみ
    partial event EventHandler<StudentChangedEventArgs> StudentChanged;
    partial event EventHandler<string> AuditLogAdded;
    
    public void UpdateName(string newName)
    {
        string oldName = Name;
        Name = newName;
        
        // イベント発火（実装は別ファイル）
        StudentChanged?.Invoke(this, 
            new StudentChangedEventArgs(oldName, newName));
    }
}

// Student.Generated.cs（自動生成コード）
public partial class Student
{
    // イベントの実装
    partial event EventHandler<StudentChangedEventArgs> StudentChanged
    {
        add
        {
            _StudentChanged += value;
            AuditLogAdded?.Invoke(this, 
                $"Event handler added: {value.Method.Name}");
        }
        remove
        {
            _StudentChanged -= value;
            AuditLogAdded?.Invoke(this, 
                $"Event handler removed: {value.Method.Name}");
        }
    }
    
    private event EventHandler<StudentChangedEventArgs> _StudentChanged;
    
    // 監査ログイベントの実装
    partial event EventHandler<string> AuditLogAdded
    {
        add => _auditLogAdded += value;
        remove => _auditLogAdded -= value;
    }
    
    private event EventHandler<string> _auditLogAdded;
}
```

### nameof改善

変更点としては、

nameof式で非バインドジェネリック型をサポートした。
それにより型引数を指定せずに型名を取得できるようになった。

なのだが、比較したほうがわかりやすい

#### Before

``` charp
// ジェネリック型の名前を取得したい

// ❌ コンパイルエラー
// string name = nameof(List<>);  
// string name2 = nameof(Dictionary<,>);  // コンパイルエラー

// 😫 型引数に何を指定するか悩む
string name3 = nameof(List<object>);   // "List" (objectを使うのが慣例)
string name4 = nameof(Dictionary<object, object>);  // "Dictionary"

// 😫 typeof を使う回避策
string name5 = typeof(List<>).Name;    // "List`1" (バッククォートが付く)
string name6 = typeof(Dictionary<,>).Name;  // "Dictionary`2"
```

こういった感じで、方引数をしてしないとコンパイルエラーになったりする。
また、typeof().Nameでバッククォートがついていた。

#### After

型指定がふようになったので奇術師としてはすっきりした。

``` charp
// ✨ 型引数なしで名前を取得可能
string name1 = nameof(List<>);           // "List"
string name2 = nameof(Dictionary<,>);    // "Dictionary"
string name3 = nameof(ValueTuple<,,,>);  // "ValueTuple"

// ネストしたジェネリック型も同様
string name4 = nameof(List<Dictionary<,>>);  // "List"
```

個人的な感想としては、そもそもnameof自体をあまりつかったことがない。
過去にSQLBuilderを自作したときにモデルクラスから数値か文字列かなどを判別するときに利用したことがある程度、たぶん普通に実装していたらなかなかないのかもしれない。

## 総括

新機能という点にフォーカスしてまとめてみました。
普段触らない機能について知る機会になって非常に勉強になりました。

