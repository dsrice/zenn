---
title: "OAuthとOIDC"
---

## はじめに

OAuthですが、正確には認証プロトコルではなく、**認可プロトコル**になります。
SAMLとは目的が異なっています。

SAML（2002-2005）の目的：
「従業員が複数の社内システムに一度のログインでアクセスしたい」
→ エンタープライズSSO（シングルサインオン）
→ 認証の問題

OAuth（2007-2012）の目的：
「パスワードを渡さずに、第三者アプリに自分のデータへのアクセス権を与えたい」
→ API連携・権限委譲
→ 認可の問題

といった感じに正確には目的が異なっています。
似たような問題なのですが、サービスを使わせたいのと自分のデータへのアクセス権を与えたいというのは認証、認可の視点からは大きく異なる問題です。

実際にはOAuth2.0の認可と認証を合わせたOIDC(OpenID Connect)が作られています。
その点も踏まえて以下の内容を読んでください。

## OAuthの歴史

### 2000年代中期の課題

以下のシナリオみたいなケースが増加していました。

```
シナリオ：写真共有サービスで印刷注文

従来の方法：
User → 写真共有サイトのID/パスワードを印刷サービスに渡す
     → 印刷サービスが写真共有サイトにログイン
     → 写真をダウンロード

問題点：
❌ パスワードを第三者に渡す必要がある
❌ 印刷サービスがアカウントを完全に制御できる
❌ パスワード変更時に連携が切れる
❌ アクセス権限の細かい制御ができない
```

実例としてもX（旧: Twitter）においてもTwitterクライアントアプリが多数登場したためにクライアントアプリにパスワードを渡していたためにセキュリティリスクが深刻化しています。

### OAuth1.0(2007年12月)

先ほどの問題を解決するために
「パスワードを共有せずに、第３者アプリに限定的なアクセス権を付与」
する方式としてOAuth1.0が登場しました。

主要な概念：

- Resource Owner（リソース所有者）：ユーザー
- Client（クライアント）：サードパーティアプリ
- Resource Server（リソースサーバー）：APIサーバー
- Authorization Server（認可サーバー）：権限付与を行うサーバー

しかし、課題として以下の内容もありました。

1. 実装の複雑さ
   - 暗号署名の生成が必須
   - リクエストごとに署名計算
   - モバイルアプリでの実装が困難

2. HTTPSへの依存度
   - 署名があってもHTTPSが推奨
   - ならば署名なしでHTTPSのみの方がシンプル

3. Webブラウザ以外への対応
   - モバイルアプリ
   - デスクトップアプリ
   - IoTデバイス

### OAuth2.0(2012年)

OAuth1.0での課題を改善すべきOAuth2.0が生まれました。

1. シンプルな実装
   - 暗号署名が不要（HTTPS必須）
   - ベアラートークン方式

2. 複数のグラントタイプ
   - Authorization Code Grant（Webアプリ）
   - Implicit Grant（SPAアプリ）※非推奨
   - Resource Owner Password Credentials（モバイル）※非推奨
   - Client Credentials（サーバー間通信）

3. スコープによる権限制御
   - 細かいアクセス権限の指定
   - ユーザーが承認する範囲を明示

4. リフレッシュトークン
   - アクセストークンの更新
   - 長期的なアクセス許可

#### 主な用途

1. API連携

```
例：カレンダーアプリとタスク管理アプリの連携

User
  ↓ (1) タスク管理アプリを使用
Task Management App
  ↓ (2) カレンダーへのアクセスを要求
Authorization Server (Google)
  ↓ (3) ユーザーに権限付与を確認
User が承認
  ↓ (4) アクセストークン発行
Task Management App
  ↓ (5) アクセストークンでAPI呼び出し
Calendar API (Google Calendar)
  ↓ (6) タスクの期日をカレンダーに追加
```

2. ソーシャルログイン

```
例：「Googleでログイン」

User → アプリにアクセス
     → 「Googleでログイン」をクリック
     → Google認可サーバーにリダイレクト
     → Googleで認証
     → アプリにリダイレクト（認可コード付き）
     → アプリが認可コードをトークンに交換
     → アプリがGoogleのUserInfo APIを呼び出し
     → ユーザー情報を取得してログイン完了
```

Googleの話は少々誤解を招くのですが、認可サーバーにリダイレクトしたときに未認証だから認証処理を実施しています。
認証処理が成功したので認可サーバーがアクセス元のアプリのアクセス権を付与するための認可コードを発行。
アプリでは、発行された認可トークンからトークンに変換してアプリ側でユーザー情報を取得しています。

## 一般的な処理フロー

一般的な処理フローは以下のようになります。

```
┌─────────┐                                  ┌──────────────┐
│ User    │                                  │Authorization │
│(Resource│                                  │Server        │
│ Owner)  │                                  │(Google/      │
└────┬────┘                                  │ GitHub etc)  │
     │                                       └──────┬───────┘
     │                                              │
     │  (A) ユーザーがクライアントアプリにアクセス      │
     │                                              │
┌────▼──────────────────────┐                       │
│  Client Application       │                       │
│  (Webアプリ/モバイルアプリ)│                        │
└────┬──────────────────────┘                       │
     │                                              │
     │ (B) 認可エンドポイントにリダイレクト            │
     │     GET /authorize?                          │
     │         response_type=code&                  │
     │         client_id=xxx&                       │
     │         redirect_uri=https://app/callback&   │
     │         scope=read:user&                     │
     │         state=random123                      │
     ├─────────────────────────────────────────────►│
     │                                              │
     │                                              │ (C) ユーザー認証
     │                                              │     とスコープ承認
     │                                              │
     │ (D) 認可コード付きでリダイレクト                │
     │     https://app/callback?                    │
     │         code=AUTH_CODE&                      │
     │         state=random123                      │
     │◄─────────────────────────────────────────────┤
     │                                              │
     │ (E) 認可コードをアクセストークンに交換          │
     │     POST /token                              │
     │         grant_type=authorization_code&       │
     │         code=AUTH_CODE&                      │
     │         redirect_uri=https://app/callback&   │
     │         client_id=xxx&                       │
     │         client_secret=yyy                    │
     ├─────────────────────────────────────────────►│
     │                                              │
     │ (F) アクセストークン＋リフレッシュトークン      │
     │     {                                        │
     │       "access_token": "...",                 │
     │       "token_type": "Bearer",                │
     │       "expires_in": 3600,                    │
     │       "refresh_token": "...",                │
     │       "scope": "read:user"                   │
     │     }                                        │
     │◄─────────────────────────────────────────────┤
     │                                              │
                                                    │
┌──────────────────┐                                │
│Resource Server   │                                │
│(API Server)      │                                │
└────┬─────────────┘                                │
     │                                              │
     │ (G) アクセストークンでAPIリクエスト            │
     │     GET /api/user                            │
     │     Authorization: Bearer ACCESS_TOKEN       │
     │◄─────────────────────────────────────────────┤
     │                                              │
     │ (H) ユーザーデータを返却                       │
     │     { "name": "Alice", "email": "..." }      │
     ├─────────────────────────────────────────────►│
     │                                              │
```

### ステップA-B: 認可リクエスト

``` http
GET /authorize?
  response_type=code&
  client_id=CLIENT_ID&
  redirect_uri=https://myapp.com/callback&
  scope=read:user+read:email&
  state=xyz123
Host: authorization-server.com
```

パラメータ：
response_type=code: 認可コードフロー
client_id: アプリの識別子
redirect_uri: 認可コード受け取りURL
scope: 要求する権限（スペース区切り）
state: CSRF対策のランダム文字列

### ステップC: ユーザー認証と同意

``` html
<!-- 認可サーバーが表示する画面 -->
<h1>MyApp がアクセス許可を求めています</h1>
<p>以下の権限を許可しますか？</p>
<ul>
  <li>✓ プロフィール情報の読み取り</li>
  <li>✓ メールアドレスの読み取り</li>
</ul>
<button>許可する</button>
<button>拒否する</button>
```

### ステップD: 認可コードの発行

``` http
HTTP/1.1 302 Found
Location: https://myapp.com/callback?
  code=AUTHORIZATION_CODE&
  state=xyz123
```

### ステップE: トークン交換

``` http
POST /token HTTP/1.1
Host: authorization-server.com
Content-Type: application/x-www-form-urlencoded
```

パラメータ：
grant_type=authorization_code&
code=AUTHORIZATION_CODE&
redirect_uri=https://myapp.com/callback&
client_id=CLIENT_ID&
client_secret=CLIENT_SECRET

### ステップF: アクセストークンの取得

``` json
{
  "access_token": "eyJhbGciOiJSUzI1NiIs...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "def50200a8f7b...",
  "scope": "read:user read:email"
}
```

### ステップG-H: API呼び出し

``` http
GET /api/user HTTP/1.1
Host: api-server.com
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
json{
  "id": 12345,
  "name": "Alice",
  "email": "alice@example.com"
}
```

## SAMLとの比較

### 基本的な違い

| 項目 | SAML 2.0 | OAuth 2.0 |
|------|----------|-----------|
| **登場年** | 2005年 | 2012年 |
| **主な目的** | 認証（Authentication）+ SSO | 認可（Authorization） |
| **データ形式** | XML | JSON（主流） |
| **トークン形式** | SAMLアサーション（XML） | アクセストークン（JWT等） |
| **想定環境** | エンタープライズ | Web/モバイル/API |
| **実装の複雑さ** | 高い | 中程度 |
| **仕様のサイズ** | 350ページ以上 | 約80ページ（コア） |

### ユースケースの違い

#### SAML 2.0が適している場面

エンタープライズSSO

- 社内の複数システムへのシングルサインオン
- 従業員が複数のWebアプリに一度のログインでアクセス
  
B2B連携

- パートナー企業とのシステム連携
- サプライチェーン管理システム
  
既存システムの統合

- レガシーシステムとの互換性
- 長期運用される企業システム

実例：

- 従業員がGoogle Workspace、Salesforce、ServiceNowに
  会社のActive Directoryで一度ログインしてアクセス
- 製造業が取引先企業の受発注システムに
  自社の認証でアクセス

#### OAuth 2.0が適している場面

API連携

- サードパーティアプリへの限定的なアクセス許可
- マイクロサービス間の認可
  
ソーシャルログイン

- 「Googleでログイン」「GitHubでログイン」
- モバイルアプリでのログイン
  
細かい権限制御

- スコープによる細かいアクセス制御
- 読み取り専用、書き込み権限など

実例：

- カレンダーアプリがGoogleカレンダーの予定を読み取る
- CI/CDツールがGitHubリポジトリにアクセス
- モバイルアプリで「Googleでログイン」

## OIDC

冒頭にいいましたが、OAuthは認可プロトコルです。
そのために。「誰が」アクセスしているのかの部分は標準化していません。
そのため、各サービスが独自の方法でユーザー情報を取得しています。

- Google: /userinfo
- GitHub: /user
- Facebook: /me

統一されたインターフェイスがないためユーザー情報の取得という最終目的の情報が個別対応する必要がでてきていました。
そこで、OAuthに認証レイヤーを追加したOIDCが生まれました。

追加要素としては、

- IDトークン（JWT形式）
- UserInfoエンドポイント（標準化）
- 標準的なクレーム（sub, name, email等）

### SAMLとOIDCの比較

| 項目 | SAML 2.0 | OpenID Connect |
|------|----------|----------------|
| ベース技術 | 独自仕様 | OAuth 2.0 + JWT |
| データ形式 | XML | JSON |
| 学習曲線 | 急 | 緩やか |
| モバイル対応 | 困難 | 容易 |
| 将来性 | 安定（変化少） | 進化中 |
| エンタープライズ | ◎ | ○（増加中） |

## まとめ

SAMLとOIDCはおそらく今の認証方式としてはこの２つのどちらかもしくは両方が採用されているといったケースが多いと思います。
サービス開発において、どの方式が採用するべきなのかというの規模が決める問題ではあります。しかし、事前に特定のサービスに紐づくような認証、認可処理にしないでおくことであとでどちらも取り入れることが可能だと思います。
アカウントに関する情報をほかサービスと共有したくないという考え方もあるのでそもそもこれらの認証方式を使わないという選択肢もあると思います。
大事なのは今必要な方式であるかの判断をできるように方式の基本知識が必要なんだと思います。
次のチャプターからはSAMLについて細かく書いていきます。
