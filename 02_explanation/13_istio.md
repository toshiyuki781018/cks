# 問題13：Workload間通信に相互認証を強制する ― 解説・模範解答

## この問題では何を考えればよかったのか

この問題では、`mtls` Namespace内のWorkload間通信をIstioのService Meshによって保護し、  
相互認証された通信だけを許可します。

重要なのは、

```text
PeerAuthenticationを作成する
```

ことだけではありません。

まず、

```text
Workload
    ↓
Istioの通信制御下にあるか
    ↓
istio-proxyを確認
```

する必要があります。

そのうえで、

```text
Workload A
    ↓
Istio Proxy
    ↓
mTLS
    ↓
Istio Proxy
    ↓
Workload B
```

という通信経路を作り、相互認証されていない通信を許可しない状態にします。

つまり、この問題では、

```text
Service Meshへの参加
        ↓
Sidecar Proxy
        ↓
Workload間通信
        ↓
mTLS
        ↓
Namespace全体で強制
```

という順番で考えることが重要です。

---

# 問題文を整理する

今回必要になる設定は、大きく2つです。

| セキュリティ要件                | 実現すること                     |
| ----------------------- | -------------------------- |
| WorkloadをIstioの通信制御下へ置く | Sidecar Injection          |
| 相互認証された通信だけを許可する        | PeerAuthenticationでmTLSを強制 |

さらに、対象は特定のDeploymentではありません。

```text
mtls Namespace
        ↓
すべてのWorkload
```

が対象です。

したがって、Namespace単位で設定することを考えます。

---

# 設計の順番

## 1. 現在のPodを確認する

まず、`mtls` Namespaceで稼働しているPodを確認します。

```bash
kubectl get pods -n mtls
```

次に、各PodのContainer構成を確認します。

```bash
kubectl get pods -n mtls \
  -o custom-columns='NAME:.metadata.name,CONTAINERS:.spec.containers[*].name'
```

期待する状態は、アプリケーションコンテナに加えて、

```text
istio-proxy
```

が存在することです。

例えば、

```text
NAME         CONTAINERS
app-a-xxxxx  app-a,istio-proxy
app-b-xxxxx  app-b,istio-proxy
```

のような状態です。

---

# なぜistio-proxyを確認するのか

Istioでは、Sidecar Proxyを使用することでWorkload間通信をService Meshの制御下へ置く構成があります。

概念的には、

```text
Application A
      ↓
istio-proxy
      ↓
   Network
      ↓
istio-proxy
      ↓
Application B
```

となります。

アプリケーション自身が、

```text
証明書を管理する
TLSを実装する
相手を認証する
```

のではなく、Proxyが通信制御を担当します。

そのため、この問題ではまず、

**対象WorkloadがIstioの通信経路へ参加していること**

を確認する必要があります。

---

# NamespaceのSidecar Injection設定を確認する

`mtls` NamespaceのLabelを確認します。

```bash
kubectl get namespace mtls --show-labels
```

Istioの自動Sidecar Injectionを利用する構成では、元資料の説明上、Namespaceに次のLabelを設定します。

```text
istio-injection=enabled
```

必要であれば設定します。

```bash
kubectl label namespace mtls \
  istio-injection=enabled \
  --overwrite
```

---

# Labelを付けただけでは既存Podは変わらない

ここは重要です。

NamespaceにSidecar Injectionを設定しても、

```text
既存Pod
    ↓
自動的にistio-proxyが追加される
```

わけではありません。

Sidecar Injectionは、新しくPodが作成される際に行われます。

したがって、

```text
NamespaceへLabel設定
        ↓
既存Podはそのまま
        ↓
Podを再作成
        ↓
新しいPod
        ↓
istio-proxyがInjection
```

という流れになります。

---

# Workloadを再作成する

元問題では、`mtls` Namespace内のDeploymentを再起動してPodを再作成しています。

対象Deploymentを確認します。

```bash
kubectl get deployment -n mtls
```

Deployment管理下のPodを再作成する場合は、例えば次のように実行できます。

```bash
kubectl rollout restart deployment -n mtls
```

再作成後、

```bash
kubectl get pods -n mtls
```

でPodが正常に起動していることを確認します。

---

# Sidecar Injectionを確認する

再度Container構成を確認します。

```bash
kubectl get pods -n mtls \
  -o custom-columns='NAME:.metadata.name,CONTAINERS:.spec.containers[*].name'
```

すべての対象Podについて、

```text
Application Container
+
istio-proxy
```

となっていることを確認します。

ここまでで、

```text
Workload
    ↓
Istio Service Meshへ参加
```

するための準備ができました。

---

# mTLSを考える

次に、Workload間通信そのものを保護します。

通常のTLSでは、概念的には、

```text
Client
    ↓
Serverの証明書を確認
    ↓
暗号化通信
```

となります。

一方、mTLSでは、

```text
Workload A
     ↓
自分のIdentityを提示
     ↕
相手のIdentityを確認
     ↕
Workload B
```

となります。

つまり、

**通信する双方が互いを認証する**

ことが特徴です。

---

# IstioでmTLSを制御する

元問題では、Istioの`PeerAuthentication`を使用しています。

今回の要件は、

```text
mtls Namespace
        ↓
すべてのWorkload
        ↓
mTLSを必須
```

です。

そのため、Namespace全体へ適用する`PeerAuthentication`を作成します。

---

# 模範解答

例えば、次のマニフェストを作成します。

```bash
vim mtls-policy.yaml
```

内容は次のようにします。

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: mtls
spec:
  mtls:
    mode: STRICT
```

適用します。

```bash
kubectl apply -f mtls-policy.yaml
```

---

# YAMLを読む

## apiVersion

```yaml
apiVersion: security.istio.io/v1beta1
```

IstioのSecurity APIを使用します。

---

## kind

```yaml
kind: PeerAuthentication
```

Workloadが受信する通信について、どのような認証方式を要求するかを制御します。

今回要求しているのは、

```text
Workload間通信
        ↓
相互認証を必須にする
```

ことです。

---

## namespace

```yaml
metadata:
  namespace: mtls
```

今回の対象は`mtls` Namespaceです。

---

## selectorがない理由

今回のマニフェストには、特定Workloadを選択する`selector`を設定していません。

```yaml
spec:
  mtls:
    mode: STRICT
```

としています。

これは今回、

```text
特定のDeploymentだけ
```

ではなく、

```text
mtls Namespace全体
```

へ適用したいためです。

---

# STRICTを考える

今回最も重要な設定が、

```yaml
mode: STRICT
```

です。

今回要求されている状態は、

```text
mTLS通信
    ↓
許可

非mTLS通信
    ↓
拒否
```

です。

つまり、

**mTLSを利用できる状態ではなく、mTLSを必須とする状態**

を作ります。

---

# 「利用できる」と「強制する」は違う

例えば、

```text
mTLSを使える
```

だけでは、

```text
mTLS
 ↓
通信可能

非mTLS
 ↓
通信可能
```

という構成が残る可能性があります。

今回求めているのは、

```text
mTLS
 ↓
通信可能

非mTLS
 ↓
通信不可
```

です。

したがって、

```text
暗号化通信に対応する
```

だけではなく、

```text
相互認証されていない通信を拒否する
```

ところまで設定する必要があります。

---

# 要件と設定の対応

| 要件                          | Istio側の設定                       |
| --------------------------- | ------------------------------- |
| WorkloadをService Meshへ参加させる | Sidecar Injection               |
| Proxy Containerを動作させる       | `istio-proxy`                   |
| Namespace単位でInjection       | Namespace Label                 |
| 相互認証を利用する                   | mTLS                            |
| 非mTLS通信を許可しない               | `STRICT`                        |
| Namespace全体へ適用              | selectorなしの`PeerAuthentication` |

この対応を、

```text
Workloadを通信制御下へ置く
        ↓
Sidecar Injection

通信相手を相互認証する
        ↓
mTLS

相互認証なしを許可しない
        ↓
STRICT
```

として理解してください。

---

# 作成後に確認すること

まず`PeerAuthentication`を確認します。

```bash
kubectl get peerauthentication -n mtls
```

詳細も確認できます。

```bash
kubectl get peerauthentication \
  -n mtls \
  -o yaml
```

`STRICT`になっていることを確認します。

---

# Podの状態を確認する

```bash
kubectl get pods -n mtls
```

すべての対象Podが正常に`Running`になっていることを確認します。

さらに、

```bash
kubectl get pods -n mtls \
  -o custom-columns='NAME:.metadata.name,CONTAINERS:.spec.containers[*].name'
```

で、

```text
istio-proxy
```

が存在することを確認します。

---

# 設定全体を整理する

今回の構成は、

```text
mtls Namespace
        ↓
Sidecar Injection
        ↓
Pod作成
        ↓
Application + istio-proxy
        ↓
PeerAuthentication
        ↓
mTLS STRICT
        ↓
相互認証された通信だけを許可
```

となります。

Sidecar InjectionとPeerAuthenticationは、同じものではありません。

---

# Sidecar InjectionとPeerAuthenticationの違い

Sidecar Injectionは、

```text
Workloadの通信を
Istio Proxy経由にする
```

ための仕組みです。

一方、PeerAuthenticationは、

```text
その通信に
どの認証方式を要求するか
```

を制御します。

整理すると、

```text
Sidecar Injection
        ↓
通信を制御する場所を作る

PeerAuthentication
        ↓
通信に要求する認証ルールを決める
```

という関係です。

したがって、

```text
istio-proxyが存在する
        ↓
mTLS STRICTが必ず適用されている
```

とは限りません。

両方を確認する必要があります。

---

# NetworkPolicyとの違い

CKSでは、NetworkPolicyとの違いも整理しておくと理解しやすくなります。  
NetworkPolicyでは主に、

```text
誰から
    ↓
どこへ
    ↓
通信してよいか
```

を制御します。

例えば、

```text
frontend
    ↓
backend
```

だけを許可するといった制御です。

一方、今回のmTLSでは、

```text
通信相手は誰なのか
        +
通信内容を保護する
```

という観点があります。

つまり、

```text
NetworkPolicy
        ↓
通信経路の許可・拒否

mTLS
        ↓
通信相手のIdentity
        +
通信の暗号化
```

と整理できます。

---

# ケーススタディ

例えば、Namespace内に、

```text
frontend
backend
payment
```

という3つのサービスがあるとします。

NetworkPolicyによって、

```text
frontend
    ↓
backend
    ↓
payment
```

という通信経路だけを許可したとしても、  
**その通信自体の相手確認や暗号化とは別の問題です。**

そこでService Meshを組み合わせます。

```text
NetworkPolicy
        ↓
通信してよい経路を制御

Istio mTLS
        ↓
通信するWorkloadのIdentityを確認
        ↓
通信を暗号化
```

これによって、

```text
どこへ通信できるか
        +
誰と通信しているか
        +
通信内容を保護できているか
```

という異なる観点からWorkload間通信を保護できます。

---

# Zero Trustとの接続

今回の構成は、

```text
同じNamespaceだから信用する

同じClusterだから信用する
```

という考え方ではありません。

代わりに、

```text
通信要求
    ↓
相手のIdentityを確認
    ↓
自分のIdentityも提示
    ↓
双方を確認
    ↓
通信
```

という構造になります。

つまり、**ネットワーク上の場所だけを理由に通信相手を信用しない**  
という考え方へつながります。

---

# この問題で学んでほしいこと

この問題で覚えてほしいのは、

```text
istio-injection=enabled
```

や、

```yaml
kind: PeerAuthentication
spec:
  mtls:
    mode: STRICT
```

という設定値だけではありません。

重要なのは、

```text
Workload間通信を保護したい
        ↓
通信をService Meshの制御下へ置く
        ↓
Sidecar Proxy
        ↓
通信相手のIdentityを確認する
        ↓
mTLS
        ↓
相互認証されていない通信を拒否する
        ↓
STRICT
```

という判断の流れです。

さらに、

```text
NetworkPolicy
        ↓
どこからどこへ通信できるか

Service Mesh + mTLS
        ↓
誰と通信しているのか
        +
通信内容をどう保護するのか
```

という違いも理解してください。

CKSでは、**「PeerAuthenticationのYAMLを書ける」ことから一段進んで、  
Workload間通信をどこで制御し、通信相手のIdentityをどのように確認し、  
相互認証されていない通信をどう排除するかを判断できる」こと**  
をこの問題の到達点とします。
