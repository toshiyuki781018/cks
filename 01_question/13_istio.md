# 問題13：Workload間通信に相互認証を強制する

## 対象領域

Minimize Microservice Vulnerabilities / Service Mesh / Istio / mTLS

## 教材種別

Drill（試験対策問題）

目安時間：10〜15分

# 背景

`mtls` Namespaceでは、複数のアプリケーションが相互に通信しています。

これらのWorkload間通信について、通信内容の保護だけでなく、通信する双方のWorkloadが  
正しい相手であることを確認できる構成へ変更することになりました。

クラスターにはIstioが導入されていますが、現在の`mtls` Namespace内のWorkloadが  
すべてIstioの通信制御下にあるかは確認されていません。

また、Namespace内のWorkload間通信について、  
互認証を必須とするセキュリティポリシーも適用されているか確認できていません。

`mtls` Namespace内のWorkloadを確認し、  
Istioを利用してWorkload間通信を保護してください。

# 初期観測情報

対象Namespaceは次のとおりです。

```text
mtls
```

このNamespaceには複数のWorkloadが稼働しています。

クラスターにはIstioが導入されており、  
Workloadの通信をIstioのService Meshによって制御できます。

Istioの通信制御を利用するWorkloadでは、  
アプリケーションコンテナとともにIstioのProxy Containerが動作します。

現在、

* `mtls` Namespace内のすべてのPodがIstioの通信制御下にあるか
* Namespace全体で相互認証が強制されているか

は確認されていません。

# 要件

以下のセキュリティ要件をすべて満たしてください。

1. `mtls` Namespace内で稼働するすべてのWorkloadをIstioの通信制御下に置くこと。
2. `mtls` Namespaceから作成されるPodについて、IstioのProxy Containerが動作する構成にすること。
3. 既存のWorkloadについても、必要に応じてPodを再作成し、Istioの通信制御を反映すること。
4. `mtls` Namespace内のWorkload間通信では、通信する双方が相互に認証された通信を使用すること。
5. 相互認証されていない通信を許可しないこと。
6. 相互認証のポリシーは、特定のWorkloadだけではなく`mtls` Namespace全体へ適用すること。
7. 設定変更後も、対象Namespace内のWorkloadが正常に稼働していること。

# 設定情報

| 項目              | 設定内容               |
| --------------- | ------------------ |
| Namespace       | `mtls`             |
| Service Mesh    | Istio              |
| Proxy Container | `istio-proxy`      |
| 相互認証の適用範囲       | `mtls` Namespace全体 |
| 通信要件            | 相互認証された通信のみ許可      |

# 制約

* `mtls` Namespaceを削除・再作成してはいけません。
* 既存のアプリケーションコンテナを変更してはいけません。
* 既存Workloadのコンテナイメージを変更してはいけません。
* アプリケーション側へTLS証明書を直接組み込んで要件を満たしてはいけません。
* 特定のWorkloadだけに相互認証を適用してはいけません。
* 相互認証されていない通信を許可する構成にしてはいけません。
* IstioのService Mesh機能を利用して要件を満たしてください。

# 完了条件

以下をすべて満たした場合、作業完了とします。

* `mtls` Namespace内のPodがIstioの通信制御下にある。
* 対象PodでIstioのProxy Containerが動作している。
* 既存Workloadから再作成されたPodにもIstioの通信制御が反映されている。
* `mtls` Namespace全体に相互認証を要求するポリシーが存在する。
* 相互認証されていない通信が許可されない構成になっている。
* 対象Namespace内のPodが正常に稼働している。
* 既存アプリケーションのコンテナイメージなどが維持されている。

# 作業後の確認項目

設定後、次の内容を確認してください。

* `mtls` Namespaceの設定がIstioによる通信制御を利用できる状態になっていること。
* `mtls` Namespace内のすべてのPodについて、`istio-proxy`が存在すること。
* 既存Workloadを再作成した場合、新しく作成されたPodにも`istio-proxy`が存在すること。
* `mtls` Namespaceに相互認証を制御するポリシーが存在すること。
* そのポリシーがNamespace全体を対象としていること。
* 相互認証を必須とする設定になっていること。
* 対象Podが正常に`Running`になっていること。

# 注意事項

今回の目的は、**「IstioのSidecarをPodへ追加する」**ことだけではありません。  
保護したいのは、Workload間の通信です。

通常、アプリケーション同士が通信する場合、

```text
Application A
      ↓
   Network
      ↓
Application B
```

という通信が発生します。

この通信について、

```text
通信内容を第三者から読まれないか

通信相手は本当に想定したWorkloadなのか
```

を考える必要があります。

今回利用するIstioでは、

```text
Application
    ↓
Istio Proxy
    ↓
Network
    ↓
Istio Proxy
    ↓
Application
```

という形でWorkload間通信をService Meshの制御下へ置くことができます。

そのため、まず、

```text
Workload
    ↓
Istioの通信制御下にあるか
    ↓
Proxy Containerを確認
```

という観測が必要です。

Proxy Containerが存在しないWorkloadがある場合は、

```text
なぜProxyが存在しないのか
        ↓
Namespaceの設定を確認
        ↓
IstioによるProxy追加を有効化
        ↓
WorkloadのPodを再作成
        ↓
Proxyが追加されたことを確認
```

という流れで考えてください。

---

また、今回必要なのは単なる通信の暗号化だけではありません。

通常のTLSでは、概念的には、

```text
Client
    ↓
ServerのIdentityを確認
    ↓
暗号化通信
```

という構成になります。

一方、相互認証では、

```text
Workload A
    ↓
相手のIdentityを確認
    ↕
互いに認証
    ↕
Workload B
    ↓
相手のIdentityを確認
```

となります。

つまり、

**通信する双方が互いのIdentityを確認する**

ことが重要です。

---

さらに、相互認証が「利用できる」ことと「必須である」ことは同じではありません。

例えば、

```text
mTLS通信
    ↓
許可

非mTLS通信
    ↓
許可
```

という状態では、相互認証を利用できても、相互認証されていない通信が残る可能性があります。

今回要求されているのは、

```text
mTLS通信
    ↓
許可

非mTLS通信
    ↓
拒否
```

という状態です。

したがって、

```text
相互認証を利用できるようにする
        ↓
だけでは不十分

相互認証されていない通信を
許可しない
        ↓
必要
```

と考えてください。

---

また、今回の要件は特定のPodやDeploymentだけを対象としていません。

対象は、

```text
mtls Namespace
        ↓
すべてのWorkload
```

です。

そのため、

```text
個別Workloadに設定
```

するのではなく、

```text
Namespace
    ↓
共通の通信セキュリティポリシー
    ↓
Namespace内のWorkloadへ適用
```

という構成を考える必要があります。

---

この問題では、

**「Istioの設定方法を覚える」のではなく、「Workload間通信をService Meshの制御下へ置き、  
通信する双方のIdentityを確認した通信だけを許可する」** という観点から設定を判断してください。
