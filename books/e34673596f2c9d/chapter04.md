---
title: "ベーシック認証"
---

## ベーシック認証（Basic Auth）とは？

HHTP/1.0に仕様（RFC1945、1996年）で標準化された認証方式で各認証方式において最もシンプルになっています。
ベーシック認証は、利用したいWebサイトにアクセスしたときにIDとパスワードが要求される方式です。
ログインページは存在せずに、ブラウザ依存の認証ダイヤログが表示されます。

![](/images/book-saml/chrome_basic.png)
*Google Chromeのベーシック認証*
![](/images/book-saml/edge_basic.png)
*Microsoft Edgeのベーシック認証*
![](/images/book-saml/safari_basic.png)
*Safariのベーシック認証*

ベーシック認証はログイン画面が不要の代わりに上記のキャプチャの認証ダイヤログが表示されます。このダイヤログのUIを変更することや認証フローを変更することが難しくなっています。

## 認証フロー

簡単ですが認証フローは以下のようになります。

```
クライアント                    サーバー
    |                              |
    | (1) GET /protected           |
    |----------------------------->|
    |                              |
    |     (2) 401 Unauthorized     |
    |     WWW-Authenticate: Basic  |
    |<-----------------------------|
    |                              |
    | (3) GET /protected           |
    |     Authorization: Basic xxx |
    |----------------------------->|
    |                              |
    |     (4) 200 OK + コンテンツ   |
    |<-----------------------------|
    |                              |
```

### Step1：初回アクセス

``` http
GET /protected/resource HTTP/1.1
Host: example.com
```

### Step2：サーバーが認証を要求

``` http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Basic realm="Restricted Area"
```

- `401 Unauthorized`: 認証が必要
- `WWW-Authenticate: Basic`: ベーシック認証を要求
- `realm`: 保護領域の名前（ブラウザに表示される）

### ステップ3: クライアントが認証情報を送信

クライアント側の処理：

```
1. ユーザー名とパスワードを取得
   username: "user"
   password: "pass123"

2. コロンで連結
   "user:pass123"

3. Base64エンコード
   Base64("user:pass123") = "dXNlcjpwYXNzMTIz"
```

HTTPリクエスト：

``` http
GET /protected/resource HTTP/1.1
Host: example.com
Authorization: Basic dXNlcjpwYXNzMTIz
```

### ステップ4: サーバーが認証情報を検証

サーバー側の処理：

```
1. Authorizationヘッダーから "Basic " 以降を取得
   "dXNlcjpwYXNzMTIz"

2. Base64デコード
   "user:pass123"

3. コロンで分割
   username = "user"
   password = "pass123"

4. 認証情報を検証
   - データベースやファイルと照合
   - パスワードハッシュと比較

5. 検証成功
```

レスポンス：

``` http
HTTP/1.1 200 OK
Content-Type: text/html

<html>保護されたコンテンツ</html>
```

認証失敗の場合

``` http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Basic realm="Restricted Area"
```

## 問題点

問題点は、Base64で暗号化されている点です。
Base64はだれでもデコードすることが可能のため暗号化されているわけではありません。
しかも、この情報をすべてのリクエストで必要になるため常にIDとパスワード情報を有しているリクエストが送信されるため漏洩してしまうリスクが非常に高くなっています。
また、ログアウト機能が用意できない仕様となります。これは、認証情報の持ち回りがIDとパスワードのBase64文字列のため、これさえ作れてしまえば使えなくなるという概念が存在していないからです。ベーシック認証は、サービスに入るというよりアクセス権を得ているという感覚のほうが正しいです。
他には、ログイン画面が存在しないためパスワードを変更する（正確には忘れてしまったときの対応）というのも難しくなっています。

## まとめ

おそらくですが、若い人たちにはベーシック認証を使ったことがない人たちのほうが多いのではないでしょうか？
私も人生で数えるくらいしかこの認証のサービスを触ったことがないですし、携わったときもベーシック認証をやめたいという案件でした。
社内ツールだとしてもベーシック認証を使うことがやめたほうがいいと思います。
ただ、認証の歴史としては知っておくべきことだとは思っています。
この認証フローを聞いてなぜ危険なのか？など問題点を指摘できる力は大切だと思います。
