# 問題01：Control Planeコンポーネントのセキュリティ設定を是正する ― 解説・模範解答

## この問題では何を考えればよかったのか

この問題では、kubeletとetcdに対して、アクセス制御を適切に設定する必要があります。  
重要なのは、設定項目を暗記することではありません。

まず、問題文に示されたセキュリティ要件を整理します。

```text
kubeletへの匿名アクセスを禁止したい
        ↓
kubeletへアクセスするクライアントを認証する必要がある
        ↓
匿名認証を無効化する
```

```text
kubeletへの要求を無条件に許可したくない
        ↓
アクセスを許可するか判断する仕組みが必要
        ↓
適切な認可方式を使用する
```

```text
etcdへ信頼できるクライアントだけを接続させたい
        ↓
クライアントの身元を確認する必要がある
        ↓
クライアント証明書を検証する
```

このように、  
**脅威・リスク → 守る対象 → セキュリティ要件 → 制御点 → 設定**  
の順番で考えることが、この問題のポイントです。

---

# 問題文を整理する

今回の要件は、大きく3つに分けられます。

| 対象      | 現在の問題           | 実現したい状態                |
| ------- | --------------- | ---------------------- |
| kubelet | 匿名アクセスを受け付ける    | 認証されたクライアントだけを扱う       |
| kubelet | 要求を無条件に許可できる    | 認可によってアクセス可否を判断する      |
| etcd    | クライアント証明書を要求しない | 信頼された証明書を持つクライアントを確認する |

ここで、  
**認証（Authentication）と認可（Authorization）は別の処理**  
であることに注意してください。

```text
Authentication
    ↓
「あなたは誰ですか？」

Authorization
    ↓
「あなたはこの操作をしてよいですか？」
```

認証できたからといって、すべての操作を許可してよいわけではありません。

---

# 設計の順番

## 1. kubeletへの匿名アクセスを禁止する

最初に考えるのは、

```text
認証情報を持たないクライアントを
kubeletが受け付けないようにするにはどうするか
```

kubeletには認証に関する設定があり、匿名アクセスを許可するかどうかを制御できます。

そのため、

```yaml
authentication:
  anonymous:
    enabled: false
```

と設定します。

これによって、匿名によるアクセスを許可しない構成にします。

---

## 2. kubeletの認可方式を変更する

次の問題は、

```text
認証されたクライアントから要求が来たとしても、
すべての操作を無条件に許可してよいのか
```

という点です。

`AlwaysAllow`では、要求を無条件に許可する認可方式となります。  
今回の要件では、この方式を使用してはいけません。

そこで、kubeletからKubernetes API Serverへ認可判断を問い合わせるWebhook方式を使用します。

```yaml
authorization:
  mode: Webhook
```

これによって、

```text
Request
   ↓
kubelet
   ↓
認可判断
   ↓
Webhook
   ↓
Kubernetes API Server
```

という形でアクセス可否を判断できるようにします。

---

## 3. etcdでクライアント証明書を要求する

etcdにはKubernetesクラスターの重要な状態情報が保存されています。  
そのため、etcdへ接続できるクライアントを適切に確認する必要があります。

今回の要件では、

```text
etcdへ接続
   ↓
クライアント証明書を提示
   ↓
信頼されたCAによって検証
   ↓
接続を許可
```

という状態を作ります。

そのため、etcdではクライアント証明書認証を有効にします。

```text
--client-cert-auth=true
```

ただし、ここで重要なのは、  
**この設定だけを有効にすればよいわけではない**  
という点です。

etcdがTLS通信およびクライアント証明書の検証を行うためには、  
関連する証明書・秘密鍵・CAが正しく指定されている必要があります。

---

# Kubernetesではどの設定になるのか

## kubelet

kubeletの設定ファイルを確認します。

```bash
/var/lib/kubelet/config.yaml
```

今回確認する主な設定は次の部分です。

```yaml
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true

authorization:
  mode: Webhook
```

これによって、

```text
匿名アクセス
    ↓
禁止

認証されたアクセス
    ↓
認可判断
    ↓
Webhook
```

という構成になります。

---

## etcd

kubeadmで構築されたControl Planeでは、etcdはStatic Podとして動作します。  
マニフェストは次の場所にあります。

```bash
/etc/kubernetes/manifests/etcd.yaml
```

今回確認する主な設定は、以下です。

```text
--client-cert-auth=true
```

さらに、証明書認証を成立させるために、  
環境によっては次の設定が適切に存在していることを確認する必要があります。

```text
--cert-file=/etc/kubernetes/pki/etcd/server.crt
--key-file=/etc/kubernetes/pki/etcd/server.key
--trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
```

設定を追加・変更する前に、指定する証明書ファイルが実際に存在していることも確認します。

---

# 模範解答

## 1. kubelet設定を変更する

`/var/lib/kubelet/config.yaml`を編集し、次の状態にします。

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1

authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt

authorization:
  mode: Webhook
```

今回の要件に直接対応しているのは、

```yaml
anonymous:
  enabled: false
```

と、

```yaml
authorization:
  mode: Webhook
```

です。

---

## 2. etcd設定を確認・変更する

`/etc/kubernetes/manifests/etcd.yaml`を確認します。

etcdコンテナの`command`に、クライアント証明書認証を有効にする設定を含めます。

```yaml
spec:
  containers:
  - command:
    - etcd
    - --cert-file=/etc/kubernetes/pki/etcd/server.crt
    - --key-file=/etc/kubernetes/pki/etcd/server.key
    - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    - --client-cert-auth=true
```

既に必要な証明書関連オプションが存在する場合は、重複して追加する必要はありません。  
既存設定を確認したうえで、今回のセキュリティ要件を満たしていない部分だけを変更します。

---

# 設定を読む

## kubelet

```yaml
authentication:
  anonymous:
    enabled: false
```

匿名クライアントとしてkubeletへアクセスすることを許可しません。

---

```yaml
authentication:
  webhook:
    enabled: true
```

Webhookを利用した認証を有効にします。

---

```yaml
authorization:
  mode: Webhook
```

kubeletが要求を無条件に許可するのではなく、Webhookを使用して認可判断を行います。

---

## etcd

```text
--client-cert-auth=true
```

クライアント証明書による認証を有効にします。

---

```text
--cert-file=/etc/kubernetes/pki/etcd/server.crt
```

etcdサーバーが使用する証明書を指定します。

---

```text
--key-file=/etc/kubernetes/pki/etcd/server.key
```

サーバー証明書に対応する秘密鍵を指定します。

---

```text
--trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
```

クライアント証明書を検証するときに信頼するCAを指定します。

---

# 要件と設定の対応

| 要件                   | 設定                                        |
| -------------------- | ----------------------------------------- |
| kubeletへの匿名アクセスを禁止   | `authentication.anonymous.enabled: false` |
| kubeletで無条件許可を使用しない  | `authorization.mode: Webhook`             |
| etcdでクライアント証明書を要求    | `--client-cert-auth=true`                 |
| etcdが証明書を使用してTLS通信   | `--cert-file` / `--key-file`              |
| クライアント証明書を信頼されたCAで検証 | `--trusted-ca-file`                       |

このように、設定値を個別に覚えるのではなく、

```text
セキュリティ要件
       ↓
どこを制御するか
       ↓
必要な設定
```

という対応関係で理解してください。

---

# 適用

kubeletの設定を変更した場合は、kubeletを再起動して設定を反映します。

```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

etcdはStatic Podとして管理されているため、`/etc/kubernetes/manifests/etcd.yaml`の変更をkubeletが検知すると、etcd Podが再作成されます。

---

# 作成後に確認すること

## kubelet

```bash
systemctl status kubelet
```

kubeletが正常に起動していることを確認します。

設定ファイルについても確認します。

```bash
grep -A 10 "^authentication:" /var/lib/kubelet/config.yaml
```

```bash
grep -A 5 "^authorization:" /var/lib/kubelet/config.yaml
```

---

## etcd

Static Podの状態を確認します。

```bash
kubectl get pods -n kube-system
```

etcd Podが正常に稼働していることを確認します。

マニフェストの設定も確認します。

```bash
grep -E "client-cert-auth|cert-file|key-file|trusted-ca-file" \
/etc/kubernetes/manifests/etcd.yaml
```

---

## クラスター全体

最後に、Kubernetes APIへ正常にアクセスできることを確認します。

```bash
kubectl get nodes
```

```bash
kubectl get pods -A
```

設定変更後にこれらの操作が正常に行えることを確認してください。

---

# 設定変更で問題が発生した場合

セキュリティ設定を強化した直後にControl Planeへアクセスできなくなった場合、

```text
設定を変更した
    ↓
動かなくなった
    ↓
設定を元に戻す
```

とすぐに判断するのではなく、

```text
どの通信が失敗しているのか
        ↓
認証なのか
        ↓
認可なのか
        ↓
証明書なのか
        ↓
CAによる検証なのか
```

を切り分けます。

特にetcdでクライアント証明書認証を有効にした場合は、

```text
client-cert-auth
       ↓
クライアント証明書が必要
       ↓
証明書を検証するCAが必要
       ↓
証明書・秘密鍵・CAの整合性
```

まで確認することが重要です。

---

# ケーススタディ

## なぜkubeletを保護する必要があるのか

kubeletは各Node上でPodやコンテナを管理する重要なコンポーネントです。

そのため、

```text
kubeletへアクセスできる
        ↓
Node上のWorkloadへ影響できる可能性
        ↓
クラスターへの影響拡大
```

というリスクを考える必要があります。

単に「kubeletの設定だから変更する」のではなく、

**Node上のWorkloadを管理する入口を保護している**

と考えると、この設定の意味が理解しやすくなります。

---

## なぜetcdを保護する必要があるのか

etcdにはKubernetesクラスターの状態を構成する重要なデータが保存されています。  
そのため、etcdへのアクセス制御はControl Planeを保護するうえで重要です。

```text
etcd
 ↓
クラスター状態を保持
 ↓
不正アクセス
 ↓
クラスター全体への影響
```

だからこそ、

```text
誰でも接続できる状態
```

ではなく、

```text
信頼できる証明書を持ったクライアント
        ↓
CAによる検証
        ↓
接続を許可
```

という境界を作ります。

---

# この問題で学んでほしいこと

この問題で重要なのは、

```text
anonymous = false

Webhook

client-cert-auth = true
```

という設定値を暗記することではありません。

覚えてほしいのは、

```text
脅威・リスク
    ↓
何を守るのか
    ↓
誰から守るのか
    ↓
どこで制御するのか
    ↓
認証・認可
    ↓
必要な設定
    ↓
変更後の検証
```

という考え方です。

CKSでは、

**「セキュリティ設定を知っている」ことと、「なぜその設定が必要なのか判断できる」ことは別です。**  
CIS Benchmarkなどのチェック結果を見たときも、単にFAILとなった項目を修正するのではなく、  
**その設定がどの境界を守り、どのようなアクセスを防いでいるのか**

ここまで考えられるようになることを、この問題の到達点とします。

