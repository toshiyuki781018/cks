# 問題09：ServiceAccount Tokenの自動配布を制限する

## 対象領域

Minimize Microservice Vulnerabilities / ServiceAccount Security

## 教材種別

Drill（試験対策問題）

目安時間：10〜15分

# 背景

`monitoring` Namespaceでは、監視用アプリケーションを`stats-monitor` Deploymentとして運用しています。

セキュリティ監査の結果、このWorkloadが使用するServiceAccountの  
API認証情報がPodへ自動的に提供される構成になっていることが確認されました。

Kubernetes APIの認証情報を必要としないPodにも無条件に認証情報が配置される構成は、  
コンテナが侵害された場合に認証情報が取得される範囲を広げる可能性があります。

そのため、ServiceAccountのAPI認証情報をPodへ自動的に提供しない構成へ変更してください。  
ただし、`stats-monitor` Deploymentではアプリケーションの処理上ServiceAccount Tokenを必要とします。

自動的な認証情報の配布は停止したうえで、このWorkloadに必要なTokenだけを明示的に提供してください。

# 初期観測情報

対象となるServiceAccountはすでに存在します。

```text
Namespace      : monitoring
ServiceAccount : stats-monitor-sa
```

対象となるDeploymentもすでに存在します。

```text
Namespace  : monitoring
Deployment : stats-monitor
```

Deploymentのマニフェストは次の場所にあります。

```text
~/stats-monitor/deployment.yaml
```

`stats-monitor` Deploymentは`stats-monitor-sa` ServiceAccountを使用しています。

また、Deploymentには既存のConfigMap Volumeが設定されています。

# 要件

以下のセキュリティ要件をすべて満たしてください。

1. `stats-monitor-sa`を使用するPodへ、ServiceAccountのAPI認証情報が自動的に提供されないようにすること。
2. `stats-monitor` Deployment自身についても、ServiceAccountのAPI認証情報を自動的にマウントしない構成にすること。
3. `stats-monitor` Deploymentには、必要なServiceAccount Tokenを明示的に提供すること。
4. `token`という名前のVolumeを使用すること。
5. ServiceAccount TokenをVolumeへ投影して利用すること。
6. Tokenはコンテナ内の次のパスから参照できること。

```text
/var/run/secrets/kubernetes.io/serviceaccount/token
```

7. Tokenを格納するVolumeはコンテナから読み取り専用として利用すること。
8. 既存のServiceAccount、Deployment、ConfigMap VolumeなどのWorkload構成を維持すること。

# 設定情報

| 項目               | 設定内容                                                  |
| ---------------- | ----------------------------------------------------- |
| Namespace        | `monitoring`                                          |
| ServiceAccount   | `stats-monitor-sa`                                    |
| Deployment       | `stats-monitor`                                       |
| Deploymentマニフェスト | `~/stats-monitor/deployment.yaml`                     |
| Token用Volume名    | `token`                                               |
| Tokenファイル名       | `token`                                               |
| Tokenの参照先        | `/var/run/secrets/kubernetes.io/serviceaccount/token` |
| Volume Mount     | Read Only                                             |

# 制約

* `stats-monitor-sa`を削除・再作成してはいけません。
* `stats-monitor` Deploymentを削除してはいけません。
* Deploymentが使用するServiceAccountを変更してはいけません。
* 既存のコンテナイメージを変更してはいけません。
* 既存のConfigMap VolumeおよびVolume Mountを削除・変更してはいけません。
* ServiceAccount Tokenの自動マウントを利用して要件を満たしてはいけません。
* TokenをSecretとして新規作成してはいけません。
* 必要なServiceAccount Tokenだけを明示的にVolumeとして提供してください。

# 完了条件

以下をすべて満たした場合、作業完了とします。

* `stats-monitor-sa`でAPI認証情報の自動マウントが無効になっている。
* `stats-monitor` DeploymentでもAPI認証情報の自動マウントが無効になっている。
* Deploymentが引き続き`stats-monitor-sa`を使用している。
* `token`という名前のVolumeが存在する。
* `token` VolumeからServiceAccount Tokenが提供される構成になっている。
* Tokenが次のパスから参照できる。

```text
/var/run/secrets/kubernetes.io/serviceaccount/token
```

* Token用Volumeが読み取り専用でマウントされている。
* Deployment更新後のPodが正常に起動している。
* 既存のConfigMap VolumeおよびVolume Mountが維持されている。

# 作業後の確認項目

設定後、次の内容を確認してください。

* `stats-monitor-sa`のAPI認証情報自動マウント設定が無効になっていること。
* DeploymentのPod TemplateでもAPI認証情報の自動マウントが無効になっていること。
* Deploymentが`stats-monitor-sa`を使用していること。
* ServiceAccount Tokenを提供するVolumeが存在すること。
* TokenがVolumeへ投影される構成になっていること。
* Token用Volumeが指定されたパスへ読み取り専用でマウントされていること。
* Deployment更新後のPodが`Running`になっていること。
* 既存のWorkload構成が維持されていること。

# 注意事項

今回の目的は、

**「ServiceAccount Tokenを一切使用しない」**

ことではありません。

問題となっているのは、

```text
ServiceAccountを使用する
        ↓
API認証情報が自動的にPodへ提供される
        ↓
そのPodがTokenを必要としているかに関係なく
認証情報が存在する
```

という状態です。

コンテナが侵害された場合、Pod内部に存在する認証情報も攻撃者から利用される可能性があります。

そのため、

```text
認証情報を自動的に配布する
        ↓
停止

必要なWorkload
        ↓
必要なTokenだけを明示的に提供
```

という構成へ変更します。

今回の`stats-monitor`はServiceAccount Tokenを必要としているため、  
単純に自動マウントを無効化するだけでは要件を満たしません。

```text
自動マウント
    ↓
無効化

しかし
    ↓
Tokenは必要

そのため
    ↓
ServiceAccount Token
    ↓
Volumeへ投影
    ↓
必要な場所へMount
```

という構造を考えてください。

また、

```text
ServiceAccount
    ↓
認証情報を自動配布するか

Deployment / Pod
    ↓
このWorkloadで認証情報を
どのように利用するか
```

という設定場所の違いも意識してください。

この問題では、**「認証情報を持たせるか、持たせないか」だけではなく、  
「必要な認証情報を、必要なWorkloadへ、どのような方法で提供するか」**  
という観点から設定を判断してください。
