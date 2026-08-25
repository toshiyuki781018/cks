# 問題09：ServiceAccount Tokenの自動配布を制限する ― 解説・模範解答

## この問題では何を考えればよかったのか

この問題では、Podへ提供するKubernetes APIの認証情報を制御します。

重要なのは、

```text
ServiceAccount Tokenは危険
        ↓
すべて削除する
```

と考えることではありません。

今回のWorkloadではServiceAccount Tokenが必要です。

問題となっているのは、

```text
ServiceAccountを使用する
        ↓
Tokenが自動的にPodへ提供される
```

という状態です。

そこで、

```text
自動的なToken配布
        ↓
無効化

必要なWorkload
        ↓
必要なTokenを明示的に提供
```

という構成へ変更します。

つまりこの問題では、**「認証情報を使用するか」ではなく、  
「認証情報をどのようにPodへ提供するか」**を考えます。

---

# 問題文を整理する

今回対象となるのは、

```text
Namespace
  monitoring

ServiceAccount
  stats-monitor-sa

Deployment
  stats-monitor
```

です。

Deploymentは引き続き、

```text
stats-monitor-sa
```

を使用します。

一方で、ServiceAccount Tokenの自動マウントは使用しません。  
しかし、アプリケーション自体はTokenを必要としています。

したがって、

```text
ServiceAccount Token
        ↓
自動マウント
        ×

ServiceAccount Token
        ↓
明示的なVolume
        ○
```

という状態を作る必要があります。

---

# 設計の順番

## 1. ServiceAccount側で自動マウントを無効化する

最初に、

```text
stats-monitor-sa
```

を使用するPodへAPI認証情報が自動的に提供されないようにします。

ServiceAccountでは、

```yaml
automountServiceAccountToken: false
```

を設定します。

これによって、このServiceAccountを使用するPodについて、  
ServiceAccount Tokenを自動マウントしないことをデフォルトの動作とします。

---

## 2. Deployment側でも自動マウントを無効化する

今回のDeploymentについても、

```yaml
automountServiceAccountToken: false
```

をPod Templateへ設定します。

つまり、

```text
ServiceAccount
    ↓
自動マウントしない

Deployment / Pod
    ↓
自動マウントしない
```

という状態にします。

---

## 3. 必要なTokenだけを明示的に提供する

しかし、`stats-monitor`はServiceAccount Tokenを必要としています。

そのため、

```text
自動マウントをOFF
        ↓
Tokenが必要
        ↓
明示的にVolumeとして提供
```

します。

今回使用するのが、

**Projected Volume**

です。

ServiceAccount TokenをVolumeへ投影し、そのVolumeをコンテナへマウントします。

---

# ServiceAccountの模範解答

対象ServiceAccountを編集します。

```bash
kubectl edit serviceaccount -n monitoring stats-monitor-sa
```

ServiceAccountへ次の設定を追加します。

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: stats-monitor-sa
  namespace: monitoring
automountServiceAccountToken: false
```

重要なのは、

```yaml
automountServiceAccountToken: false
```

です。

既存ServiceAccountのその他の情報を不要に変更する必要はありません。

---

# ServiceAccount側の設定を確認する

設定後、次のコマンドで確認できます。

```bash
kubectl get serviceaccount stats-monitor-sa \
  -n monitoring \
  -o yaml
```

次の設定が存在することを確認します。

```yaml
automountServiceAccountToken: false
```

---

# Deploymentの模範解答

対象マニフェストを編集します。

```bash
vim ~/stats-monitor/deployment.yaml
```

必要な設定を追加すると、主要部分は次のようになります。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stats-monitor
  namespace: monitoring
  labels:
    app: stats-monitor
spec:
  replicas: 1
  selector:
    matchLabels:
      app: stats-monitor
  template:
    metadata:
      labels:
        app: stats-monitor
    spec:
      serviceAccountName: stats-monitor-sa
      automountServiceAccountToken: false

      containers:
      - name: nginx
        image: vicuu/nginx:hello
        imagePullPolicy: IfNotPresent

        volumeMounts:
        - name: token
          mountPath: /var/run/secrets/kubernetes.io/serviceaccount/token
          readOnly: true

        - name: httpcf
          mountPath: /usr/local/apache2/conf/httpd.conf
          subPath: httpd.conf

      volumes:
      - name: token
        projected:
          sources:
          - serviceAccountToken:
              path: token

      - name: httpcf
        configMap:
          name: httpcf
          items:
          - key: httpd.conf
            path: httpd.conf
```

---

# Deploymentの設定を読む

## serviceAccountName

```yaml
serviceAccountName: stats-monitor-sa
```

Deploymentが使用するServiceAccountを指定しています。

今回の要件では、ServiceAccountそのものを変更するのではなく、

```text
stats-monitor
    ↓
stats-monitor-sa
```

という既存の関係を維持します。

---

# automountServiceAccountToken

```yaml
automountServiceAccountToken: false
```

PodへServiceAccountのAPI認証情報が自動的にマウントされることを無効化します。

つまり、

```text
ServiceAccountを使用する
        ↓
Tokenが自動的に提供される
```

という動作を停止します。

---

# Projected Volume

今回の重要な部分が次の設定です。

```yaml
volumes:
- name: token
  projected:
    sources:
    - serviceAccountToken:
        path: token
```

ここでは、

```text
ServiceAccount Token
        ↓
Projected Volume
        ↓
token
```

というVolumeを作っています。

---

# projectedとは何か

Projected Volumeは、複数の情報源を1つのVolumeへ投影するために利用できます。

今回使用する情報源は、

```yaml
serviceAccountToken:
```

です。

したがって、

```text
ServiceAccount
        ↓
Token
        ↓
Projection
        ↓
Volume
```

という構造になります。

---

# pathの意味

```yaml
serviceAccountToken:
  path: token
```

ここで指定している`path`は、Projected Volume内部に作成されるファイル名です。

つまり、

```text
Projected Volume
        ↓
token
```

というファイルが作られます。

---

# Volume Mount

作成した`token` Volumeをコンテナへマウントします。

```yaml
volumeMounts:
- name: token
  mountPath: /var/run/secrets/kubernetes.io/serviceaccount/token
  readOnly: true
```

これによって、

```text
Projected Volume
        ↓
token
        ↓
Container
        ↓
/var/run/secrets/kubernetes.io/serviceaccount/token
```

からTokenを参照できるようにします。

---

# 読み取り専用にする

今回のTokenは認証情報です。

コンテナから変更する必要はありません。

そのため、

```yaml
readOnly: true
```

としてマウントします。

考え方としては、

```text
認証情報
    ↓
アプリケーションから参照する
    ↓
書き換える必要はない
    ↓
Read Only
```

です。

---

# 要件と設定の対応

| 要件                       | Kubernetes設定                           |
| ------------------------ | -------------------------------------- |
| ServiceAccount側の自動マウント停止 | `automountServiceAccountToken: false`  |
| Pod側の自動マウント停止            | `automountServiceAccountToken: false`  |
| 使用するServiceAccountを維持    | `serviceAccountName: stats-monitor-sa` |
| Tokenを明示的に提供             | Projected Volume                       |
| TokenをVolumeへ投影          | `serviceAccountToken`                  |
| Volume名                  | `token`                                |
| Tokenファイル                | `path: token`                          |
| 指定パスへマウント                | `mountPath`                            |
| 読み取り専用                   | `readOnly: true`                       |

この対応を、

```text
認証情報を自動配布したくない
        ↓
automountを無効化

しかしTokenは必要
        ↓
ServiceAccount TokenをProjection

コンテナから利用したい
        ↓
Volume Mount

変更させる必要はない
        ↓
Read Only
```

という順番で理解してください。

---

# Deploymentを適用する

編集したマニフェストを適用します。

```bash
kubectl apply -f ~/stats-monitor/deployment.yaml
```

Podを確認します。

```bash
kubectl get pods -n monitoring
```

Deployment更新後のPodが、

```text
Running
```

になっていることを確認します。

---

# Deploymentの設定を確認する

```bash
kubectl get deployment stats-monitor \
  -n monitoring \
  -o yaml
```

次の設定を確認します。

```yaml
serviceAccountName: stats-monitor-sa
automountServiceAccountToken: false
```

さらに、

```yaml
volumes:
- name: token
  projected:
```

が存在することも確認します。

---

# Tokenのマウントを確認する

Pod名を取得します。

```bash
kubectl get pods -n monitoring
```

コンテナ内部を確認します。

```bash
kubectl exec -n monitoring <Pod名> -- \
  ls -l /var/run/secrets/kubernetes.io/serviceaccount/token
```

Tokenが指定された場所から参照できることを確認します。

---

# なぜ自動マウントを無効化するのか

ServiceAccount TokenはKubernetes APIへアクセスするための認証情報です。

例えばPodが侵害された場合、

```text
Pod侵害
 ↓
Filesystemを探索
 ↓
ServiceAccount Tokenを発見
 ↓
Kubernetes APIへアクセス
```

という経路が生まれる可能性があります。

もちろん、そのTokenで実際に何ができるかはRBACなどのAuthorization設定によって決まります。

しかし、

```text
そもそも必要のないTokenを
Pod内部へ置かない
```

ことで、認証情報そのものの露出範囲を小さくできます。

---

# 「使わない」と「必要なものだけ渡す」は違う

今回の問題で特に重要なのがここです。

単純な対策なら、

```text
ServiceAccount Token
        ↓
全部マウントしない
```

でも構いません。

しかし、実際のアプリケーションではKubernetes APIへアクセスする必要がある場合があります。

そのため、

```text
Tokenが不要なWorkload
        ↓
渡さない

Tokenが必要なWorkload
        ↓
必要な形で明示的に渡す
```

という設計が必要になります。

つまり、**「ゼロか全部か」ではなく、  
必要性に応じて認証情報の配布範囲を制御する**という考え方です。

---

# ServiceAccountとPodの設定場所の違い

`automountServiceAccountToken`は、ServiceAccount側とPod側の両方で設定できます。

概念的には、

```text
ServiceAccount
    ↓
このServiceAccountを使用するときの
自動マウント方針

Pod
    ↓
このPod自身の
自動マウント方針
```

と整理できます。

両方に値が設定されている場合は、Pod側の設定が優先されます。

そのため、

```text
ServiceAccount
automountServiceAccountToken: false

Pod
automountServiceAccountToken: true

        ↓

Pod側の指定が優先
```

となります。

今回では両方を`false`とすることで、自動配布を使用しない方針を明示しています。

---

# Projected ServiceAccount Tokenとして考える

今回のTokenは、

```text
ServiceAccount
 ↓
Token
 ↓
Projected Volume
 ↓
Pod
```

として提供します。

これは、

```text
ServiceAccountを使った
    ↓
Tokenが勝手に置かれた
```

という構成とは違います。

アプリケーションの要件を確認したうえで、

```text
このWorkloadにはTokenが必要
        ↓
明示的にTokenをProjection
```

しています。

つまり、**認証情報の存在をWorkload設計の一部として扱っている**わけです。

---

# ケーススタディ

## Tokenを制御してもRBACは必要

ServiceAccount Tokenを必要なPodだけに提供すれば、それだけですべてのセキュリティ問題が解決するわけではありません。

ServiceAccount Tokenは、

```text
あなたは誰か
```

をKubernetes APIへ示すための認証情報です。

一方、

```text
そのユーザーが何をしてよいか
```

は別の仕組みで制御します。

整理すると、

```text
ServiceAccount Token
        ↓
Authentication
        ↓
誰なのか

RBAC
        ↓
Authorization
        ↓
何をしてよいのか
```

です。

したがって、

```text
Tokenの露出範囲を小さくする
        +
RBACで権限を小さくする
```

という両方の対策が重要になります。

---

# コンテナ侵害後の影響範囲として考える

例えば、Webアプリケーションに脆弱性があり、攻撃者がコンテナ内でコマンドを実行できたとします。

自動マウントによってTokenが存在すると、

```text
Application侵害
      ↓
Container内へ侵入
      ↓
ServiceAccount Tokenを取得
      ↓
Kubernetes APIへアクセス
```

という次の攻撃経路が生まれる可能性があります。

不要なTokenを配置しなければ、

```text
Container侵害
      ↓
利用可能な認証情報なし
```

として、侵害後に利用できる選択肢を減らせます。

これは、

**侵入そのものを防ぐ対策ではなく、侵入された後の可動範囲を小さくする対策**

として考えることができます。

---

# 第7問との接続

第7問ではNetworkPolicyを使用して、

```text
侵害されたPod
    ↓
どこへ通信できるか
```

を制限しました。

今回のServiceAccount Tokenでは、

```text
侵害されたPod
    ↓
どの認証情報を取得できるか
```

を制限しています。

整理すると、

```text
NetworkPolicy
    ↓
通信可能範囲を制限

ServiceAccount Token
    ↓
認証情報の露出範囲を制限

RBAC
    ↓
API操作可能範囲を制限
```

となります。

複数の制御を組み合わせることで、コンテナが侵害された場合の影響範囲を小さくできます。

---

# この問題で学んでほしいこと

この問題で覚えてほしいのは、

```yaml
automountServiceAccountToken: false
```

や、

```yaml
projected:
  sources:
  - serviceAccountToken:
      path: token
```

というYAMLだけではありません。

重要なのは、

```text
Podへ認証情報が存在する
        ↓
本当に必要なのか
        ↓
不要なら渡さない

必要なら
        ↓
自動的に配布するのではなく
必要な形で明示的に提供する
```

という考え方です。

つまり、

```text
認証情報
    ↓
誰に渡すのか
    ↓
なぜ必要なのか
    ↓
どのように渡すのか
    ↓
渡した認証情報で何ができるのか
```

まで考える必要があります。

CKSでは、  
**「ServiceAccount Tokenの自動マウントを無効化できる」ことから一段進んで、  
「Kubernetes APIの認証情報を必要なWorkloadだけへ明示的に提供し、その露出範囲を制御できる」こと**  
をこの問題の到達点とします。
