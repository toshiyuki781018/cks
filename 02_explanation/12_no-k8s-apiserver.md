# 問題12：Kubernetes APIへの未認証アクセスを禁止する ― 解説・模範解答

## この問題では何を考えればよかったのか

この問題では、Kubernetes APIへのアクセスを、

```text
誰でもアクセスできる状態
        ↓
正規に認証された主体だけがアクセスできる状態
```

へ変更します。

ただし、匿名認証だけを無効化すれば終わりではありません。  
Kubernetes APIへの要求は、

```text
API Request
    ↓
Authentication
    ↓
Authorization
    ↓
Admission Control
    ↓
処理
```

という複数の段階で制御されます。

今回の要件では、

```text
未認証アクセス
    ↓
禁止

認証された主体
    ↓
Node / RBACで認可

Nodeからの要求
    ↓
NodeRestrictionでも制御
```

という状態を作ります。

さらに、匿名アクセスを前提として残されている不要なClusterRoleBindingも削除します。

つまり、**「認証 → 認可 → Admission」というAPI Serverのセキュリティ境界を整理する**  
ことが、この問題のポイントです。

---

# 問題文を整理する

今回必要になる制御は次の4つです。

| セキュリティ要件     | 実現したい状態                                    |
| ------------ | ------------------------------------------ |
| 未認証アクセスを禁止   | 匿名ユーザーとしてAPIへアクセスさせない                      |
| API操作を認可     | NodeとRBACを使用する                             |
| Nodeからの操作を制限 | NodeRestrictionを使用する                       |
| 不要な匿名権限を削除   | `system:anonymous` ClusterRoleBindingを削除する |

さらに、API Serverを安全な状態へ変更すると、  
現在のkubectl設定が利用できなくなる可能性があります。

そのため、

```text
API ServerをHardening
        ↓
現在のkubectl設定が利用不能
        ↓
正規の管理者用kubeconfigを使用
```

という切り替えも必要です。

---

# 設計の順番

## 1. 現在のkube-apiserver設定を確認する

kube-apiserverはStatic Podとして構成されています。  
対象マニフェストは、

```text
/etc/kubernetes/manifests/kube-apiserver.yaml
```

です。

まず、現在の設定を確認します。

今回変更するのは、

```text
Authentication
Authorization
Admission Control
```

に関係する部分です。

その他の設定は変更しません。

---

## 2. 設定変更前にバックアップする

kube-apiserverはControl Planeの中核コンポーネントです。

設定を誤ると、

```text
kube-apiserver起動失敗
        ↓
Kubernetes API利用不能
        ↓
kubectlも利用できない
```

という状態になる可能性があります。

そのため、元問題でも変更前にマニフェストをコピーしています。

```bash
cp /etc/kubernetes/manifests/kube-apiserver.yaml /opt/
```

---

# 匿名認証を無効化する

今回の最初の要件は、

```text
認証情報を持たないクライアントから
Kubernetes APIへアクセスさせない
```

ことです。

kube-apiserverでは、匿名認証の動作を制御できます。

必要な設定は、

```yaml
- --anonymous-auth=false
```

です。

これによって、

```text
認証情報なし
    ↓
匿名ユーザーとして認証
    ↓
API利用
```

という経路を無効化します。

---

# Authorization Modeを設定する

次に、

```text
認証された主体が
何をしてよいのか
```

を制御します。

今回要求されているAuthorization Modeは、

```text
Node
RBAC
```

です。

kube-apiserverでは、

```yaml
- --authorization-mode=Node,RBAC
```

と設定します。

すでにこの値になっている場合は、変更する必要はありません。  
元問題でもその点が明示されています。

---

# AuthenticationとAuthorizationの違い

ここは重要です。

```text
Authentication
        ↓
あなたは誰ですか？

Authorization
        ↓
あなたは何をしてよいですか？
```

という違いがあります。

例えば、

```text
証明書を提示
    ↓
system:node:worker01として認証

その後
    ↓
このNodeはこのAPI操作を
実行してよいのか？
```

という形です。

したがって、

```text
匿名認証を禁止した
        ↓
すべてのAPI Securityが完了
```

ではありません。

認証後にAuthorizationが必要です。

---

# Node Authorization

今回使用する1つ目のAuthorization Modeが、

```text
Node
```

です。

Node Authorizationは、kubeletからのAPI要求を認可するための仕組みです。

概念的には、

```text
kubelet
    ↓
Nodeとして認証
    ↓
Node Authorizer
    ↓
そのNodeに必要なAPI操作を認可
```

という役割を持ちます。

---

# RBAC Authorization

もう1つが、

```text
RBAC
```

です。

RBACでは、

```text
誰に
    ↓
どのResourceについて
    ↓
どの操作を許可するか
```

をRoleやClusterRole、RoleBinding、ClusterRoleBindingなどによって制御します。

つまり、

```text
Authentication
    ↓
User / ServiceAccountなどを特定
    ↓
RBAC
    ↓
許可された操作だけ実行
```

という関係です。

---

# NodeRestrictionを有効化する

次の要件は、

```text
NodeからKubernetes APIへ行われる操作について
Nodeに許可された範囲を超える変更を制限する
```

ことです。

今回使用するAdmission Pluginが、

```text
NodeRestriction
```

です。

kube-apiserverでは、

```yaml
- --enable-admission-plugins=NodeRestriction
```

とします。

元問題では、既存のAdmission Plugin設定を  
`NodeRestriction`へ修正する構成になっています。

---

# Node AuthorizerとNodeRestrictionの違い

名前が似ていますが、役割は異なります。

```text
Node Authorizer
        ↓
NodeがそのAPI操作を
実行してよいか認可する

NodeRestriction
        ↓
Nodeから送られた要求について
変更可能な対象をさらに制限する
```

つまり、

```text
kubelet
    ↓
Authentication
    ↓
Node Authorization
    ↓
NodeRestriction
    ↓
API処理
```

という形で複数の制御を組み合わせます。

---

# kube-apiserverの模範設定

対象ファイルを編集します。

```bash
vim /etc/kubernetes/manifests/kube-apiserver.yaml
```

今回確認・修正する主要部分は次の通りです。

```yaml
spec:
  containers:
  - command:
    - kube-apiserver

    # 既存設定
    - --advertise-address=<既存のIPアドレス>
    - --allow-privileged=true

    # Authorization
    - --authorization-mode=Node,RBAC

    # Admission Control
    - --enable-admission-plugins=NodeRestriction

    # Authentication
    - --anonymous-auth=false

    # その他の既存設定
    ...
```

今回重要なのは、

```yaml
--anonymous-auth=false
--authorization-mode=Node,RBAC
--enable-admission-plugins=NodeRestriction
```

の3点です。

その他のkube-apiserver設定は既存の値を維持します。

---

# 要件とkube-apiserver設定の対応

| 要件           | kube-apiserver設定                             |
| ------------ | -------------------------------------------- |
| 匿名アクセス禁止     | `--anonymous-auth=false`                     |
| Nodeによる認可    | `--authorization-mode=Node,RBAC`             |
| RBACによる認可    | `--authorization-mode=Node,RBAC`             |
| Nodeからの操作を制限 | `--enable-admission-plugins=NodeRestriction` |

この対応を、

```text
未認証アクセスをどうする？
        ↓
Authentication

認証後に何を許可する？
        ↓
Authorization

Nodeからの要求をさらにどう制限する？
        ↓
Admission Control
```

として理解してください。

---

# kube-apiserverへ設定を反映する

元問題では、マニフェスト編集後に次の操作を行っています。

```bash
systemctl daemon-reload
systemctl restart kubelet.service
```

kube-apiserverはStatic Podなので、マニフェストの変更によって再作成されます。

設定変更後は、API Serverが正常に復帰したことを確認します。

---

# なぜ通常のkubectlが使えなくなるのか

ここが今回の問題で非常に重要です。

元問題では、

> すべてのkubectl設定環境／ファイルも、認証されていないアクセスや不正なアクセスを許可するように設定されている

とされています。

つまり、これまでのkubectl操作が、

```text
API Serverが匿名アクセスを許可
        ↓
kubectlからアクセス可能
```

という状態に依存しています。

匿名認証を禁止すると、

```text
kubectl
    ↓
正規の認証情報なし
    ↓
API Server
    ↓
Authentication失敗
```

となります。

これは設定失敗ではありません。

**今回のセキュリティ設定が機能した結果です。**

---

# 正規の管理者用kubeconfigを使用する

Control Planeには、正規の管理者用kubeconfigが存在します。

```text
/etc/kubernetes/admin.conf
```

以降の管理操作では、このkubeconfigを明示的に使用します。

例えば、

```bash
kubectl --kubeconfig /etc/kubernetes/admin.conf get nodes
```

とします。

正常であれば、認証済みの管理者としてNode情報を取得できます。

---

# ClusterRoleBindingを削除する

API Server側で匿名アクセスを禁止した後、匿名アクセス用として残されている不要なClusterRoleBindingを削除します。

対象は、

```text
system:anonymous
```

です。

管理者用kubeconfigを使用して削除します。

```bash
kubectl --kubeconfig /etc/kubernetes/admin.conf \
  delete clusterrolebinding system:anonymous
```

元問題ではリソース名を完全修飾した、

```bash
kubectl --kubeconfig /etc/kubernetes/admin.conf \
  delete clusterrolebindings.rbac.authorization.k8s.io system:anonymous
```

という形式が示されています。

どちらも、対象となるClusterRoleBindingを指定して削除する考え方は同じです。

---

# なぜClusterRoleBindingも削除するのか

ここでは、

```text
匿名認証
```

と、

```text
匿名ユーザーへ与えられた権限
```

を分けて考える必要があります。

API Server側では、

```text
--anonymous-auth=false
        ↓
匿名ユーザーとして認証しない
```

ようにしています。

一方、ClusterRoleBindingは、

```text
特定の主体
    ↓
ClusterRole
    ↓
権限を与える
```

ためのAuthorization設定です。

そのため今回、

```text
Authentication
    ↓
匿名アクセス禁止

Authorization
    ↓
匿名アクセス用の不要なBindingを削除
```

という両方を整理します。

---

# 作業後の確認

まず、管理者用kubeconfigでクラスターへアクセスできることを確認します。

```bash
kubectl --kubeconfig /etc/kubernetes/admin.conf get nodes
```

続いて、Control Planeを含むPodを確認します。

```bash
kubectl --kubeconfig /etc/kubernetes/admin.conf get pods -A
```

kube-apiserverを含むControl Planeコンポーネントが正常であることを確認します。

---

# kube-apiserver設定を確認する

Static Podマニフェストを確認します。

```bash
grep -E \
'anonymous-auth|authorization-mode|enable-admission-plugins' \
/etc/kubernetes/manifests/kube-apiserver.yaml
```

要件に対応する設定が確認できればよいでしょう。

---

# ClusterRoleBindingの削除を確認する

```bash
kubectl --kubeconfig /etc/kubernetes/admin.conf \
  get clusterrolebinding system:anonymous
```

削除されていれば、対象のClusterRoleBindingは取得できません。

---

# API Serverの入口として考える

今回の設定を全体で見ると、

```text
Client
  ↓
Kubernetes API
  ↓
Authentication
  │
  │ 匿名アクセス禁止
  ↓
Authorization
  │
  │ Node / RBAC
  ↓
Admission Control
  │
  │ NodeRestriction
  ↓
Resource
```

となります。

それぞれ役割が違います。

---

# Authenticationだけでは足りない

例えば、正規の証明書を持つユーザーがいたとします。

Authenticationだけなら、

```text
証明書
    ↓
本人確認成功
```

で終わります。

しかし、

```text
本人だと分かった
        ↓
何でも操作してよい
```

わけではありません。

そこで、

```text
Authentication
        ↓
誰なのか

Authorization
        ↓
何をしてよいのか
```

を分離します。

これはKubernetes API Securityを理解するうえで基本となる考え方です。

---

# ケーススタディ

## API Serverを外部へ公開する場合

例えば、Control PlaneのAPI Endpointへ複数の管理者やシステムからアクセスする環境を考えます。

```text
Administrator
CI/CD
Node
Controller
        ↓
Kubernetes API
```

それぞれ必要な権限は異なります。

そのため、

```text
APIへ到達できる
        ↓
APIを自由に操作できる
```

という設計にはしません。

まず認証し、

```text
誰なのか
```

を確認します。

次に認可によって、

```text
その主体が何をしてよいのか
```

を制御します。

さらに必要に応じてAdmission Controlで、

```text
その要求自体を
クラスターへ受け入れてよいのか
```

を判断します。

---

# 第11問との接続

第11問ではImagePolicyWebhookを使用しました。

第11問では、

```text
認証
 ↓
認可
 ↓
Admission
 ↓
ImagePolicyWebhook
 ↓
このImageを受け入れてよいか
```

を判断しました。

今回のNodeRestrictionもAdmission Controlです。

つまり、

```text
Authentication
    ↓
主体を確認

Authorization
    ↓
操作権限を確認

Admission
    ↓
要求内容をさらに検査
```

というAPI Serverの処理構造が、第11問と第12問でつながります。

---

# この問題で学んでほしいこと

この問題で覚えてほしいのは、

```yaml
--anonymous-auth=false
--authorization-mode=Node,RBAC
--enable-admission-plugins=NodeRestriction
```

という3つのオプションだけではありません。

重要なのは、

```text
APIへ要求が来る
        ↓
誰なのか？
        ↓
Authentication

何をしてよいのか？
        ↓
Authorization

その要求を受け入れてよいのか？
        ↓
Admission Control
```

という制御の流れです。

さらに、

```text
匿名アクセスを禁止
        ↓
今まで使えていたkubectlが使えない
        ↓
なぜ？
        ↓
正規の認証情報を持っていないから
        ↓
admin.confを使用
        ↓
認証済み管理者として操作
```

という現象まで理解できれば、単なる設定暗記ではなくなります。

CKSでは、

**「匿名認証を無効化するコマンドを知っている」ことから一段進んで、  
「Kubernetes APIへの要求がAuthentication・Authorization・Admission Controlをどのように通過し、  
どの段階で何を制御しているのかを理解したうえでAPI ServerをHardeningできる」こと**  
をこの問題の到達点とします。
