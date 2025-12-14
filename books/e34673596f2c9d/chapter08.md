---
title: "SAML認証の全体像"
---

## 全体像

SAML2.0の標準仕様書は以下のURLにあります。

https://wiki.oasis-open.org/security/FrontPage

簡単に構成を説明すると

```
SAML 2.0仕様書（OASIS標準）

├─ Core（コア仕様）
│  └─ アサーション、プロトコルの基本構造
│     約90ページ
│
├─ Bindings（バインディング仕様）
│  └─ HTTPなどのトランスポートプロトコルへのマッピング
│     約70ページ
│
├─ Profiles（プロファイル仕様）
│  └─ 具体的なユースケースでの使用方法
│     約100ページ
│
├─ Metadata（メタデータ仕様）
│  └─ IdP/SPの情報交換フォーマット
│     約50ページ
│
├─ Conformance（適合性仕様）
│  └─ 実装の適合性要件
│     約40ページ
│
└─ その他
   ├─ Authentication Context（認証コンテキスト）
   ├─ Security and Privacy Considerations（セキュリティ考慮事項）
   └─ Glossary（用語集）
```

また、SAMLの階層は以下のように表現できます。

```
┌─────────────────────────────────────┐
│         Applications                │ ← ユースケース
│  (SSO, SLO, Attribute Exchange)     │
├─────────────────────────────────────┤
│           Profiles                  │ ← 実装パターン
│  (Web Browser SSO Profile, etc)     │
├─────────────────────────────────────┤
│          Protocols                  │ ← リクエスト/レスポンス
│  (AuthnRequest, Response, etc)      │
├─────────────────────────────────────┤
│         Assertions                  │ ← 情報の表現
│  (Authentication, Attribute, etc)   │
├─────────────────────────────────────┤
│          Bindings                   │ ← トランスポート
│  (HTTP Redirect, HTTP POST, etc)    │
├─────────────────────────────────────┤
│       Transport Protocol            │ ← 通信プロトコル
│         (HTTP/HTTPS)                │
└─────────────────────────────────────┘
```

## 登場人物と役割

SAML認証には3つの主要な登場人物がいます。

```
┌─────────────┐
│   User      │ ← エンドユーザー（従業員、利用者）
│ (Principal) │
└──────┬──────┘
       │
       │ ①サービスを使いたい
       │
       ↓
┌──────────────────────────┐
│  Service Provider (SP)   │ ← サービス提供者
│                          │    ・Google Workspace
│  役割：                   │    ・Salesforce
│  - サービスの提供         │    ・AWS Console
│  - 認証をIdPに委譲       │    ・社内Webアプリ
│  - アサーションの検証     │
└──────┬───────────────────┘
       │
       │ ②認証依頼
       │ ④アサーション検証
       │
       ↓
┌──────────────────────────┐
│ Identity Provider (IdP)  │ ← ID管理者・認証者
│                          │    ・Okta
│  役割：                   │    ・Azure AD
│  - ユーザー認証          │    ・Auth0
│  - ユーザー情報管理       │    ・Active Directory
│  - アサーション発行       │    ・OneLogin
└──────────────────────────┘
```

### 各役割の詳細

1. User（ユーザー / Principal）

```
- サービスを利用したい人
- ログイン情報（ID/パスワード等）を持つ
- IdPで認証を行う
- ブラウザを通じてSPとIdP間を移動

例：
会社員の田中さんがSalesforceにアクセスしたい
```

2. Service Provider（サービスプロバイダー / SP）

```
役割：
✓ ユーザーが利用するサービスを提供
✓ ユーザー認証はIdPに委譲
✓ IdPから受け取ったアサーションを検証
✓ ユーザーにサービスへのアクセスを許可

責務：
- アサーションの署名検証
- タイムスタンプ検証
- Audience（対象者）の検証
- セッション管理

例：
- Salesforce（CRM）
- Google Workspace（メール、ドライブ）
- AWS Management Console
- 社内の経費精算システム
```

3. Identity Provider（アイデンティティプロバイダー / IdP）

```
役割：
✓ ユーザーの認証情報を管理
✓ ユーザー認証を実施
✓ ユーザー属性情報を保持
✓ SAMLアサーションを発行

責務：
- ユーザーの身元確認（パスワード、MFA等）
- アサーションの生成
- アサーションへの署名
- セッション管理（SSO）

例：
- Azure AD / Entra ID
- Okta
- Active Directory Federation Services (AD FS)
- Google Cloud Identity
- Auth0
```

### エンティティID（Entity ID）

IdPとSPは、それぞれ**エンティティID**という一意の識別子を持ちます。
```
エンティティIDの例：

IdP:
- https://idp.example.com
- urn:example:idp:production
- https://accounts.google.com/o/saml2?idpid=C01234567

SP:
- https://sp.example.com
- urn:amazon:webservices
- https://www.salesforce.com
```

**重要なポイント：**

- エンティティIDは**必ず一意**である必要がある
- 通常はURL形式だが、URN形式でも可
- アサーション内の`Issuer`や`Audience`で使用される
- メタデータ交換時の識別に使用

### 信頼関係の構築

IdPとSPは事前に**信頼関係**を確立する必要があります。

```
信頼関係の構築手順：

1. メタデータ交換
   IdP ←→ SP
   お互いの情報（エンティティID、証明書、エンドポイント等）を交換

2. 証明書の交換
   - IdPの公開鍵証明書をSPに登録
   - SPがアサーションの署名を検証するため

3. 設定の確認
   - エンティティIDの確認
   - エンドポイントURLの確認
   - 属性マッピングの設定

4. テスト
   - 実際にSAML認証が動作するか確認
```

信頼関係を構築する必要性は以下のような攻撃シナリオが想定されるからです。

```
攻撃シナリオ（信頼関係がない場合）：

悪意のIdP → 偽のアサーション生成
           → SPに送信
           → SPが検証せず受け入れ
           → 不正ログイン成功

対策：
SPは「信頼するIdP」のリストを持ち、
その証明書でのみアサーション検証を許可
```

## SAML認証の全体フロー

### SP-initiated フロー（SPから開始）

最も一般的なパターンです。ユーザーがSPのサービスに直接アクセスする場合に使用されます。

#### シーケンス図
```
User             SP                    IdP
 |                |                     |
 | ①アクセス      |                      |
 |--------------->|                     |
 |                |                     |
 |                | ②未認証を検出        |
 |                |                     |
 | ③AuthnRequest  |                     |
 | (リダイレクト)  |                     |
 |<---------------|                     |
 |                                      |
 | ④AuthnRequestを転送                   |
 |------------------------------------->|
 |                                      |
 |                                      | ⑤ユーザー認証
 |                                      | （ログイン画面表示）
 |                                      |
 | ⑥認証情報入力                         |
 |------------------------------------->|
 |                                      |
 |                                      | ⑦認証処理
 |                                      | アサーション生成
 |                                      |
 | ⑧Response + Assertion                |
 | (HTMLフォーム自動POST)                |
 |<-------------------------------------|
 |                                      |
 | ⑨Response転送                         |
 |--------------->|                     |
 |                |                     |
 |                | ⑩アサーション検証     |
 |                | - 署名検証           |
 |                | - タイムスタンプ      |
 |                | - Audience確認       |
 |                |                     |
 |                | ⑪セッション作成      |
 |                |                     |
 | ⑫サービス画面  |                      |
 |<---------------|                     |
 |                |                     |

```

#### 各ステップの詳細

##### ステップ①：ユーザーがSPにアクセス

``` http
GET https://sp.example.com/app/dashboard HTTP/1.1
Host: sp.example.com
```

##### ステップ②：SPが未認証を検出**

```
SPの処理：
1. セッションCookieをチェック
2. セッションが存在しない
3. SAML認証が必要と判断
```

##### ステップ③：AuthnRequestの生成とリダイレクト

```
HTTP/1.1 302 Found
Location: https://idp.example.com/sso?SAMLRequest=<Base64エンコードされたAuthnRequest>
```

AuthRequestについては次チャプターにて補足します。
サンプルとしては

``` xml
<samlp:AuthnRequest 
    xmlns:samlp="urn:oasis:names:tc:SAML:2.0:protocol"
    ID="_request_abc123"
    Version="2.0"
    IssueInstant="2024-12-15T10:00:00Z"
    Destination="https://idp.example.com/sso"
    AssertionConsumerServiceURL="https://sp.example.com/acs">
  
  <saml:Issuer>https://sp.example.com</saml:Issuer>
  
  <samlp:NameIDPolicy 
      Format="urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress"
      AllowCreate="true"/>
  
</samlp:AuthnRequest>
```

##### ステップ④：ブラウザがIdPにリダイレクト

``` http
GET https://idp.example.com/sso?SAMLRequest=PHNhbWxwOkF1dGhuUmVxdWVzdC... HTTP/1.1
Host: idp.example.com
```

##### ステップ⑤-⑥：IdPがユーザー認証

```
IdPの処理：
1. SAMLRequestをデコード・解析
2. ユーザーのセッションをチェック
3. セッションなし → ログイン画面表示
4. ユーザーがID/パスワード入力
5. （オプション）多要素認証
```

認証部分がID＆パスワード＋多要素認証となる書き方をしていますが、
このルールはIdp側の提供次第ですので多少の変化はあります。

##### ステップ⑦：アサーション生成

```
IdPの処理：
1. 認証成功を確認
2. ユーザー情報（属性）を取得
3. SAMLアサーション生成
   - ユーザーID（NameID）
   - 認証方法（AuthnStatement）
   - 属性情報（AttributeStatement）
4. アサーションにデジタル署名
5. Responseメッセージにアサーションを含める
```

##### ステップ⑧：HTMLフォームでSPにPOST

``` html
<!DOCTYPE html>
<html>
<head>
  <title>SAML POST</title>
</head>
<body onload="document.forms[0].submit()">
  <form method="post" action="https://sp.example.com/acs">
    <input type="hidden" name="SAMLResponse" value="PHNhbWxwOlJlc3BvbnNlI..." />
    <input type="hidden" name="RelayState" value="https://sp.example.com/app/dashboard" />
    <noscript>
      <button type="submit">Continue</button>
    </noscript>
  </form>
</body>
</html>
```

SAMLResponseの構成に関しても別チャプターで補足します。
ここではサンプルを掲載します。

``` xml
<samlp:Response 
    ID="_response_xyz789"
    InResponseTo="_request_abc123"
    Version="2.0"
    IssueInstant="2024-12-15T10:00:05Z"
    Destination="https://sp.example.com/acs">
  
  <saml:Issuer>https://idp.example.com</saml:Issuer>
  
  <samlp:Status>
    <samlp:StatusCode Value="urn:oasis:names:tc:SAML:2.0:status:Success"/>
  </samlp:Status>
  
  <saml:Assertion>
    <!-- アサーションの詳細は後述 -->
  </saml:Assertion>
  
</samlp:Response>
```

##### ステップ⑨-⑩：SPでアサーション検証

```
SPの検証処理：

1. Base64デコード
2. XMLパース
3. 署名検証
   ✓ IdPの公開鍵で署名を検証
   ✓ 改ざんされていないことを確認
4. タイムスタンプ検証
   ✓ NotBefore <= 現在時刻 <= NotOnOrAfter
   ✓ 有効期限内か確認
5. InResponseTo検証
   ✓ 元のAuthnRequestのIDと一致するか
   ✓ リプレイ攻撃対策
6. Audience検証
   ✓ 自分（SP）宛てのアサーションか確認
7. Destination検証
   ✓ 正しいエンドポイントか確認
```

##### ステップ⑪：セッション作成

```
SPの処理：
1. アサーションからユーザー情報を抽出
   - NameID（ユーザー識別子）
   - 属性（メール、名前、部署等）
2. ローカルセッションを作成
3. セッションCookieを発行
4. RelayStateで指定されたURLにリダイレクト
```

##### ステップ⑫：サービス画面の表示

``` html
HTTP/1.1 302 Found
Location: https://sp.example.com/app/dashboard
Set-Cookie: session_id=xxx; HttpOnly; Secure; SameSite=Strict
```

### IdP-initiated フロー（IdPから開始）

ユーザーが最初にIdPのポータル画面にアクセスし、そこからSPのサービスを選択する場合に使用されます。

#### シーケンス図
```
User             IdP                   SP
 |                |                     |
 | ①IdPポータルに  |                     |
 |   アクセス      |                     |
 |--------------->|                     |
 |                |                     |
 | ②ログイン画面   |                     |
 |<---------------|                     |
 |                |                     |
 | ③認証情報入力   |                     |
 |--------------->|                     |
 |                |                     |
 |                | ④認証処理           |
 |                |                     |
 | ⑤ポータル画面   |                     |
 | （アプリ一覧）  |                     |
 |<---------------|                     |
 |                |                     |
 | ⑥アプリ選択     |                     |
 | （例：Salesforce）|                  |
 |--------------->|                     |
 |                |                     |
 |                | ⑦アサーション生成   |
 |                | （AuthnRequestなし）|
 |                |                     |
 | ⑧Response + Assertion                |
 | (HTMLフォーム自動POST)               |
 |<---------------|                     |
 |                                      |
 | ⑨Response転送                        |
 |------------------------------------->|
 |                                      |
 |                                      | ⑩検証
 |                                      |
 | ⑪サービス画面                        |
 |<-------------------------------------|
 |                                      |
```

#### SP-initiated との違い

| 項目 | SP-initiated | IdP-initiated |
|------|--------------|---------------|
| **開始点** | SPのURL | IdPのポータル |
| **AuthnRequest** | あり | なし |
| **InResponseTo** | AuthnRequestのID | 空 |
| **RelayState** | 元のURL | アプリ識別子 |
| **セキュリティ** | より安全 | やや弱い |

AuthRequestがないので一見良さそうなのですが、実はセキュリティ懸念があります。

**IdP-initiated のセキュリティ懸念：**

```
問題：
AuthnRequestがないため、以下の検証ができない
- InResponseTo検証（リプレイ攻撃対策が弱い）
- SPが意図したリクエストへの応答か不明

推奨：
可能な限りSP-initiatedフローを使用
IdP-initiatedは必要な場合のみ
```

### RelayState（リレーステート）

両方のフローで使用される重要なパラメータです。

```
目的：
ユーザーが最初にアクセスしようとしたURLを保持

SP-initiated の場合：
User → https://sp.example.com/app/reports
     → SPがRelayStateに保存
     → IdPにリダイレクト
     → IdPが認証後、RelayStateをそのまま返す
     → SPがRelayStateのURLにリダイレクト
     → User が /app/reports にアクセスできる

例：
<input type="hidden" name="RelayState" 
       value="https://sp.example.com/app/reports" />
```

リレーステートはユーザーが実際にアクセスしようとした画面に戻せるので
UX的には重要や機能ですが、なんでも設定できるようにするべきではなく、
セキュリティ面で考慮すべき事項があります。

**セキュリティ考慮事項：**

```
✓ RelayStateの値は検証する
✓ オープンリダイレクトを防ぐ
✓ 同一ドメイン内のURLのみ許可

❌ 悪い例：
   RelayState=https://evil.com
   → 攻撃者のサイトにリダイレクト

✓ 良い例：
   RelayState=/app/reports
   → 相対パスのみ許可
```
