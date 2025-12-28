---
title: "プロファイル"
---

## プロファイルとは
特定のユースケースで、どのバインディングとプロトコルをどう組み合わせて使うかを定義した「実装パターン」になります。

```
例：
「Webブラウザベースのシングルサインオン」
→ どのバインディングを使う？
→ どの順番で？
→ これを定義するのがProfile
```

## Web Browser SSO Profile

**最も一般的なプロファイル**になります。

### SP-initiated SSO（SPから開始）

SAML認証として一番浸透しているプロファイルだと思います。

```
使用するバインディング：
- AuthnRequest: HTTP Redirect Binding
- Response: HTTP POST Binding

フロー：
User → SP → IdP (HTTP Redirect)
         ← IdP (HTTP POST)
       ← SP
```

#### シーケンス

```
User         SP                IdP
 |            |                 |
 | ①アクセス  |                 |
 |---------->|                 |
 |           |                 |
 |           | ②未認証検出      |
 |           |                 |
 | ③302 Redirect               |
 | Location: https://idp.../sso?
 | SAMLRequest=...             |
 |<----------|                 |
 |                             |
 | ④GET /sso?SAMLRequest=...   |
 |---------------------------->|
 |                             |
 |                             | ⑤認証処理
 |                             |
 | ⑥200 OK (HTMLフォーム)       |
 |<----------------------------|
 |                             |
 | ⑦POST /acs                  |
 | SAMLResponse=...            |
 |---------->|                 |
 |           |                 |
 |           | ⑧検証・セッション作成
 |           |                 |
 | ⑨302 Redirect               |
 | Location: /dashboard        |
 |<----------|                 |
 |           |                 |
```

### IdP-initiated SSO（IdPから開始）

同様にIdpから始める方式ですが、こちらは認証後のSPの行先に困るパターンではあります。

```
使用するバインディング：
- Response: HTTP POST Binding のみ

フロー：
User → IdP (ログイン)
     → IdP → SP (HTTP POST)
```

#### シーケンス

```
User         IdP               SP
 |            |                 |
 | ①IdPログイン|                |
 |---------->|                 |
 |           |                 |
 |           | ②認証成功        |
 |           |                 |
 | ③ポータル画面                |
 |<----------|                 |
 |           |                 |
 | ④アプリ選択                  |
 | (Salesforce等)             |
 |---------->|                 |
 |           |                 |
 | ⑤200 OK (HTMLフォーム)       |
 | SAMLResponse=...            |
 |<----------|                 |
 |                             |
 | ⑥POST /acs                  |
 | SAMLResponse=...            |
 |---------------------------->|
 |                             |
 |                             | ⑦検証
 |                             |
 | ⑧ログイン完了               |
 |<----------------------------|
 |                             |
```

## Single Logout Profile

**すべてのSPから一斉にログアウト**

### Front-Channel Logout（ブラウザ経由）

```
使用するバインディング：
- LogoutRequest: HTTP Redirect Binding
- LogoutResponse: HTTP Redirect Binding

フロー：
User → SP1 (ログアウト)
     → IdP → SP2
           → SP3
           → ...
```

#### シーケンス

```
User     SP1         IdP         SP2         SP3
 |        |           |           |           |
 | ①ログアウト         |           |           |
 |------->|           |           |           |
 |        |           |           |           |
 | ②302 Redirect      |           |           |
 | LogoutRequest      |           |           |
 |<-------|           |           |           |
 |                    |           |           |
 | ③GET /slo?         |           |           |
 | LogoutRequest=...  |           |           |
 |------------------->|           |           |
 |                    |           |           |
 |                    | ④LogoutRequest        |
 |                    |---------->|           |
 |                    |           |           |
 |                    |           | ⑤ログアウト処理
 |                    |           |           |
 |                    | ⑥LogoutResponse       |
 |                    |<----------|           |
 |                    |           |           |
 |                    | ⑦LogoutRequest        |
 |                    |-------------------->|
 |                    |           |           |
 |                    |           |           | ⑧ログアウト処理
 |                    |           |           |
 |                    | ⑨LogoutResponse       |
 |                    |<--------------------|
 |                    |           |           |
 | ⑩302 Redirect      |           |           |
 | (ログアウト完了)    |           |           |
 |<-------------------|           |           |
 |                    |           |           |
 ```

### Back-Channel Logout（サーバー間通信）

```
使用するバインディング：
- LogoutRequest: SOAP Binding
- LogoutResponse: SOAP Binding

フロー：
User → SP1 (ログアウト)
     → IdP ⇄ SP2 (SOAP)
           ⇄ SP3 (SOAP)
```

**メリット・デメリット：**

| 方式 | メリット | デメリット |
|------|---------|-----------|
| **Front-Channel** | 実装簡単 | ブラウザ経由で遅い |
| **Back-Channel** | 高速、確実 | SOAP実装が必要 |

この辺りのログアウト問題は難しいところではあります。
ユーザーがさまざまな方法でサービスでアクセスができてしまうので利用サービスのどれかでログアウトされたときにすべて無効にする必要があるためです。
恐らくですが、Back-ChanelでSP側にSOAPの実装が求められることが増えてきているのでは？と思っています。
Front-Chanel方式は実装は楽なので、浸透しているのはこの方式なのだろうと思っています。

### Enhanced Client/Proxy (ECP) Profile

**リッチクライアント向け（ほとんど使われない）**になります。

```
用途：
- デスクトップアプリケーション
- モバイルネイティブアプリ
- APIクライアント

特徴：
- ブラウザを使わない
- SOAPベース
- ほとんど実装されていない
```

ブラウザを使用しないケースのために用意されているプロファイルですが、
アプリでもシステムブラウザなどを開く方式が採用されていることが多くて使われていない方式にはなっています。もしくは、OpenID Connectにこの層は取られがちのようです。

