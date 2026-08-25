# 問題07：Namespace間の通信経路を制限する

## 対象領域

Minimize Microservice Vulnerabilities / Network Security

## 教材種別

Drill（試験対策問題）

目安時間：10〜15分

# 背景

Kubernetesクラスターでは、用途の異なるWorkloadを`prod`と`data`の2つのNamespaceに分離して運用しています。

セキュリティレビューの結果、Namespaceを分離しているものの、Pod間通信について必要な制限が設定されておらず、  
想定していないWorkloadから通信を受け付ける可能性があることが確認されました。

コンテナが侵害された場合に、別のWorkloadへ不必要にアクセスされることを防ぐためPod間の通信経路を制限してください。  
既存のNamespaceおよびPodは変更せず、必要な通信経路だけが利用できる状態を作成してください。

# 初期観測情報

クラスターには、次のNamespaceがあります。

```text
prod
data
```

両Namespaceには、次のラベルが設定されています。

```text
env: prod
```

現在稼働しているNamespaceおよびPodについて、構成そのものを変更することはできません。  
セキュリティ要件を満たすため、新しい通信制御用リソースを作成してください。

# 要件

以下の通信要件をすべて満たしてください。

1. `prod` Namespace内のすべてのPodについて、他のPodから開始される受信通信を許可しないこと。
2. `data` Namespace内のすべてのPodについて、受信通信の許可元を制限すること。
3. `data` Namespaceへの受信通信は、`prod` Namespaceに属するPodからの通信だけを許可すること。
4. `prod` Namespaceの識別には、Namespaceへ設定されているラベルを使用すること。
5. 既存のNamespaceおよびPodを変更せずに通信境界を作成すること。

# 設定情報

| 項目                 | 設定内容              |
| ------------------ | ----------------- |
| 受信通信を拒否するNamespace | `prod`            |
| NetworkPolicy名     | `deny-policy`     |
| 通信先Namespace       | `data`            |
| 許可元Namespace       | `prod`            |
| NetworkPolicy名     | `allow-from-prod` |
| Namespace識別ラベル     | `env: prod`       |
| 制御対象               | Ingress           |

# 制約

* 既存のNamespaceを変更してはいけません。
* 既存のNamespaceを削除してはいけません。
* 既存のPodを変更してはいけません。
* 既存のPodを削除してはいけません。
* NamespaceやPodへ新しいラベルを追加してはいけません。
* 既存の`env: prod`ラベルを利用して通信元Namespaceを識別してください。
* セキュリティ要件を満たすために必要なNetworkPolicyのみを作成してください。

# 完了条件

以下をすべて満たした場合、作業完了とします。

* `prod` Namespaceに`deny-policy`が作成されている。
* `prod` Namespace内のすべてのPodがIngress通信の制御対象になっている。
* `prod` Namespace内のPodへの受信通信が許可されていない。
* `data` Namespaceに`allow-from-prod`が作成されている。
* `data` Namespace内のすべてのPodがIngress通信の制御対象になっている。
* `prod` Namespaceから`data` Namespaceへの通信が許可されている。
* 要件に含まれないNamespaceから`data` Namespaceへの受信通信が許可されていない。
* 既存のNamespaceおよびPodが変更されていない。

# 作業後の確認項目

設定後、次の内容を確認してください。

* `prod` Namespaceに必要なNetworkPolicyが存在すること。
* `data` Namespaceに必要なNetworkPolicyが存在すること。
* 各NetworkPolicyが想定したPodを制御対象としていること。
* `prod` NamespaceへのIngress通信が許可されていないこと。
* `prod` Namespaceから`data` Namespaceへの通信が許可されること。
* その他のNamespaceから`data` Namespaceへの通信が許可されないこと。
* 既存のNamespaceおよびPodが変更されていないこと。

# 注意事項

Namespaceを分離しただけでは、

**Namespace間のPod通信まで自動的に禁止されるとは限りません。**

今回考える必要があるのは、

```text
Namespaceを分離
        ↓
Workloadの管理単位は分かれた
        ↓
通信経路はどうなっているのか
        ↓
必要な通信だけを許可する
```

という通信境界です。

特に`data` Namespaceについては、

```text
dataへの通信をすべて許可
```

でも、

```text
dataへの通信をすべて拒否
```

でも要件を満たしません。

必要なのは、

```text
prod
  │
  │ 許可
  ▼
data

その他のNamespace
  │
  │ 拒否
  ×
data
```

という状態です。

そのため、

```text
どのPodを保護するのか
        ↓
どの方向の通信を制御するのか
        ↓
どこからの通信を許可するのか
        ↓
Namespaceをどう識別するのか
        ↓
NetworkPolicy
```

という順番で考えてください。

また、NetworkPolicyでは、**「拒否ルールと許可ルールを上から順番に評価する」**  
という考え方ではありません。

複数のNetworkPolicyが同じPodへ適用される場合、そのPodに対して許可される通信は、  
適用されるPolicyによって許可された通信の組み合わせとして決まります。

今回も、

```text
通信境界を作る
        ↓
必要な通信経路だけを明示的に許可する
```

という考え方を意識してください。

このような通信制御は、あるPodが侵害された場合に、

```text
Pod侵害
   ↓
クラスター内部を探索
   ↓
別Workloadへ接続
   ↓
さらに侵害範囲を拡大
```

といった横方向への影響拡大を抑えるためにも利用できます。

この問題では、**「NetworkPolicyのYAMLを覚える」のではなく、  
「どのWorkloadからどのWorkloadへの通信を信頼するのかを定義し、必要な通信経路だけを残す」**  
という観点から設定を判断してください。
