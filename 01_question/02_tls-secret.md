# 問題02：TLSを使用するWebアプリケーションを正常化する

## 対象領域

System Hardening / Minimize Microservice Vulnerabilities

## 教材種別

Drill（試験対策問題）

目安時間：5〜10分

# 背景

社内向けWebアプリケーションをKubernetesクラスターへデプロイしました。  
このWebアプリケーションは、クライアントとの通信をTLSで保護するように設計されています。

Deploymentのデプロイは完了していますが、Podが正常に起動していません。  
アプリケーションで使用するサーバー証明書と秘密鍵は、  
Control Plane上に事前に配置されています。

現在の状態を確認し、提供されている証明書と秘密鍵を使用して、  
Webアプリケーションを正常に起動させてください。

# 初期観測情報

対象Deploymentを確認すると、次の状態になっています。

```text
NAME            READY   UP-TO-DATE   AVAILABLE
clever-cactus   0/1     1            0
```

Podのイベントには、次の内容が記録されています。

```text
MountVolume.SetUp failed for volume "tls-certificate" :
secret "clever-cactus" not found

MountVolume.SetUp failed for volume "tls-key" :
secret "clever-cactus" not found
```

Deploymentはすでに、TLS通信に必要な情報をKubernetesから取得するよう構成されています。

# 要件

以下の要件をすべて満たしてください。

1. 提供されているサーバー証明書と秘密鍵を使用すること。
2. 証明書と秘密鍵を、TLS用途としてKubernetesから利用できる状態にすること。
3. アプリケーションが既存設定のまま証明書と秘密鍵を利用できること。
4. 設定後、対象DeploymentのPodが正常に起動すること。

# 設定情報

| 項目              | 設定内容                          |
| --------------- | ----------------------------- |
| Namespace       | `clever-cactus`               |
| Deployment      | `clever-cactus`               |
| アプリケーションが参照する名前 | `clever-cactus`               |
| 証明書             | `~/ca-cert/web.k8s.local.crt` |
| 秘密鍵             | `~/ca-cert/web.k8s.local.key` |

# 制約

* 既存のDeploymentを変更してはいけません。
* Deploymentを削除・再作成してはいけません。
* 提供されている証明書および秘密鍵を使用してください。
* 証明書や秘密鍵の内容そのものを変更してはいけません。
* 既存Deploymentが期待している名前を変更してはいけません。

# 完了条件

以下をすべて満たした場合、作業完了とします。

* 証明書と秘密鍵がKubernetes上でTLS用途として利用できる状態になっている。
* 対象Namespaceから必要な情報を参照できる。
* 既存Deploymentの設定を変更していない。
* 対象Podが正常に起動している。
* Deploymentが利用可能な状態になっている。

# 作業後の確認項目

設定後、次の内容を確認してください。

* 作成したオブジェクトが対象Namespaceに存在すること。
* TLS用途として適切な形式になっていること。
* 証明書と秘密鍵の両方が登録されていること。
* Podで証明書と秘密鍵を利用できていること。
* `FailedMount`が解消していること。
* DeploymentのPodが`Running`になっていること。

# 注意事項

今回のDeploymentには、TLSを使用するための設定がすでに行われています。

そのため、

**「Podが起動しないからDeploymentを修正する」**

とすぐに判断するのではなく、まずPodの状態やイベントから原因を確認してください。

```text
Podが起動しない
    ↓
現在の状態を確認
    ↓
何が不足しているのか
    ↓
アプリケーションが何を参照しているのか
    ↓
不足しているものを適切な形式で提供する
```

また、証明書と秘密鍵は異なる役割を持っています。  
提供されたファイルを単にKubernetesへ保存するのではなく、

**「TLS通信で使用する証明書と秘密鍵を、Kubernetesではどのように扱うのか」**  
を考えて作業してください。
