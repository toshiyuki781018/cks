# 問題08：外部公開するWebアプリケーションの通信を暗号化する

## 対象領域

Minimize Microservice Vulnerabilities / Network Security

## 教材種別

Drill（試験対策問題）

目安時間：10〜15分

# 背景

`prod02` Namespaceでは、社内向けWebアプリケーションをKubernetes上で運用しています。  

現在、アプリケーションを外部から利用できるようにするためのServiceは作成されていますが、  
Webアクセスを安全に公開するための入口がまだ構成されていません。

セキュリティ要件として、クライアントとクラスターのWeb入口との間では、  
通信内容を平文のまま送信しないことが求められています。

また、利用者が誤って平文通信でアクセスした場合も、そのままアプリケーションを利用させず、  
安全な通信方式へ誘導する必要があります。

既存の証明書とServiceを利用して、Webアプリケーションを安全に公開してください。

# 初期観測情報

対象Namespaceには、Webアプリケーションへ接続するためのServiceがすでに存在します。

```text
Namespace : prod02
Service   : web
Port      : 80
```

また、Web公開に使用する証明書と秘密鍵は、Kubernetes Secretとしてすでに登録されています。

```text
Secret : web-cert
Type   : kubernetes.io/tls
```

Webアプリケーションは、次のホスト名で公開します。

```text
web.k8sng.local
```

Ingress ControllerとしてNGINX Ingress Controllerが利用できる環境です。

# 要件

以下の要件をすべて満たしてください。

1. `prod02` Namespaceに、Webアプリケーションを外部公開するための`web`というリソースを作成すること。
2. `web.k8sng.local`宛てのアクセスを既存の`web` Serviceへルーティングすること。
3. `/`以下のすべてのパスを既存Serviceへ転送できること。
4. クライアントからWeb入口までの通信をTLSで保護すること。
5. TLS通信には、既存の`web-cert` Secretに格納されている証明書と秘密鍵を利用すること。
6. 平文のHTTPでアクセスされた場合、そのままアプリケーションを利用させずHTTPSへリダイレクトすること。
7. 既存のServiceおよびSecretを変更せずに要件を実現すること。

# 設定情報

| 項目                 | 設定内容                     |
| ------------------ | ------------------------ |
| Namespace          | `prod02`                 |
| 作成するリソース名          | `web`                    |
| Host               | `web.k8sng.local`        |
| Path               | `/`                      |
| Pathの対象            | `/`以下のすべてのパス             |
| Backend Service    | `web`                    |
| Backend Port       | `80`                     |
| TLS Secret         | `web-cert`               |
| Secret Type        | `kubernetes.io/tls`      |
| Ingress Controller | NGINX Ingress Controller |
| HTTPアクセス           | HTTPSへリダイレクト             |

# 制約

* 既存の`web` Serviceを変更してはいけません。
* 既存の`web` Serviceを削除してはいけません。
* 既存の`web-cert` Secretを変更してはいけません。
* 既存の`web-cert` Secretを削除してはいけません。
* 新しいTLS証明書や秘密鍵を作成してはいけません。
* 新しいTLS Secretを作成してはいけません。
* TLS通信には既存の`web-cert`を使用してください。
* Webアプリケーション側のPodを変更してはいけません。
* セキュリティ要件を満たすために必要なWeb公開設定のみを作成してください。

# 完了条件

以下をすべて満たした場合、作業完了とします。

* `prod02` Namespaceに`web`というIngressリソースが存在する。
* `web.k8sng.local`へのアクセスが`web` Serviceへルーティングされる。
* `/`以下のパスがBackendへ転送される。
* HTTPSでWebアプリケーションへアクセスできる。
* TLS通信で`web-cert`が使用されている。
* HTTPでアクセスした場合にHTTPSへリダイレクトされる。
* 既存のServiceおよびSecretが変更されていない。

# 作業後の確認項目

設定後、次の内容を確認してください。

* `prod02` Namespaceに作成したIngressが存在すること。
* Hostが`web.k8sng.local`になっていること。
* Backendとして`web` ServiceのPort `80`が指定されていること。
* TLSに`web-cert`が使用されていること。
* HTTPS用の通信経路が有効になっていること。
* HTTPSでWebアプリケーションへアクセスできること。
* HTTPアクセスがHTTPSへリダイレクトされること。

最終的な疎通確認として、次のコマンドを使用してください。

```bash
curl -Lk https://web.k8sng.local
```

Webアプリケーションから正常なレスポンスが返ることを確認してください。

# 注意事項

今回の目的は、

**「Ingressを作成してWebアプリケーションへアクセスできるようにする」**

だけではありません。

外部からWebアプリケーションへアクセスする場合、

```text
Client
   ↓
HTTP
   ↓
Web Application
```

では、通信内容が平文のままネットワークを通過します。

そのため、

```text
外部公開する
    ↓
通信を保護する必要がある
    ↓
TLSを使用
    ↓
証明書と秘密鍵が必要
    ↓
既存TLS Secretを利用
    ↓
Web入口でTLS通信を処理
```

という順番で考えてください。

また、HTTPSでアクセスできるだけでは、HTTPによる平文アクセスが残っている可能性があります。

今回求められている状態は、

```text
HTTPS
   ↓
Web Application

HTTP
   ↓
HTTPSへRedirect
   ↓
Web Application
```

です。

そのため、**「HTTPSを利用できること」と「HTTPをHTTPSへ誘導すること」は別の要件**  
として考えてください。

さらに、今回利用する証明書と秘密鍵はすでにSecretとして存在しています。

```text
証明書・秘密鍵
      ↓
TLS Secret
      ↓
Web入口
      ↓
HTTPS
```

となるため、新しい証明書を作成するのではなく、  
**既存のセキュリティ資産を、必要な通信経路へ正しく関連付ける**  
ことが求められます。

今回の構成では、TLSによって保護される範囲についても意識してください。

```text
Client
   │
   │ HTTPS
   ▼
Ingress
   │
   │ Backendへの通信
   ▼
Service
   ↓
Pod
```

Web入口でTLS通信を処理する構成では、クライアントからWeb入口までの通信は保護されます。

一方、この設定だけで、**IngressからBackendまでを含むクラスター内部の  
すべての通信がTLS化されるわけではありません。**

この問題では、

```text
何から通信を守るのか
        ↓
どの通信区間を暗号化するのか
        ↓
どこでTLSを処理するのか
        ↓
どの証明書を使用するのか
        ↓
平文アクセスをどう扱うのか
```

という観点から、Webアプリケーションの公開方法を判断してください。
