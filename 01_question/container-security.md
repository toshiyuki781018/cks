# 問題10：セキュリティ基準を満たすようにWorkloadの実行権限を制限する

## 対象領域

Minimize Microservice Vulnerabilities / Pod Security Standards / Container Hardening

## 教材種別

Drill（試験対策問題）

目安時間：10〜15分

# 背景

Kubernetesクラスターでは、すべてのNamespaceにPodのセキュリティ基準が適用されています。

`confidential` Namespaceでは機密性の高いWorkloadを運用しており、  
許可されたセキュリティ基準を満たすPodだけを実行できるようにしています。

しかし、現在配置されている`nginx-unprivileged-deployment`は  
要求されるセキュリティ基準を満たしておらず、Podを正常に実行できません。

既存のDeploymentマニフェストを修正し、アプリケーションの構成を維持したまま、  
要求されるセキュリティ基準を満たす状態にしてください。

# 初期観測情報

対象となるDeploymentは次のとおりです。

```text
Namespace  : confidential
Deployment : nginx-unprivileged-deployment
```

Deploymentのマニフェストファイルは次の場所にあります。

```text
~/nginx-unprivileged.yaml
```

Deploymentでは次のコンテナイメージを使用しています。

```text
nginxinc/nginx-unprivileged
```

コンテナはPort `8080`を使用します。

現在、このDeploymentから作成されるPodは、Namespaceに適用されているセキュリティ基準を満たしていません。

# 要件

以下のセキュリティ要件をすべて満たすように、指定されたDeploymentマニフェストを修正してください。

1. コンテナから親プロセス以上の権限を取得できないようにすること。
2. コンテナに不要なLinux Capabilityを保持させないこと。
3. コンテナをrootユーザーとして実行しないこと。
4. コンテナランタイムが提供する標準的なseccompプロファイルを使用すること。
5. 修正後のDeploymentからPodが正常に作成され、実行できること。
6. 既存のアプリケーション構成は維持すること。

# 設定情報

| 項目             | 設定内容                            |
| -------------- | ------------------------------- |
| Namespace      | `confidential`                  |
| Deployment     | `nginx-unprivileged-deployment` |
| マニフェスト         | `~/nginx-unprivileged.yaml`     |
| Container      | `nginx`                         |
| Image          | `nginxinc/nginx-unprivileged`   |
| Container Port | `8080`                          |
| seccomp        | コンテナランタイムの標準プロファイル              |

# 制約

* Namespaceのセキュリティ設定を変更してはいけません。
* Namespaceを削除・再作成してはいけません。
* Deployment名を変更してはいけません。
* コンテナイメージを変更してはいけません。
* Container Portを変更してはいけません。
* セキュリティ基準を緩和してPodを実行してはいけません。
* 指定されたDeploymentマニフェスト側を修正して要件を満たしてください。

# 完了条件

以下をすべて満たした場合、作業完了とします。

* `nginx-unprivileged-deployment`がNamespaceのセキュリティ基準を満たしている。
* コンテナの特権昇格が許可されていない。
* コンテナが不要なLinux Capabilityを保持していない。
* コンテナがrootユーザーとして実行されない。
* コンテナランタイムの標準的なseccompプロファイルが使用されている。
* Deploymentから作成されたPodが正常に`Running`になる。
* 既存のイメージ、Portなどのアプリケーション構成が維持されている。

# 作業後の確認項目

設定後、次の内容を確認してください。

* 修正したDeploymentマニフェストを正常に適用できること。
* Podの作成時にセキュリティ基準への違反が発生しないこと。
* `confidential` NamespaceのPodが正常に`Running`になっていること。
* Deploymentが引き続き`nginxinc/nginx-unprivileged`を使用していること。
* Container Port `8080`が維持されていること。
* 要求された実行権限制限がコンテナへ適用されていること。

# 注意事項

今回の目的は、Namespace側に設定されているセキュリティ基準を変更することではありません。

考える必要があるのは、

```text
Namespaceにセキュリティ基準がある
        ↓
現在のWorkloadが基準を満たしていない
        ↓
なぜ拒否されているのか確認
        ↓
要求されている実行条件を整理
        ↓
Workload側の実行権限を制限
        ↓
基準を満たした状態でPodを実行
```

という流れです。

例えば、

```text
特権昇格させない
不要なLinux Capabilityを持たせない
rootで実行させない
利用可能なsystem callを制限する
```

という要求は、それぞれコンテナの実行環境に対するセキュリティ要件です。

これらの要件から、

```text
何を禁止したいのか
        ↓
コンテナのどの実行能力を制御するのか
        ↓
Kubernetesではどこへ設定するのか
```

を判断してください。

また、今回の対策はアプリケーションへの侵入そのものを防ぐことだけを目的としているわけではありません。

仮にコンテナが侵害された場合でも、

```text
コンテナ侵害
        ↓
利用できる権限やOS機能を制限
        ↓
侵害後に実行できる操作を減らす
```

という形で影響範囲を小さくすることができます。

この問題では、**「SecurityContextの設定値を覚える」のではなく、  
「要求されるセキュリティ基準を、コンテナの実行制約へ翻訳する」**  
という観点から設定を判断してください。
