---
title: "プロトコル"
---

プロトコルとは、IdpとSP間でアサーションを交換するためのリクエスト/レスポンスの形式を定義したものです。

主要なプロトコルとして

- Authentication Request Protocol
  - 認証要求とレスポンス
- Single Logout Protocol
  - ログアウト要求とレスポンス
- Artifact Resolution Protocol
  - Artifactからアサーションを取得
- Name Identifier Management Protocol
  - NameIDの管理
- Assertion Query and Request Protocol
  - アサーションの問い合わせ

がありますが、よく使われるのは認証とログアウトの２つのみとなっており、残りの３つに関しては実装があまり確認できないものになっている。（SAMLの仕様書には記載はある）

## AutheRequest(認証要求)

SPからIdpへの認証要求になります。

``` xml
<samlp:AuthnRequest xmlns:samlp="urn:oasis:names:tc:SAML:2.0:protocol"
                    xmlns:saml="urn:oasis:names:tc:SAML:2.0:assertion"
                    ID="_cea6c4f3b49c7a8e2d9f3b6c8e4a7f9d"
                    Version="2.0"
                    IssueInstant="2024-12-15T10:00:00Z"
                    Destination="https://idp.example.com/sso"
                    AssertionConsumerServiceURL="https://sp.example.com/acs"
                    ProtocolBinding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST">
  
  <!-- 要求元SP -->
  <saml:Issuer>https://sp.example.com</saml:Issuer>
  
  <!-- NameIDポリシー -->
  <samlp:NameIDPolicy 
      Format="urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress"
      AllowCreate="true"/>
  
  <!-- 要求する認証コンテキスト -->
  <samlp:RequestedAuthnContext Comparison="exact">
    <saml:AuthnContextClassRef>
      urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport
    </saml:AuthnContextClassRef>
  </samlp:RequestedAuthnContext>
  
</samlp:AuthnRequest>
```

**主要な属性：**

| 属性 | 必須 | 説明 |
|--------|------|-----|
|ID|✓|リクエストの一意識別子|
|Version|✓|SAMLバージョン（2.0）|
|IssueInstant|✓|発行時刻Destination-IdPのSSOエンドポイント|
|AssertionConsumerServiceURL|-|レスポンスの送信先URL|
|ProtocolBinding|-|使用するバインディング|
|ForceAuthn|-|true: 強制再認証|
|IsPassive|-|true: ユーザー操作なし|

**NameIDPolicy:**
Response時の受け取りたいNameIDのPolicy要求になります。

``` xml
<samlp:NameIDPolicy 
    Format="urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress"
    AllowCreate="true"
    SPNameQualifier="https://sp.example.com"/>

Format: 希望するNameID形式
AllowCreate: IdPが新しいNameIDを作成してよいか
SPNameQualifier: SPの識別子
```

**RequestedAuthnContext:**
認証方式要求になります。
以下の内容なら多要素認証で認証したいという内容になります。

``` xml
<samlp:RequestedAuthnContext Comparison="exact">
  <saml:AuthnContextClassRef>
    urn:oasis:names:tc:SAML:2.0:ac:classes:MultiFactor
  </saml:AuthnContextClassRef>
</samlp:RequestedAuthnContext>

Comparison値:
- exact: 完全一致
- minimum: 指定以上のセキュリティレベル
- maximum: 指定以下のセキュリティレベル
- better: より強力な認証
```

## Response (認証レスポンス)

IdpからSPへの認証レスポンスになります。

``` xml
<samlp:Response xmlns:samlp="urn:oasis:names:tc:SAML:2.0:protocol"
                xmlns:saml="urn:oasis:names:tc:SAML:2.0:assertion"
                ID="_8e8dc5f69a98cc4c1ff3427e5ce34606fd672f91e6"
                Version="2.0"
                IssueInstant="2024-12-15T10:00:05Z"
                Destination="https://sp.example.com/acs"
                InResponseTo="_cea6c4f3b49c7a8e2d9f3b6c8e4a7f9d">
  
  <!-- 発行者 -->
  <saml:Issuer>https://idp.example.com</saml:Issuer>
  
  <!-- ステータス -->
  <samlp:Status>
    <samlp:StatusCode Value="urn:oasis:names:tc:SAML:2.0:status:Success"/>
  </samlp:Status>
  
  <!-- アサーション -->
  <saml:Assertion>
    <!-- 前述のアサーション内容 -->
  </saml:Assertion>
  
</samlp:Response>
```

**Status（ステータスコード）:**

```
Success（成功）:
urn:oasis:names:tc:SAML:2.0:status:Success

Requester（リクエストエラー）:
urn:oasis:names:tc:SAML:2.0:status:Requester
└─ AuthnFailed: 認証失敗
└─ InvalidAttrNameOrValue: 無効な属性
└─ InvalidNameIDPolicy: 無効なNameIDポリシー
└─ RequestDenied: リクエスト拒否
└─ UnsupportedBinding: 非対応バインディング

Responder（レスポンダーエラー）:
urn:oasis:names:tc:SAML:2.0:status:Responder
└─ NoPassive: パッシブ認証不可
└─ NoAuthnContext: 認証コンテキストなし
└─ UnsupportedBinding: 非対応バインディング

VersionMismatch（バージョン不一致）:
urn:oasis:names:tc:SAML:2.0:status:VersionMismatch
```

エラーレスポンスの例は以下のようになります。
ここで使っていないStatusDetailという要素もあるのですが、あまり使われないのでここでは省略します。

``` xml
<samlp:Response>
  <saml:Issuer>https://idp.example.com</saml:Issuer>
  <samlp:Status>
    <samlp:StatusCode Value="urn:oasis:names:tc:SAML:2.0:status:Requester">
      <samlp:StatusCode Value="urn:oasis:names:tc:SAML:2.0:status:AuthnFailed"/>
    </samlp:StatusCode>
    <samlp:StatusMessage>
      Invalid username or password
    </samlp:StatusMessage>
  </samlp:Status>
</samlp:Response>
```

## LogoutRequest(ログアウト要求)

``` xml
<samlp:LogoutRequest xmlns:samlp="urn:oasis:names:tc:SAML:2.0:protocol"
                     xmlns:saml="urn:oasis:names:tc:SAML:2.0:assertion"
                     ID="_135ad2e0cd0f454bb4d9f8ca3ca0a1c2"
                     Version="2.0"
                     IssueInstant="2024-12-15T15:00:00Z"
                     Destination="https://idp.example.com/slo">
  
  <saml:Issuer>https://sp.example.com</saml:Issuer>
  
  <!-- ログアウトするユーザー -->
  <saml:NameID 
      Format="urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress"
      SPNameQualifier="https://sp.example.com">
    alice@example.com
  </saml:NameID>
  
  <!-- セッション識別子 -->
  <samlp:SessionIndex>
    _be9967abd904ddcae3c0eb4189adbe3f71e327cf93
  </samlp:SessionIndex>
  
</samlp:LogoutRequest>
```

## LogoutResponse(ログアウトレスポンス)

``` xml
<samlp:LogoutResponse xmlns:samlp="urn:oasis:names:tc:SAML:2.0:protocol"
                      xmlns:saml="urn:oasis:names:tc:SAML:2.0:assertion"
                      ID="_6c3a4f8d64744c7babfc2f1b3c8e4f5a"
                      Version="2.0"
                      IssueInstant="2024-12-15T15:00:05Z"
                      Destination="https://sp.example.com/slo"
                      InResponseTo="_135ad2e0cd0f454bb4d9f8ca3ca0a1c2">
  
  <saml:Issuer>https://idp.example.com</saml:Issuer>
  
  <samlp:Status>
    <samlp:StatusCode Value="urn:oasis:names:tc:SAML:2.0:status:Success"/>
  </samlp:Status>
  
</samlp:LogoutResponse>
```

## Artifact Resolution Protocol（アーティファクト解決プロトコル）

HTTP POST Bindingの代わりに参照(Artifact)だけを送る方式になります。

Artifact Bindingは以下のようになります。

```
IdP → ブラウザ → SP
     (Artifact参照のみ送信)
     約100バイト程度
     
SP → IdP（バックチャネル/SOAP）
   (Artifactでアサーション要求)
   
IdP → SP
   (アサーション本体を返却)
```

**シーケンス図**

```
User         SP          IdP
 |            |           |
 | ①認証済み  |           |
 |            |           |
 | ②Artifact送信          |
 | (HTTP Redirect)        |
 |<----------------------|
 |                        |
 | ③Artifact転送          |
 |---------->|            |
 |           |            |
 |           | ④ArtifactResolve |
 |           |  (SOAP/バックチャネル)
 |           |----------->|
 |           |            |
 |           |            | ⑤アサーション
 |           |            |   取得・検証
 |           |            |
 |           | ⑥ArtifactResponse |
 |           |  (アサーション本体) |
 |           |<-----------|
 |           |            |
 |           | ⑦検証      |
 |           |            |
 | ⑧ログイン完了           |
 |<----------|            |
 |           |            |
```

**Artifactの例**

```
SAMLart=AAQAADWNEw5VT47wcO4zX%2FiEzMmFQvGknDfws2ZtqSGdkNSbsW1cmVR0bzU%3D

構造：
- TypeCode（2バイト）：0x0004
- EndpointIndex（2バイト）：0x0000
- SourceID（20バイト）：IdPの識別子のSHA-1ハッシュ
- MessageHandle（20バイト）：ランダム値
```

### ArtifactResolveリクエスト（SOAP）

``` xml
<SOAP-ENV:Envelope 
    xmlns:SOAP-ENV="http://schemas.xmlsoap.org/soap/envelope/">
  <SOAP-ENV:Body>
    <samlp:ArtifactResolve 
        xmlns:samlp="urn:oasis:names:tc:SAML:2.0:protocol"
        ID="_request123"
        Version="2.0"
        IssueInstant="2024-12-15T10:00:00Z">
      
      <saml:Issuer>https://sp.example.com</saml:Issuer>
      
      <!-- Artifact -->
      <samlp:Artifact>
        AAQAADWNEw5VT47wcO4zX/iEzMmFQvGknDfws2ZtqSGdkNSbsW1cmVR0bzU=
      </samlp:Artifact>
      
    </samlp:ArtifactResolve>
  </SOAP-ENV:Body>
</SOAP-ENV:Envelope>
```

### ArtifactResponseレスポンス

``` xml
<SOAP-ENV:Envelope>
  <SOAP-ENV:Body>
    <samlp:ArtifactResponse
        ID="_response456"
        InResponseTo="_request123"
        Version="2.0"
        IssueInstant="2024-12-15T10:00:01Z">
      
      <saml:Issuer>https://idp.example.com</saml:Issuer>
      
      <samlp:Status>
        <samlp:StatusCode Value="urn:oasis:names:tc:SAML:2.0:status:Success"/>
      </samlp:Status>
      
      <!-- 実際のアサーション -->
      <samlp:Response>
        <saml:Assertion>
          <!-- アサーション本体 -->
        </saml:Assertion>
      </samlp:Response>
      
    </samlp:ArtifactResponse>
  </SOAP-ENV:Body>
</SOAP-ENV:Envelope>
```

### メリット・デメリット

**メリット：**

```
✓ ブラウザを経由するデータが小さい
  - URLの長さ制限を回避
  - ネットワーク効率が良い

✓ セキュリティが高い
  - アサーション本体がブラウザを経由しない
  - バックチャネル（サーバー間直接通信）で取得
  - 中間者攻撃のリスクが低い

✓ 大きなアサーションも送信可能
  - 多数の属性を含む場合
```

**デメリット：**

```
❌ 実装が複雑
  - バックチャネル通信のインフラが必要
  - SOAP対応が必要
  - 2段階の通信（Artifact送信 + 解決）

❌ パフォーマンス
  - 追加のラウンドトリップが発生
  - IdPへの追加リクエストが必要

❌ インフラ要件
  - SPからIdPへの直接通信が必要
  - ファイアウォール設定が複雑
```

### なぜあまり使われないのか

```
現実的な理由：

1. HTTP POST Bindingで十分
   - 通常のアサーションは数KB程度
   - ブラウザ経由でも問題なし

2. 実装コストが高い
   - SOAP対応が必要
   - バックチャネル通信の実装
   - デバッグが困難

3. インフラ要件
   - SP→IdPの直接接続が必要
   - プライベートネットワーク環境では困難

4. 標準ライブラリのサポートが薄い
   - HTTP POST/Redirectは広くサポート
   - Artifact Bindingは限定的

使用例（非常に稀）：
- 軍事・政府機関の高セキュリティ環境
- 非常に多数の属性を持つアサーション
- ブラウザ経由を避けたい特殊要件
```

インフラ要件が厳しすぎて一般的には使われていないものです。
しかし、高セキュリティ要件には使われているものになります。

## Name Identifier Management Protocol

ユーザーのNameID（識別子）を変更・終了するためのプロトコルです。

### ManageNameIDRequest

``` xml
<samlp:ManageNameIDRequest
    xmlns:samlp="urn:oasis:names:tc:SAML:2.0:protocol"
    ID="_request789"
    Version="2.0"
    IssueInstant="2024-12-15T10:00:00Z"
    Destination="https://idp.example.com/manage">
  
  <saml:Issuer>https://sp.example.com</saml:Issuer>
  
  <!-- 現在のNameID -->
  <saml:NameID 
      Format="urn:oasis:names:tc:SAML:2.0:nameid-format:persistent">
    _old_identifier_abc123
  </saml:NameID>
  
  <!-- 新しいNameID -->
  <samlp:NewID>_new_identifier_xyz789</samlp:NewID>
  
</samlp:ManageNameIDRequest>
```

### ユースケース

```
1. Pseudonymous（仮名）識別子の変更
   - プライバシー保護のため定期的に変更
   - 欧州のGDPR対応など

2. NameID形式の変更
   - Transient → Persistent
   - Email → Persistent

3. NameIDの終了
   - ユーザーがサービスを退会
   - 識別子の無効化
```

### なぜあまり使われないのか

```
理由：

1. NameIDを変更する必要が少ない
   - Persistent NameIDは永続的に使用
   - 変更の必要性がほとんどない

2. 変更が必要な場合の代替手段
   - 新しいアサーションで上書き
   - ユーザーを再登録
   - より単純な方法で対応可能

3. 実装の複雑さ
   - このプロトコルをサポートするIdP/SPが少ない
   - 標準ライブラリでもサポート外が多い

4. プライバシー要件が厳しい場合のみ有用
   - 一般的なエンタープライズSSOでは不要

実装例：
- ほぼ見たことがない
- 仕様書に記載されているが実装は稀
```

## Assertion Query and Request Protocol

SPからIdPに対して、**アサーションを問い合わせる（プル型）**プロトコルです。

### Assertion ID Request（アサーションID要求）

``` xml
<samlp:AssertionIDRequest
    ID="_request_abc"
    Version="2.0"
    IssueInstant="2024-12-15T10:00:00Z">
  
  <saml:Issuer>https://sp.example.com</saml:Issuer>
  
  <!-- 取得したいアサーションのID -->
  <saml:AssertionIDRef>_assertion_xyz789</saml:AssertionIDRef>
  
</samlp:AssertionIDRequest>
```

### Attribute Query（属性問い合わせ）

``` xml
<samlp:AttributeQuery
    ID="_query_123"
    Version="2.0"
    IssueInstant="2024-12-15T10:00:00Z">
  
  <saml:Issuer>https://sp.example.com</saml:Issuer>
  
  <!-- 誰の属性を問い合わせるか -->
  <saml:Subject>
    <saml:NameID>alice@example.com</saml:NameID>
  </saml:Subject>
  
  <!-- どの属性が欲しいか（省略可） -->
  <saml:Attribute Name="email"/>
  <saml:Attribute Name="department"/>
  
</samlp:AttributeQuery>
```

### Authorization Decision Query（認可決定問い合わせ）

``` xml
<samlp:AuthzDecisionQuery
    ID="_authz_query_456"
    Version="2.0"
    IssueInstant="2024-12-15T10:00:00Z"
    Resource="https://sp.example.com/documents/secret.pdf">
  
  <saml:Issuer>https://sp.example.com</saml:Issuer>
  
  <saml:Subject>
    <saml:NameID>alice@example.com</saml:NameID>
  </saml:Subject>
  
  <!-- どのアクションが許可されるか -->
  <samlp:Action>Read</samlp:Action>
  
</samlp:AuthzDecisionQuery>
```

### ユースケース（理論上）

```
1. リアルタイムの属性取得
   - ユーザーの部署が変更された
   - 最新の情報を取得したい

2. Just-In-Time（JIT）の認可決定
   - リソースアクセス時に毎回確認
   - 動的な権限管理

3. 属性の差分更新
   - すべての属性を送らず必要なものだけ取得
```

### なぜあまり使われないのか

```
理由：

1. プッシュ型が主流
   - アサーションを送る方式（通常のSAMLフロー）
   - ログイン時に必要な情報をすべて含める
   - 追加の問い合わせが不要

2. パフォーマンス
   - 毎回IdPに問い合わせると遅い
   - キャッシュや定期更新の方が効率的

3. 実装の複雑さ
   - バックチャネル通信が必要
   - 認証・認可の仕組みが別途必要
   - SPからIdPへの直接接続が必要

4. セキュリティ考慮
   - SPがIdPに問い合わせる権限管理
   - どのSPがどの属性を取得できるか
   - 複雑な権限設定が必要

5. 代替手段の存在
   - アサーションに必要な情報を全部含める
   - 有効期限を短くして鮮度を保つ
   - 別のAPI（REST API等）で属性取得

実際の実装：
- Attribute Queryのサポート：一部のIdPのみ
- Authorization Decision Query：ほぼ見ない
- 使用例：非常に特殊なケースのみ
```

### 実際の属性更新方法

```
一般的なアプローチ：

方法1：再認証
User → SP → IdP（再認証）
     → 新しいアサーション
     → 最新の属性を取得

方法2：定期的な同期
SP → IdP（SCIM API等）
  → ユーザー情報の一括同期
  → SAMLとは別の仕組み

方法3：短い有効期限
アサーションの有効期限を短く設定（5分等）
→ 頻繁に再認証
→ 常に最新の情報

方法4：Webhookによる通知
IdP → SP（属性変更の通知）
  → SPがキャッシュを更新
  → SAMLとは別の仕組み
```
