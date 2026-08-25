# 問題06：Kubernetes APIの操作を監査できる状態にする

## 対象領域

Cluster Setup / System Hardening

## 教材種別

Drill（試験対策問題）

目安時間：15〜20分

# 背景

`kubeadm`で構築されたKubernetesクラスターについて、  
セキュリティインシデント発生時にAPI操作を追跡できるよう、監査機能を有効化することになりました。

現在のクラスターでは、監査ポリシーを配置するためのファイルと、監査ログを保存するための領域は準備されていますが、  
要求された監査内容が設定されておらず、API Serverから監査ログを継続的に記録できる状態になっていません。

セキュリティチームから提示された監査要件に基づいて、

* どのAPI操作を記録するのか
* どこまで詳細に記録するのか
* 監査ログをどこへ保存するのか
* どの程度保持するのか

を設定し、クラスターで監査証跡を取得できる状態にしてください。

# 初期観測情報

監査ポリシーとして使用するファイルは、次の場所にあります。

```text
/etc/kubernetes/logpolicy/sample-audit.yaml
```

API ServerはStatic Podとして稼働しており、マニフェストは次の場所にあります。

```text
/etc/kubernetes/manifests/kube-apiserver.yaml
```

監査ポリシーファイルおよび監査ログ保存先をAPI Serverから  
利用するために必要なVolumeとVolume Mountは、すでにマニフェストへ設定されています。  
既存の監査ポリシーには、一部の操作を監査対象から除外するルールが含まれています。

# 要件

以下の監査要件をすべて満たしてください。

1. `persistentvolumes`に対するAPI操作について、リクエストとレスポンスの内容まで監査証跡として記録すること。
2. `front-apps` Namespaceの`configmaps`に対するAPI操作について、リクエスト内容まで記録すること。
3. すべてのNamespaceにおける`configmaps`および`secrets`への操作について、少なくともリクエストのメタデータを記録すること。
4. 上記以外のAPIリクエストについても、メタデータを監査証跡として記録すること。
5. 作成した監査ポリシーをkube-apiserverが使用すること。
6. 監査ログを指定されたファイルへ保存すること。
7. 監査ログについて、指定された世代数と保持日数を適用すること。
8. 設定変更後もKubernetes API Serverが正常に稼働すること。

# 設定情報

| 項目                            | 設定内容                                            |
| ----------------------------- | ----------------------------------------------- |
| クラスター構築方式                     | `kubeadm`                                       |
| Audit Policy                  | `/etc/kubernetes/logpolicy/sample-audit.yaml`   |
| kube-apiserverマニフェスト          | `/etc/kubernetes/manifests/kube-apiserver.yaml` |
| Audit Log                     | `/var/log/kubernetes/audit-logs.txt`            |
| ログバックアップ数                     | `2`                                             |
| ログ保持日数                        | `10`                                            |
| PVの監査粒度                       | Request + Response                              |
| `front-apps` / ConfigMapの監査粒度 | Request                                         |
| ConfigMap / Secretの監査粒度       | Metadata                                        |
| その他のRequest                   | Metadata                                        |

# 制約

* kube-apiserverを削除・再作成してはいけません。
* 既存のAudit Policyにある監査除外ルールを削除してはいけません。
* 既存のVolumeを削除・変更してはいけません。
* 既存のVolume Mountを削除・変更してはいけません。
* 監査要件を満たすために必要な範囲のみ設定を変更してください。
* 監査ログの出力先は変更してはいけません。
* 必要以上に詳細な監査レベルへ変更してはいけません。

# 完了条件

以下をすべて満たした場合、作業完了とします。

* `persistentvolumes`の操作について、リクエストとレスポンスを含む監査ログが記録される。
* `front-apps` Namespaceの`configmaps`について、リクエスト内容を含む監査ログが記録される。
* `configmaps`および`secrets`の操作について、メタデータが記録される。
* その他のAPIリクエストについてもメタデータが記録される。
* kube-apiserverが指定されたAudit Policyを使用している。
* `/var/log/kubernetes/audit-logs.txt`へ監査ログが出力される。
* 監査ログのバックアップ数が`2`に設定されている。
* 監査ログの保持日数が`10`日に設定されている。
* kube-apiserverが正常に稼働している。
* `kubectl`を使用してクラスターへ正常にアクセスできる。

# 作業後の確認項目

設定後、次の内容を確認してください。

* Audit Policyの構文に問題がないこと。
* 要求されたリソースとNamespaceに対して適切な監査レベルが設定されていること。
* kube-apiserverが指定されたAudit Policyを読み込んでいること。
* 監査ログの出力先が正しく設定されていること。
* ログのバックアップ数と保持日数が要求通りになっていること。
* kube-apiserverが正常に稼働していること。
* Kubernetes APIへの操作によって監査ログが実際に出力されること。
* 監査ログから、実行されたAPI操作を追跡できること。

# 注意事項

監査では、**「すべての操作をできるだけ詳しく記録すればよい」**とは限りません。  
監査ログには、操作に関する情報をどこまで記録するかという粒度があります。

今回の要件では、対象によって必要な情報量が異なります。

```text
PersistentVolume
        ↓
リクエスト
+
レスポンスまで必要

front-apps / ConfigMap
        ↓
リクエスト内容まで必要

ConfigMap / Secret
        ↓
メタデータを記録

その他
        ↓
メタデータを記録
```

そのため、

```text
何を監査したいのか
        ↓
どこまでの情報が必要なのか
        ↓
対象リソース・Namespace
        ↓
監査レベル
        ↓
Audit Policy
```

という順番で考えてください。

また、Audit Policyを作成しただけでは監査機能は成立しません。

```text
監査要件
    ↓
Audit Policy
    ↓
kube-apiserverがPolicyを使用
    ↓
Audit Logを出力
    ↓
ログを保持
    ↓
必要なときに追跡できる
```

まで成立している必要があります。

監査ログはセキュリティインシデントが発生した後に、  
**「誰が、いつ、どのAPIに対して、どのような操作を行ったのか」**  
を追跡するための証跡になります。

単にログファイルを作成するのではなく、  
**後から必要な操作を追跡できる監査証跡を設計する**  
という観点から設定してください。
