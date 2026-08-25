# 問題07：Namespace間の通信経路を制限する ― 解説・模範解答

## この問題では何を考えればよかったのか

この問題では、Namespace間の通信について、

```text
どこから
    ↓
どこへの通信を
    ↓
許可するのか
```

をNetworkPolicyで制御します。

重要なのは、

```text
Namespaceが分かれている
    ↓
通信も分離されている
```

と考えないことです。

NamespaceはKubernetesリソースを管理上分離するための境界ですが、  
それだけでPod間通信が自動的に禁止されるわけではありません。

そのため、

```text
Workloadを分離した
        ↓
通信経路も制限したい
        ↓
どの通信を許可するか決める
        ↓
NetworkPolicy
```

という流れで考えます。

---

# 問題文を整理する

今回の通信要件は、大きく2つです。

```text
prod
 ↓
Ingressを許可しない
```

と、

```text
prod
 ↓
data
 ↓
許可

その他
 ↓
data
 ↓
許可しない
```

です。

整理すると次のようになります。

| 通信先    | 許可元    | 動作 |
| ------ | ------ | -- |
| `prod` | すべて    | 拒否 |
| `data` | `prod` | 許可 |
| `data` | その他    | 拒否 |

この状態をNetworkPolicyで作ります。

---

# 設計の順番

## 1. どの方向の通信を制御するか

今回制御するのは、

```text
Ingress
```

です。

つまり、

```text
Podから外へ出ていく通信
```

ではなく、

```text
Podへ入ってくる通信
```

を制御します。

```text
Source Pod
     ↓
Destination Pod
     ↑
   Ingress
```

今回のNetworkPolicyは、通信先となるPod側に作成します。

---

## 2. prod NamespaceへのIngressを拒否する

最初の要件は、

```text
prod Namespace内のすべてのPod
        ↓
Ingressを許可しない
```

ことです。

NetworkPolicyでは、まずPolicyの対象となるPodを選択します。

今回対象になるのは、

```text
prod Namespace内のすべてのPod
```

です。

そのため、

```yaml
podSelector: {}
```

を使用します。

空の`podSelector`は、そのNetworkPolicyが存在するNamespace内のすべてのPodを対象とします。

---

# Ingressを許可しない状態を作る

対象PodをIngress Policyの制御対象にしながら、許可するIngressルールを定義しなければ、

```text
対象Pod
 ↓
Ingress Isolationの対象
 ↓
許可されたIngressなし
 ↓
Ingress拒否
```

という状態になります。

したがって、`prod`側では次のNetworkPolicyを作成します。

---

# 模範解答1：prodへのIngressを拒否する

`deny-policy.yaml`を作成します。

```bash
vim deny-policy.yaml
```

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-policy
  namespace: prod
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

適用します。

```bash
kubectl apply -f deny-policy.yaml
```

---

# deny-policyを読む

## metadata.namespace

```yaml
namespace: prod
```

このNetworkPolicyは`prod` Namespaceに作成されます。

NetworkPolicyはNamespaceに属するリソースです。

---

## podSelector

```yaml
podSelector: {}
```

`prod` Namespace内のすべてのPodを対象にします。

つまり、

```text
prod
 ├─ Pod A
 ├─ Pod B
 └─ Pod C

すべてNetworkPolicyの対象
```

となります。

---

## policyTypes

```yaml
policyTypes:
- Ingress
```

Ingress方向の通信を制御します。

---

## ingressが存在しない意味

今回のPolicyには、

```yaml
ingress:
```

の許可ルールがありません。

そのため、

```text
Ingressを制御対象にする
        ↓
しかし許可するIngressがない
        ↓
受信通信を許可しない
```

という状態になります。

---

# 3. data Namespaceへの許可元を限定する

次の要件は、

```text
data Namespace内のすべてのPod
        ↓
prod NamespaceからだけIngressを許可
```

です。

ここでも、通信先は`data` Namespace内のすべてのPodなので、

```yaml
podSelector: {}
```

を使用します。

次に、

```text
誰からの通信を許可するのか
```

を指定します。

今回はPod個別のラベルではなく、

**Namespaceに付けられているラベル**

を使用します。

そのため、Namespace Selectorを利用します。

---

# Namespaceをラベルで識別する

今回提供されている条件は、

```text
env: prod
```

です。

そのため、

```yaml
namespaceSelector:
  matchLabels:
    env: prod
```

とすることで、該当ラベルを持つNamespaceからのIngressを許可します。

---

# 模範解答2：prodからdataへのIngressを許可する

`allow-from-prod.yaml`を作成します。

```bash
vim allow-from-prod.yaml
```

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-prod
  namespace: data
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          env: prod
```

適用します。

```bash
kubectl apply -f allow-from-prod.yaml
```

---

# allow-from-prodを読む

## namespace

```yaml
namespace: data
```

このPolicyが保護する通信先は`data` Namespaceです。

---

## podSelector

```yaml
podSelector: {}
```

`data` Namespace内のすべてのPodを対象にします。

---

## namespaceSelector

```yaml
namespaceSelector:
  matchLabels:
    env: prod
```

Ingress通信を許可するNamespaceをラベルで選択します。

つまり、

```text
Namespace
 ↓
env: prod
 ↓
許可元
```

という判定になります。

---

# 要件とNetworkPolicyの対応

| 要件                    | NetworkPolicy            |
| --------------------- | ------------------------ |
| `prod`内の全Podを対象       | `podSelector: {}`        |
| `prod`へのIngressを許可しない | Ingress Policy + 許可ルールなし |
| `data`内の全Podを対象       | `podSelector: {}`        |
| `prod`からのみ許可          | `namespaceSelector`      |
| Namespaceラベルで識別       | `matchLabels`            |
| 制御対象方向                | `policyTypes: Ingress`   |

この対応を、

```text
通信要件
    ↓
通信先
    ↓
通信方向
    ↓
許可元
    ↓
Selector
    ↓
NetworkPolicy
```

という順番で考えてください。

---

# NetworkPolicyを適用する

作成したPolicyを確認します。

```bash
kubectl get networkpolicy -n prod
```

期待するPolicyは、

```text
deny-policy
```

です。

`data`側も確認します。

```bash
kubectl get networkpolicy -n data
```

期待するPolicyは、

```text
allow-from-prod
```

です。

---

# Policyの詳細を確認する

`prod`側を確認します。

```bash
kubectl describe networkpolicy deny-policy -n prod
```

`data`側も確認します。

```bash
kubectl describe networkpolicy allow-from-prod -n data
```

次の点を確認します。

```text
どのPodを対象としているか
どの通信方向を制御しているか
どこからのIngressを許可しているか
```

---

# Namespaceのラベルを確認する

Namespace SelectorはNamespaceのラベルを使用するため、実際のラベルを確認します。

```bash
kubectl get namespace --show-labels
```

または、

```bash
kubectl get namespace prod --show-labels
```

などで確認できます。

今回の条件では、

```text
env=prod
```

が対象Namespaceに存在することを確認します。

---

# 通信を確認する

NetworkPolicyを適用した後は、実際の通信が要件通りになっていることを確認します。

確認したい状態は、

```text
他Namespace
    ↓
prod
    ×

prod
    ↓
data
    ○

その他Namespace
    ↓
data
    ×
```

です。

実際の検証方法は、環境に存在するPodやServiceに応じて異なります。

例えば通信元Podから、

```bash
kubectl exec -n <namespace> <pod> -- \
  curl <destination>
```

などを使用して確認できます。

---

# NetworkPolicyは「Denyルール」を書く機能なのか

ここは重要なポイントです。

NetworkPolicyでは、

```text
deny
allow
deny
allow
```

というFirewallルールを上から順番に評価する仕組みとして考えない方がよいです。

例えば今回の`deny-policy`は、

```yaml
podSelector: {}
policyTypes:
- Ingress
```

という構造です。

ここには、

```text
deny
```

という命令そのものはありません。

実際には、

```text
PodがIngress Policyの対象になる
        ↓
そのPodへのIngressは
明示的に許可されたものだけになる
        ↓
許可ルールが存在しない
        ↓
結果としてIngressを受け付けない
```

という仕組みです。

---

# 複数のNetworkPolicyが存在した場合

同じPodに複数のNetworkPolicyが適用される場合、

```text
Policy A
 ↓
Source Aを許可

Policy B
 ↓
Source Bを許可
```

なら、基本的な考え方は、

```text
許可される通信
 ↓
Source A
+
Source B
```

です。

つまり、

**NetworkPolicyは、複数Policyの許可条件を組み合わせて考える**

必要があります。

そのため、

```text
Default Deny Policyを作った
        ↓
別のPolicyで必要な通信を許可
```

という設計も可能です。

---

# Namespace SelectorとPod Selectorの違い

NetworkPolicyでは、

```text
どこから来た通信なのか
```

を複数の方法で識別できます。

今回使用しているのは、

```yaml
namespaceSelector:
```

です。

これは、

```text
どのNamespaceから来たのか
```

を判定します。

一方、

```yaml
podSelector:
```

をIngressの`from`で利用すれば、

```text
どのPodから来たのか
```

をPodラベルによって判定できます。

整理すると、

```text
namespaceSelector
    ↓
Namespaceを選択

podSelector
    ↓
Podを選択
```

です。

---

# ケーススタディ

## なぜNamespaceだけでは通信境界にならないのか

例えば、

```text
frontend
backend
database
```

を別Namespaceへ分離したとします。

これは管理上は分離されています。  
しかし、ネットワーク通信まで制御していなければ、

```text
侵害されたfrontend Pod
        ↓
backendを探索
        ↓
databaseを探索
```

という通信が可能になる場合があります。

そのため、

```text
Namespace
        ↓
管理境界

NetworkPolicy
        ↓
通信境界
```

と役割を分けて考えます。

---

# Lateral Movementを抑える

攻撃者が一つのPodへ侵入した場合、

```text
Pod侵害
 ↓
Cluster内部を探索
 ↓
別Serviceへ接続
 ↓
認証情報などを取得
 ↓
さらに別Workloadへ侵入
```

という形で影響範囲が広がる可能性があります。

このような横方向への侵害拡大を、**Lateral Movement（横展開）** と呼びます。  
NetworkPolicyによって、

```text
侵害されたPod
        ↓
通信可能な範囲を限定
```

しておくことで、侵害後の可動範囲を小さくできます。

---

# 信頼境界として考える

今回の要件をセキュリティの視点から見ると、

```text
prod
 ↓
信頼する通信元

data
 ↓
保護する対象
```

という関係になります。

つまりNetworkPolicyは、

```text
誰からの通信を信頼するのか
```

をKubernetes上で表現する仕組みとして考えることもできます。

今回なら、

```text
data
 ↓
prodからの通信
 ↓
許可

それ以外
 ↓
許可しない
```

という信頼境界を作っています。

---

# この問題で学んでほしいこと

この問題で覚えてほしいのは、

```yaml
podSelector: {}
```

や、

```yaml
namespaceSelector:
  matchLabels:
    env: prod
```

という記述だけではありません。

重要なのは、

```text
何を守るのか
    ↓
どの方向の通信を制御するのか
    ↓
誰からの通信を許可するのか
    ↓
どのラベルで識別するのか
    ↓
NetworkPolicy
```

という考え方です。

また、

```text
Namespaceを分離した
```

ことと、

```text
ネットワーク通信を分離した
```

ことは同じではありません。

今回のように、

```text
Defaultで通信範囲を小さくする
        ↓
必要な経路だけを許可する
```

という設計によって、Workload間の通信境界を作ることができます。

CKSでは、**「NetworkPolicyのYAMLを書ける」ことから一段進んで、  
「どのWorkload間を信頼し、どの通信経路だけを残すべきかを判断できる」こと**  
をこの問題の到達点とします。
