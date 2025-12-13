---
title: "SAML認証"
---

## SAML認証誕生の背景

2000年代初期のエンタープライズ課題はとしては

1. パスワード疲労
   - 従業員が社内の複数システムで異なるパスワードを管理
   - パスワード忘れによるヘルプデスク負荷の増大
2. 企業間連携の困難さ
   - パートナー企業とのシステム連携
   - サプライチェーン管理
   - 各企業で個別にアカウント作成が必要
3. セキュリティリスク
   - 同じパスワードの使いまわし
   - 退職者のアカウント削除漏れ
   - 個別にアカウントを作っているためのアクセス制限の一元管理ができない

といった様々な問題点を抱えていた。
しかし、既存技術では、

- ベーシック認証：各アプリで個別認証が必要
- セッションベース認証：ドメインをまたげない

といった問題にも悩まされた。
これらの問題を解決するために統一的なSSOソリューションを策定された。
それがOASISによって策定された**SAML**になる

## SAMLの歴史

OASIS（Organization for the Advancement of Structured Information Standards）によって2001年から策定を開始。
参加企業や団体にはIBM、Micosoftなどが参加した。

### SAML1.0 (2002年11月)

#### 目的

- 異なるセキュリティドメイン間での認証・認可情報の交換
- XMLベースの標準化された形式
- Webブラウザベースのシングルサインオン

#### 主な機能

- Authentication Assertion（認証アサーション）
- Attribute Assertion（属性アサーション）
- Authorization Decision Assertion（認可アサーション）

### SAML2.0(2005年3月)

#### 改善点

- SAML 1.1とLiberty Alliance Project IDFFの統合
- より柔軟なプロトコル設計
- Single Logout（SLO）のサポート
- Enhanced Client/Proxy（ECP）プロファイル

#### 現在の状況

- デファクトスタンダードとして広く採用
- 2024年現在も主要なエンタープライズSSO技術
- 後方互換性のため大きな変更なし

現在もSAML認証といえばSAML2.0になっています。
おそらく1.0をサポートしているサーバーは多分少ないと思っています。（調べていないのでおそらくです）

## SAMLの基本概念

細かい話は別チャプターにて話しますが、主要な役割概要は以下のようになっています。

```
┌─────────────────┐
│   User/Client   │  エンドユーザー
└────────┬────────┘
         │
         │ ①アクセス要求
         │
         ↓
┌─────────────────────────┐
│  Service Provider (SP)  │  サービス提供者
│  ・Google Workspace     │  ・ユーザーが利用したいサービス
│  ・Salesforce           │  ・認証をIdPに委譲
│  ・AWS Console          │
└────────┬────────────────┘
         │
         │ ②認証要求
         │ ④アサーション検証
         │
         ↓
┌──────────────────────────┐
│ Identity Provider (IdP)  │  ID管理者
│  ・Okta                  │  ・認証情報を一元管理
│  ・Azure AD              │  ・アサーションを発行
│  ・Auth0                 │  ・ユーザー属性を保持
└──────────────────────────┘
         ↑
         │ ③認証
         │
    ┌────┴──────┐
    │ユーザー情報│
    │データベース│
    └───────────┘
```

主要な役割は大きく分けて３つ

- エンドユーザー
  - サービスを利用する人
- サービス提供者（SP: Service Provider)
  - エンドユーザーが利用したいサービス
  - 認証をIdpに委譲している
- ID管理者（Idp：Identity Provider）
  - 認証情報を一元管理している
  - 一部、ユーザー属性を保持している

フローに関しては
①アクセス要求
　エンドユーザーがサービスを利用しようとしたことを指している
②認証要求
　サービス提供者はサービス利用者がまだ未認証のためID管理者に認証要求をする
③認証
　ID管理者は、エンドユーザーの認証対応をお行う
④アサーション発行
　認証結果となるアサーションを発行して、サービス提供者はこの情報をもって認証済みとしてエンドユーザーにサービスを提供する

### エンタープライズSSOのニーズ

#### Single Sign-On(SSO)

```
従来の認証：
User → App1 ログイン（ID/PW）
     → App2 ログイン（ID/PW）
     → App3 ログイン（ID/PW）

SAMLによるSSO：
User → IdP ログイン（1回のみ）
     → App1 アクセス（自動認証）
     → App2 アクセス（自動認証）
     → App3 アクセス（自動認証）
```

#### フェデレーション（サービス間連携）

```
企業A（製造業）         企業B（サプライヤー）
    │                       │
    IdP ←── 信頼関係 ──→ SP（受発注システム）
    │                       │
    └─ 企業Aの従業員が企業Bのシステムに
       企業Aの認証でアクセス可能
```

#### アイデンティティの一元管理

```
入社：IdPに1アカウント作成 → 全SPに自動的にアクセス可能
異動：IdPで属性変更 → 全SPの権限が自動更新
退社：IdPで無効化 → 全SPへのアクセスが即座に停止
```

### XMLベースの仕様

#### なぜXMLなのか？

2000年代初期の技術環境：

- XMLが企業システムの標準データ形式
- SOAPなどXMLベースのWebサービスが主流
- 既存のXMLツール・ライブラリが豊富
- エンタープライズ向けの標準化が重視された

といった当時の技術環境も左右されています。

XMLの利点としては、

- 階層構造で複雑な情報を表現可能
- XMLスキーマによる厳密な検証
- 名前空間による拡張性
- デジタル署名（XML Signature）の標準サポート

一番最後は認証という重要性から考えたら重要です。
XMLを採用しても細工されたレスポンスで処理が進んでしまったら意味がありません。
ID管理者が発行したものであることを証明することは絶対条件になります。

XMLを採用したことで仕様が重厚になります。

#### エンタープライズ要件の複雑さ

JSON方式が現代的なアプローチですが、レスポンスをまねることが容易です。
実際は、ID管理者は認証のみを行うので、認可はサービス管理者が担うことになります。
認証の有効期間などを受け取る必要があるケースもあるのでキーを決める必要があったりするのでJSONが採用されていたとしてもの最終系は同じくら複雑だったと思われます。

``` json
<!-- 単純なJSON（現代的なアプローチ）-->
{
  "user": "alice",
  "authenticated": true
}
```

``` xml
<!-- SAMLアサーション（エンタープライズ要件） -->
<saml:Assertion xmlns:saml="urn:oasis:names:tc:SAML:2.0:assertion"
                ID="_abc123"
                IssueInstant="2024-12-13T10:00:00Z"
                Version="2.0">
  <saml:Issuer>https://idp.example.com</saml:Issuer>
  
  <!-- デジタル署名で改ざん防止 -->
  <ds:Signature xmlns:ds="http://www.w3.org/2000/09/xmldsig#">
    <ds:SignedInfo>
      <ds:CanonicalizationMethod Algorithm="..."/>
      <ds:SignatureMethod Algorithm="..."/>
      <ds:Reference URI="#_abc123">
        <ds:DigestMethod Algorithm="..."/>
        <ds:DigestValue>...</ds:DigestValue>
      </ds:Reference>
    </ds:SignedInfo>
    <ds:SignatureValue>...</ds:SignatureValue>
    <ds:KeyInfo>
      <ds:X509Data>
        <ds:X509Certificate>...</ds:X509Certificate>
      </ds:X509Data>
    </ds:KeyInfo>
  </ds:Signature>
  
  <!-- 認証の詳細情報 -->
  <saml:Subject>
    <saml:NameID Format="urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress">
      alice@example.com
    </saml:NameID>
    <saml:SubjectConfirmation Method="urn:oasis:names:tc:SAML:2.0:cm:bearer">
      <saml:SubjectConfirmationData 
        NotOnOrAfter="2024-12-13T10:05:00Z"
        Recipient="https://sp.example.com/acs"/>
    </saml:SubjectConfirmation>
  </saml:Subject>
  
  <!-- 認証方法と時刻 -->
  <saml:AuthnStatement AuthnInstant="2024-12-13T10:00:00Z"
                       SessionIndex="_session123">
    <saml:AuthnContext>
      <saml:AuthnContextClassRef>
        urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport
      </saml:AuthnContextClassRef>
    </saml:AuthnContext>
  </saml:AuthnStatement>
  
  <!-- ユーザー属性 -->
  <saml:AttributeStatement>
    <saml:Attribute Name="email">
      <saml:AttributeValue>alice@example.com</saml:AttributeValue>
    </saml:Attribute>
    <saml:Attribute Name="department">
      <saml:AttributeValue>Engineering</saml:AttributeValue>
    </saml:Attribute>
    <saml:Attribute Name="role">
      <saml:AttributeValue>Senior Developer</saml:AttributeValue>
      <saml:AttributeValue>Team Lead</saml:AttributeValue>
    </saml:Attribute>
  </saml:AttributeStatement>
  
  <!-- 有効期限 -->
  <saml:Conditions NotBefore="2024-12-13T09:55:00Z"
                   NotOnOrAfter="2024-12-13T10:05:00Z">
    <saml:AudienceRestriction>
      <saml:Audience>https://sp.example.com</saml:Audience>
    </saml:AudienceRestriction>
  </saml:Conditions>
</saml:Assertion>
```

#### セキュリティ要件

認証という重要な処理である以上、規格としてセキュリティ要件が厳格であるべきです。

```
✓ デジタル署名（改ざん防止）
- XML Signature標準
- X.509証明書ベース
  
✓ 暗号化（機密性保護）
- XML Encryption標準
- アサーション全体の暗号化

✓ リプレイ攻撃対策
- タイムスタンプ（NotBefore/NotOnOrAfter）
- ユニークなID（AssertionID）
  
✓ 対象者制限
- Audience Restriction
- 特定のSPのみ有効

✓ 監査証跡
- 認証方法の記録
- セッションインデックス
```

#### 相互運用性

サービスの種類が増えてきたことで異なるベンダー間の連携に課題でした。

解決：
詳細な仕様により統一された動作

- 厳密なXMLスキーマ定義
- プロファイルによる実装パターンの標準化
- コンフォーマンステストによる検証
  
## まとめ

SAML認証の登場は当時は画期的だったと思います。
異なるベンダーでの連携とわかりやすい話と思い採用していますが、
１つのベンダーでも部署が違うサービスと連携させるのも一苦労でした。
新規サービスを作ったけど既存サービスと連携するために3種類くらいの認証処理を作ったことがあるのでベンダーが異なったら悪夢だったと思います。（人づてで聞いたら社内に5種類はあるはずといわれたので聞いたときに突っ込みをいれたのもいい思い出）

SAMLはID管理者としてMicosoftやGoogle、最近よく使われるOktaでも使われています。アカウント連携することでサービス利用の拡大を図ることを考えたらSAML認証をサービス管理者は取り入れてユーザーにパスワード管理の負担をかけないということが重要になってきました。
今回は、SAML規格について書いていきますが、その前にSAMLの次に登場したOAuthについて触れます。
