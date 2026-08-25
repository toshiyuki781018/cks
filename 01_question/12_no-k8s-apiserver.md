# 問題12：Kubernetes APIへの未認証アクセスを禁止する

## 対象領域

Cluster Setup / Kubernetes API Security / Authentication / Authorization / Admission Control

## 教材種別

Drill（試験対策問題）

目安時間：10〜15分

# 背景

kubeadmで構築されたKubernetesクラスターにおいて、API Serverのセキュリティ設定を確認したところ、  
テスト目的で未認証クライアントからのアクセスを許可する構成が残っていることが判明しました。

さらに、匿名ユーザーからのアクセスを許可するための権限設定もクラスター内に残されています。  
この状態では、正規の認証情報を持たないクライアントからKubernetes APIへアクセスされる可能性があります。

Control Planeの設定を修正し、Kubernetes APIへアクセスする  
クライアントに適切な認証・認可を要求する構成へ変更してください。

また、NodeからKubernetes APIへ行われる操作についても、  
Nodeに許可された範囲を超える変更が行われないようにしてください。

# 初期観測情報

kube-apiserverはStatic Podとして構成されています。

マニフェストは次の場所にあります。

```text
/etc/kubernetes/manifests/kube-apiserver.yaml
```

現在のkube-apiserverには、テスト目的で未認証アクセスを許可する設定が含まれています。

また、クラスターには匿名アクセス用として次のClusterRoleBindingが存在します。

```text
system:anonymous
```

現在使用しているkubectl設定も、未認証アクセスが許可されていることを前提とした構成になっています。

API Serverを安全な構成へ変更した後は、現在のkubectl設定ではクラスターを操作できなくなる可能性があります。

Control Planeには、クラスター管理者用の正規のkubeconfigが次の場所に存在します。

```text
/etc/kubernetes/admin.conf
```

# 要件

以下のセキュリティ要件をすべて満たしてください。

1. Kubernetes APIへ認証情報を持たないクライアントがアクセスできないようにすること。
2. APIへの操作権限について、NodeおよびRBACによる認可方式を使用すること。
3. NodeからKubernetes APIへ行われる操作について、Nodeに許可された範囲を超える変更を制限すること。
4. kube-apiserverの既存設定のうち、今回のセキュリティ要件と関係のない設定は維持すること。
5. kube-apiserverの設定変更後も、API Serverが正常に稼働すること。
6. API Serverを安全な構成へ変更した後は、正規の管理者用認証情報を使用してクラスターを操作すること。
7. 匿名アクセスのために残されている不要なClusterRoleBindingを削除すること。
8. 設定変更後、認証済みの管理者としてクラスターを正常に操作できること。

# 設定情報

| 項目                     | 設定内容                                            |
| ---------------------- | ----------------------------------------------- |
| kube-apiserverマニフェスト   | `/etc/kubernetes/manifests/kube-apiserver.yaml` |
| Authorization          | `Node`, `RBAC`                                  |
| Nodeに対するAdmission制御    | `NodeRestriction`                               |
| 管理者用kubeconfig         | `/etc/kubernetes/admin.conf`                    |
| 削除対象ClusterRoleBinding | `system:anonymous`                              |

# 制約

* kube-apiserverの既存設定について、今回の要件と関係のない項目を変更してはいけません。
* Kubernetes APIの認証そのものを無効化してはいけません。
* APIへ未認証でアクセスできる状態を残してはいけません。
* Authorizationを無効化してはいけません。
* Nodeに対するAdmission制御を無効化してはいけません。
* `/etc/kubernetes/admin.conf`を変更してはいけません。
* `system:anonymous`以外のClusterRoleBindingを削除してはいけません。
* kube-apiserverを通常のPodやDeploymentとして再作成してはいけません。

# 完了条件

以下をすべて満たした場合、作業完了とします。

* Kubernetes APIへの匿名アクセスが許可されていない。
* API ServerでNodeおよびRBACによるAuthorizationが使用されている。
* Nodeに対する必要なAdmission制御が有効になっている。
* kube-apiserverが正常に稼働している。
* 管理者用の正規のkubeconfigを使用してKubernetes APIへアクセスできる。
* `system:anonymous` ClusterRoleBindingが削除されている。
* 認証済みの管理者としてクラスターのリソースを正常に取得できる。
* 今回の要件と関係のないkube-apiserver設定が維持されている。

# 作業後の確認項目

設定後、次の内容を確認してください。

* kube-apiserverが正常に起動していること。
* Kubernetes APIへの匿名アクセスが無効になっていること。
* API ServerのAuthorization方式が要件を満たしていること。
* Nodeに対するAdmission制御が有効になっていること。
* 管理者用kubeconfigを使用してクラスターへアクセスできること。
* `system:anonymous` ClusterRoleBindingが存在しないこと。
* Control Planeを含むクラスターのPodが正常な状態を維持していること。

# 注意事項

今回の目的は、

**「匿名アクセスに関する設定を1つ変更する」**

ことだけではありません。

Kubernetes APIへアクセスする場合、

```text
API Request
    ↓
Authentication
    ↓
誰からの要求なのか
    ↓
Authorization
    ↓
そのユーザーが何をしてよいのか
    ↓
Admission Control
    ↓
その要求を受け入れてよいのか
```

という複数の制御があります。

今回のクラスターでは、最初の入口であるAuthenticationにおいて、

```text
認証情報なし
    ↓
匿名ユーザーとしてAPIへアクセス
```

できる状態が残っています。

そのため、

```text
未認証アクセス
    ↓
禁止

認証済みアクセス
    ↓
Authorizationで権限を確認
```

という状態へ変更する必要があります。

また、認証と認可は別の役割を持っています。

```text
Authentication
    ↓
誰なのか

Authorization
    ↓
何をしてよいのか
```

したがって、

```text
匿名アクセスを禁止した
    ↓
API Security完了
```

ではありません。

認証されたユーザーやNodeについても、

```text
認証
    ↓
Authorization
    ↓
許可された操作だけ実行
```

という制御が必要です。

さらに、Nodeについては通常のAuthorizationだけではなく、Nodeから送信されるAPI要求に対して、  
許可された範囲を超える変更を防ぐためのAdmission制御も考える必要があります。

```text
NodeからAPI Request
        ↓
認証
        ↓
Nodeとして認可
        ↓
Nodeに許可された変更か
        ↓
Admissionで確認
```

という構造です。

今回もう一つ重要なのが、API Serverを安全な状態へ変更した後のkubectlです。  
現在のkubectl設定が、

```text
未認証アクセスが許可されている
        ↓
APIを操作できる
```

という状態に依存している場合、匿名アクセスを禁止すると利用できなくなります。  
これは設定失敗ではありません。

```text
匿名アクセス禁止
        ↓
未認証のkubectl
        ↓
アクセスできなくなる
```

のは、今回期待している動作です。

その後の管理操作では、

```text
/etc/kubernetes/admin.conf
        ↓
正規の管理者認証情報
        ↓
認証済みユーザー
        ↓
API操作
```

という経路を使用してください。

また、API Server側で匿名アクセスを禁止しても、匿名アクセスのために作成された不要な権限設定を残す必要はありません。

```text
匿名アクセス禁止
        +
匿名ユーザー用の不要な権限を削除
```

という形で、認証設定と権限設定の両方を整理します。

この問題では、**「kube-apiserverのオプションを覚える」のではなく、  
「Kubernetes APIの入口で誰を認証し、その主体にどこまで操作を許可し、要求をどの段階で制御するのか」**  
という観点から設定を判断してください。
