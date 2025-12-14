---
title: "アサーション"
---

SAML認証において、XML文書をアサーション（Assertion）と言います。
前チャプターで出てきたAuthnRequestやSAMLResponseに関してもアサーションで定義された構造を取っています。

## 基本構造

``` xml
<saml:Assertion xmlns:saml="urn:oasis:names:tc:SAML:2.0:assertion"
                ID="_abc123def456"
                Version="2.0"
                IssueInstant="2024-12-15T10:00:00Z">
  
  <!-- 発行者 -->
  <saml:Issuer>https://idp.example.com</saml:Issuer>
  
  <!-- デジタル署名（セキュリティの要） -->
  <ds:Signature xmlns:ds="http://www.w3.org/2000/09/xmldsig#">
    ...
  </ds:Signature>
  
  <!-- 主体（誰について） -->
  <saml:Subject>
    ...
  </saml:Subject>
  
  <!-- 条件（いつ、どこで有効） -->
  <saml:Conditions>
    ...
  </saml:Conditions>
  
  <!-- ステートメント（何を主張するか） -->
  <saml:AuthnStatement>...</saml:AuthnStatement>
  <saml:AttributeStatement>...</saml:AttributeStatement>
  <saml:AuthzDecisionStatement>...</saml:AuthzDecisionStatement>
  
</saml:Assertion>
```

### 必須要素の詳細

#### アサーションのルート要素

``` xml
<saml:Assertion 
    xmlns:saml="urn:oasis:names:tc:SAML:2.0:assertion"
    ID="_abc123def456"           <!-- 一意識別子（必須） -->
    Version="2.0"                <!-- SAMLバージョン（必須） -->
    IssueInstant="2024-12-15T10:00:00Z"> <!-- 発行時刻（必須） -->
```

ID属性の要件としては

- 先頭は文字またはアンダースコア
- 一意である必要がある（UUIDやランダム文字列を使用）
- リプレイ攻撃対策のため記録される。
  
Versionは基本SAML2.0を使うと思うので2.0となる。

IssueInstant:

- ISO 8601形式のUTC時刻
- タイムスタンプ検証に使用

#### Issuer（発行者）

``` xml
<saml:Issuer>https://idp.example.com</saml:Issuer>
```

- アサーションを発行したIdPの識別子(エンティティID)
- 通常はURLまたはURN形式
- SPはこの値でIdPを特定し、適切な検証鍵を選択

#### Subject(主体)

誰についての情報かを示します。

``` xml
<saml:Subject>
  <!-- NameID: ユーザーの識別子 -->
  <saml:NameID 
      Format="urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress"
      SPNameQualifier="https://sp.example.com">
    user@example.com
  </saml:NameID>
  
  <!-- SubjectConfirmation: 主体の確認方法 -->
  <saml:SubjectConfirmation 
      Method="urn:oasis:names:tc:SAML:2.0:cm:bearer">
    <saml:SubjectConfirmationData 
        NotOnOrAfter="2024-12-15T10:05:00Z"
        Recipient="https://sp.example.com/acs"
        InResponseTo="_request123"/>
  </saml:SubjectConfirmation>
</saml:Subject>
```

**NameID Format（よく使われる形式）：**
NameIDのFormatはURN形式でかかれ「urn:oasis:names:tc:SAML:1.1:nameid-format:」は共通で実際のFormat指定において重要なのは最後の部分になります。

| Format | 説明 | 例 |
|--------|------|-----|
| `emailAddress` | メールアドレス | user@example.com |
| `persistent` | 永続的な識別子 | _abc123def456 |
| `transient` | 一時的な識別子 | _xyz789 |
| `unspecified` | 未指定 | employee001 |
| `entity` | エンティティ識別子 | https://sp.example.com |

**SubjectConfirmation Method（確認方法）：**

```
bearer（ベアラー）:
- 最も一般的
- アサーションを持っている者が主体
- ブラウザベースのSSOで使用

holder-of-key:
- 公開鍵を保持している者が主体
- より強力な認証
- あまり使用されない

sender-vouches:
- 送信者が主体を保証
- バックチャネル通信で使用
```

#### Conditions（条件）

アサーションの有効性を制限します。

``` xml
<saml:Conditions 
    NotBefore="2024-12-15T09:55:00Z"
    NotOnOrAfter="2024-12-15T10:05:00Z">
  
  <!-- 対象者制限 -->
  <saml:AudienceRestriction>
    <saml:Audience>https://sp.example.com</saml:Audience>
  </saml:AudienceRestriction>
  
  <!-- その他の条件 -->
  <saml:OneTimeUse/>  <!-- 一度だけ使用可能 -->
  <saml:ProxyRestriction Count="0"/>  <!-- プロキシ禁止 -->
</saml:Conditions>
```

**時刻制限：**

```
NotBefore:  このアサーションが有効になる時刻
NotOnOrAfter: このアサーションが無効になる時刻

有効期間: 通常5-10分程度
理由: リプレイ攻撃のリスク軽減
```

**Audience（対象者）の重要性：**
アサーションの利用の限定機能です。
だれでも使える状態だと悪用されてしまうので必要な設定になります。

```
攻撃シナリオ（Audience制限がない場合）:

1. 攻撃者がSP-Aで正規にログイン
2. IdPから受け取ったアサーションをコピー
3. そのアサーションをSP-Bに送信
4. SP-Bが検証せずに受け入れてしまう
   → 攻撃者がSP-Bにもログイン成功

対策:
Audienceを必ず検証
- SP-AのアサーションにはAudience="SP-A"
- SP-Bは自分のエンティティIDと一致するか確認
```

### 3種類のステートメント

#### Authentication Statement(認証ステートメント)

「いつ、どのような方法で認証されたか」を示します

``` xml
<saml:AuthnStatement 
    AuthnInstant="2024-12-15T10:00:00Z"
    SessionIndex="_session123"
    SessionNotOnOrAfter="2024-12-15T18:00:00Z">
  
  <!-- 認証コンテキスト -->
  <saml:AuthnContext>
    <saml:AuthnContextClassRef>
      urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport
    </saml:AuthnContextClassRef>
  </saml:AuthnContext>
  
</saml:AuthnStatement>
```

**SessionIndex:**

- セッションの一意識別子
- Single Logout（SLO）で使用
- IdPが管理する全SPセッションの追跡に利用

**AuthnContextClassRef（認証方法）の例：**

```
Password:
urn:oasis:names:tc:SAML:2.0:ac:classes:Password
└─ パスワード認証

PasswordProtectedTransport:
urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport
└─ TLS保護されたパスワード認証

X509:
urn:oasis:names:tc:SAML:2.0:ac:classes:X509
└─ クライアント証明書認証

MultiFactor:
urn:oasis:names:tc:SAML:2.0:ac:classes:MultiFactor
└─ 多要素認証

Kerberos:
urn:oasis:names:tc:SAML:2.0:ac:classes:Kerberos
└─ Kerberos認証
```

#### Attribute Statement（属性ステートメント）

「ユーザーの属性情報」を提供します。

``` xml
<saml:AttributeStatement>
  
  <!-- メールアドレス -->
  <saml:Attribute 
      Name="email"
      NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:basic">
    <saml:AttributeValue xmlns:xs="http://www.w3.org/2001/XMLSchema"
                         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                         xsi:type="xs:string">
      alice@example.com
    </saml:AttributeValue>
  </saml:Attribute>
  
  <!-- 名前 -->
  <saml:Attribute Name="displayName">
    <saml:AttributeValue xsi:type="xs:string">
      Alice Smith
    </saml:AttributeValue>
  </saml:Attribute>
  
  <!-- 部署 -->
  <saml:Attribute Name="department">
    <saml:AttributeValue xsi:type="xs:string">
      Engineering
    </saml:AttributeValue>
  </saml:Attribute>
  
  <!-- 役割（複数の値） -->
  <saml:Attribute Name="role">
    <saml:AttributeValue xsi:type="xs:string">
      Developer
    </saml:AttributeValue>
    <saml:AttributeValue xsi:type="xs:string">
      TeamLead
    </saml:AttributeValue>
  </saml:Attribute>
  
  <!-- グループ -->
  <saml:Attribute Name="groups">
    <saml:AttributeValue xsi:type="xs:string">
      Engineering
    </saml:AttributeValue>
    <saml:AttributeValue xsi:type="xs:string">
      ProjectX
    </saml:AttributeValue>
  </saml:Attribute>
  
</saml:AttributeStatement>
```

**よく使われる属性名：**
属性の種類に関しては仕様としては定義されていません。
LDAP/Active Directory 属性がよく使われる傾向にはあります。
これはSAMLが策定されたタイミングでの企業側のユーザー管理がLDAP/Active Directoryであることが多かったために採用されたといわれています。
ここは提供するIdp側が決めてよいルールにもなるので利用する場合は確認するようにしましょう。

```
標準的な属性（LDAP/Active Directory由来）:

- uid: ユーザーID
- email/mail: メールアドレス
- displayName/cn: 表示名
- givenName: 名
- sn/surname: 姓
- department/ou: 部署
- title: 役職
- telephoneNumber: 電話番号
- employeeNumber: 社員番号
- memberOf/groups: 所属グループ
```

**Attribute Name Format:**

```
basic:
urn:oasis:names:tc:SAML:2.0:attrname-format:basic
└─ シンプルな名前

uri:
urn:oasis:names:tc:SAML:2.0:attrname-format:uri
└─ URI形式の名前

unspecified:
urn:oasis:names:tc:SAML:2.0:attrname-format:unspecified
└─ 未指定（最も柔軟）
```

#### Authorization Decision Statement（認可決定ステートメント）

「特定のリソースへのアクセス許可」を示します。

``` xml
<saml:AuthzDecisionStatement 
    Decision="Permit"
    Resource="https://sp.example.com/documents/confidential.pdf">
  
  <!-- アクション -->
  <saml:Action 
      Namespace="urn:oasis:names:tc:SAML:1.0:action:rwedc-negation">
    Read
  </saml:Action>
  
  <!-- 証拠（オプション） -->
  <saml:Evidence>
    <saml:Assertion>
      <!-- 別のアサーション -->
    </saml:Assertion>
  </saml:Evidence>
  
</saml:AuthzDecisionStatement>
```

**Decision値：**

```
Permit: 許可
Deny: 拒否
Indeterminate: 判定不能
```