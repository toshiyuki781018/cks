# 問題08：外部公開するWebアプリケーションの通信を暗号化する ― 解説・模範解答

## この問題では何を考えればよかったのか

この問題では、外部からWebアプリケーションへアクセスする通信をTLSで保護します。

重要なのは、

```text
Ingressを作る
```

こと自体ではありません。

まず、

```text
外部からWebアプリケーションへアクセスする
        ↓
平文通信のままでは通信内容を保護できない
        ↓
HTTPSを使用する
        ↓
TLSをどこで処理するか
        ↓
Ingress
        ↓
証明書と秘密鍵が必要
        ↓
既存TLS Secretを利用
```

という順番で考えます。

さらに今回の要件では、HTTPSを利用できるようにするだけでなく、

```text
HTTP
 ↓
HTTPSへRedirect
```

する必要があります。

つまり、

**HTTPSを提供することと、平文HTTPを残さないこと**

の両方がポイントです。

---

# 問題文を整理する

今回すでに存在しているものは次の通りです。

```text
Namespace
  prod02

Service
  web
  Port 80

TLS Secret
  web-cert

Host
  web.k8sng.local
```

つまり、新しく作る必要があるのは、

```text
Client
 ↓
Web公開用の入口
 ↓
Service
```

の部分です。

既存ServiceやTLS Secretを変更する必要はありません。

---

# 設計の順番

## 1. Webアプリケーションへの入口を作る

今回のWebアプリケーションは、

```text
web.k8sng.local
```

というHost名で公開します。

また、`/`以下のすべてのパスを既存の`web` Serviceへ転送する必要があります。

そのため、

```text
Host
  web.k8sng.local

Path
  /

Backend
  web:80
```

というルーティングを作ります。

Kubernetesでは、このようなHTTP/HTTPSルーティングにIngressを使用できます。

---

## 2. TLSを有効にする

次に、

```text
Client
 ↓
Ingress
```

の通信をTLSで保護します。

今回、証明書と秘密鍵はすでに、

```text
web-cert
```

というTLS Secretとして存在しています。

そのため、IngressからこのSecretを参照します。

```text
Ingress
 ↓
web-cert
 ↓
証明書 + 秘密鍵
 ↓
TLS
```

という構成になります。

---

## 3. HTTPアクセスをHTTPSへ誘導する

HTTPSが利用可能になっても、

```text
http://web.k8sng.local
```

へのアクセスをそのまま処理できる構成では、平文通信が残ります。

今回の要件では、

```text
HTTP
 ↓
HTTPSへRedirect
```

する必要があります。

利用しているのはNGINX Ingress Controllerなので、Controllerが提供する設定を使用してリダイレクトを有効にします。

---

# 模範解答

Ingressマニフェストを作成します。

```bash
vim ingress-web.yaml
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  namespace: prod02
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - web.k8sng.local
    secretName: web-cert

  rules:
  - host: web.k8sng.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web
            port:
              number: 80
```

作成したIngressを適用します。

```bash
kubectl apply -f ingress-web.yaml
```

---

# Ingressを読む

## metadata

```yaml
metadata:
  name: web
  namespace: prod02
```

`prod02` Namespaceに`web`というIngressを作成します。

---

## HTTP→HTTPS Redirect

```yaml
annotations:
  nginx.ingress.kubernetes.io/ssl-redirect: "true"
```

NGINX Ingress Controllerに対して、HTTPアクセスをHTTPSへリダイレクトするよう指定します。

ここで重要なのは、このAnnotationが、

**Ingress APIそのものの一般的なTLS設定ではなく、NGINX Ingress Controller側の設定**

だということです。

---

# TLS設定

```yaml
tls:
- hosts:
  - web.k8sng.local
  secretName: web-cert
```

この設定によって、

```text
Host
  web.k8sng.local

TLS Secret
  web-cert
```

を関連付けます。

`web-cert`にはTLS通信に必要な証明書と秘密鍵が格納されています。

---

# Routing設定

```yaml
rules:
- host: web.k8sng.local
```

このIngressが処理するHostを指定します。

---

```yaml
paths:
- path: /
  pathType: Prefix
```

`/`をPrefixとして指定するため、

```text
/
 /login
 /api
 /images
```

など、`/`以下のパスが対象になります。

---

# Backend Service

```yaml
backend:
  service:
    name: web
    port:
      number: 80
```

Ingressが受け取ったRequestを、既存の`web` ServiceのPort `80`へ転送します。

---

# 要件と設定の対応

| 要件              | Ingress設定              |
| --------------- | ---------------------- |
| `prod02`に作成     | `metadata.namespace`   |
| Ingress名`web`   | `metadata.name`        |
| Host            | `rules.host`           |
| `/`以下をルーティング    | `path: /` + `Prefix`   |
| Backend Service | `web:80`               |
| TLSを使用          | `spec.tls`             |
| TLS Secret      | `secretName: web-cert` |
| HTTP→HTTPS      | NGINX Annotation       |

設定値を暗記するのではなく、

```text
Web公開要件
 ↓
Host
 ↓
Path
 ↓
Backend
 ↓
TLS
 ↓
Redirect
```

という順番で整理すると考えやすくなります。

---

# 作成後の確認

Ingressを確認します。

```bash
kubectl get ingress -n prod02
```

例えば次のように表示されます。

```text
NAME   CLASS   HOSTS             ADDRESS         PORTS
web    nginx   web.k8sng.local   10.110.73.189   80,443
```

確認したいのは、

```text
Host
  web.k8sng.local

Ports
  80,443
```

です。

---

# Ingressの詳細を確認する

```bash
kubectl describe ingress web -n prod02
```

次の内容を確認します。

```text
Host
TLS
Backend Service
Backend Port
```

特に、

```text
web.k8sng.local
 ↓
web:80
```

となっていることを確認します。

---

# HTTPS通信を確認する

最終的な疎通確認を行います。

```bash
curl -Lk https://web.k8sng.local
```

元問題では、正常時に次のようなレスポンスが返ります。

```text
Hello World ^_^
```

これによって、

```text
Client
 ↓ HTTPS
Ingress
 ↓
web Service
 ↓
Application
```

という経路が成立していることを確認できます。

---

# HTTP→HTTPS Redirectを確認する

必要に応じて、HTTP側も確認できます。

```bash
curl -I http://web.k8sng.local
```

HTTPからHTTPSへのRedirectが行われることを確認します。

考え方としては、

```text
HTTP Request
    ↓
Ingress
    ↓
Redirect
    ↓
HTTPS Request
```

となります。

---

# TLS Secretは何をしているのか

今回使用している`web-cert`は、

```text
Type
  kubernetes.io/tls
```

のSecretです。

TLS Secretには通常、

```text
tls.crt
tls.key
```

というキーで証明書と秘密鍵が格納されます。

Ingressは、このSecretを利用してTLS通信を処理します。

```text
Certificate
+
Private Key
    ↓
TLS Secret
    ↓
Ingress
    ↓
HTTPS
```

つまり、Secret自体が通信を暗号化しているわけではありません。

**TLSを処理するIngressへ、必要な証明書と秘密鍵を提供している**

と考えてください。

---

# TLS終端として考える

今回の構成では、

```text
Client
   │
   │ HTTPS
   ▼
Ingress
   │
   │ HTTP
   ▼
Service
   ↓
Pod
```

という形になります。

IngressでHTTPS通信を受信し、TLSを復号してBackendへ転送します。

このように、

```text
TLS通信
 ↓
復号
```

を行う場所を、

**TLS Termination（TLS終端）**

と呼びます。

今回のTLS終端点はIngressです。

---

# HTTPSにしたら全部の通信が暗号化されるのか

ここは重要です。

今回設定しているTLSが保護するのは、

```text
Client
 ↓
Ingress
```

の通信です。

一方、

```text
Ingress
 ↓
Service
 ↓
Pod
```

の通信まで、今回の設定だけで自動的にTLS化されるわけではありません。

したがって、

```text
IngressにTLSを設定した
        ↓
クラスター内部もすべて暗号化された
```

とは考えないようにしてください。

---

# 必要であれば内部通信にも別の対策が必要

もし要件として、

```text
Client
 ↓
Ingress
 ↓
Service
 ↓
Pod

すべての区間を暗号化したい
```

のであれば、Ingress TLSだけでは足りません。

その場合は、

```text
Backend側でもTLSを使用する

Service MeshなどでmTLSを使用する

アプリケーション自身でTLSを使用する
```

など、別の設計が必要になります。

今回の問題では、

**外部ClientからWeb入口までをTLSで保護する**

ことが対象です。

---

# IngressとIngress Controllerの違い

今回の問題では、HTTP→HTTPS Redirectのために、

```yaml
nginx.ingress.kubernetes.io/ssl-redirect: "true"
```

を使用しています。

これは、

```text
Ingress
```

というKubernetes APIだけで動作するものではありません。

実際にIngressのルールを処理するのは、

```text
Ingress Controller
```

です。

今回なら、

```text
Ingress Resource
        ↓
NGINX Ingress Controller
        ↓
実際のHTTP/HTTPS Routing
```

という関係です。

したがって、Annotationなどの設定は使用するIngress Controllerによって異なる場合があります。

---

# ケーススタディ

## なぜHTTPを残さないのか

HTTPSを有効にしても、

```text
HTTPS
○

HTTP
○
```

という状態では、利用者がHTTPでアクセスする可能性があります。

そのため、

```text
HTTP
 ↓
HTTPS
```

へ強制的に誘導します。

これによって、

```text
利用者がURLをHTTPで入力
        ↓
HTTPSへRedirect
        ↓
TLS通信
```

という状態を作れます。

---

# 第2問との接続

第2問では、

```text
Certificate
+
Private Key
    ↓
TLS Secret
```

を扱いました。

今回の第8問では、

```text
TLS Secret
    ↓
Ingress
    ↓
実際のHTTPS通信
```

を扱っています。

つまり、

```text
第2問
 ↓
セキュリティ資材をKubernetesへ登録する

第8問
 ↓
登録された資材を通信制御へ利用する
```

という関係です。

---

# 第7問との違い

第7問のNetworkPolicyでは、

```text
誰から誰へ通信できるのか
```

を制御しました。

今回のTLSでは、

```text
その通信内容をどう保護するのか
```

を制御しています。

整理すると、

```text
NetworkPolicy
    ↓
通信可能範囲

TLS
    ↓
通信内容の保護
```

です。

両方を組み合わせることで、

```text
必要な相手とだけ通信できる
        +
通信内容も保護される
```

というネットワークセキュリティを作れます。

---

# この問題で学んでほしいこと

この問題で覚えてほしいのは、

```yaml
tls:
- hosts:
  - web.k8sng.local
  secretName: web-cert
```

というYAMLだけではありません。

重要なのは、

```text
Webアプリケーションを公開する
        ↓
どの通信を保護するのか
        ↓
TLSをどこで処理するのか
        ↓
どの証明書を使うのか
        ↓
平文通信をどう扱うのか
        ↓
Ingress
```

という考え方です。

さらに、

```text
HTTPSで公開した
```

という事実だけではなく、

```text
どの区間がTLSで守られているのか
```

まで理解する必要があります。

CKSでは、**「IngressにTLS設定を書ける」ことから一段進んで、  
「どの通信区間を保護し、どこでTLSを終端し、平文通信をどう排除するかを判断できる」こと**  
をこの問題の到達点とします。
