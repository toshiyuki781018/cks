# 問題11：安全性を確認できないコンテナイメージのDeployを拒否する

## 対象領域

Supply Chain Security / Admission Control / ImagePolicyWebhook

## 教材種別

Drill（試験対策問題）

目安時間：15〜20分

# 背景

Kubernetesクラスターでは、コンテナイメージをDeployする前に、  
外部のイメージ検査サービスを利用して安全性を確認する仕組みを導入しています。

イメージ検査サービスは、次のHTTPSエンドポイントで稼働しています。

```text
https://image-bouncer-webhook.default.svc:1323/image_policy
```

しかし、Control Plane上に配置されているAdmission Control関連の設定が不完全なため、  
現在はコンテナイメージに対する検査が正しく機能していません。

また、セキュリティ要件として、イメージ検査サービスへ接続できないなどの理由で安全性を確認できなかった場合、  
そのイメージをクラスターへ受け入れてはいけません。

既存の設定ファイルを修正し、コンテナイメージがクラスターへ受け入れられる前に  
外部サービスによる判定が行われるようにしてください。

# 初期観測情報

Admission Control関連の設定ファイルは、Control Plane上の次のディレクトリに配置されています。

```text
/etc/kubernetes/epconfig
```

このディレクトリには、イメージ検査に使用する設定ファイルが存在します。

```text
image-policy-config.yaml
kube-config.yaml
```

kube-apiserverはStatic Podとして構成されており、マニフェストは次の場所にあります。

```text
/etc/kubernetes/manifests/kube-apiserver.yaml
```

イメージ検査サービスは次のEndpointで利用できます。

```text
https://image-bouncer-webhook.default.svc:1323/image_policy
```

Admission Controlの動作確認には、次のテスト用マニフェストを使用します。

```text
~/web1.yaml
```

このマニフェストでは、イメージ検査サービスによって拒否される対象のコンテナイメージが使用されています。

# 要件

以下のセキュリティ要件をすべて満たしてください。

1. kube-apiserverで、コンテナイメージをAdmission段階で検査する仕組みを有効にすること。
2. kube-apiserverが既存のAdmission Control設定ファイルを使用すること。
3. イメージ検査の問い合わせ先として、指定されたHTTPS Endpointを使用すること。
4. イメージ検査サービスがイメージを拒否した場合、そのイメージを使用するWorkloadをクラスターへ受け入れないこと。
5. イメージ検査サービスへ接続できないなど、安全性を確認できない場合もイメージを受け入れないこと。
6. 設定変更後、kube-apiserverが正常に稼働していること。
7. `~/web1.yaml`を使用して、拒否対象のコンテナイメージがAdmission段階で拒否されることを確認すること。

# 設定情報

| 項目                   | 設定内容                                                          |
| -------------------- | ------------------------------------------------------------- |
| Admission設定ディレクトリ    | `/etc/kubernetes/epconfig`                                    |
| Admission設定          | `/etc/kubernetes/epconfig/image-policy-config.yaml`           |
| Webhook接続設定          | `/etc/kubernetes/epconfig/kube-config.yaml`                   |
| kube-apiserverマニフェスト | `/etc/kubernetes/manifests/kube-apiserver.yaml`               |
| Webhook Endpoint     | `https://image-bouncer-webhook.default.svc:1323/image_policy` |
| Webhook Cluster名     | `bouncer_webhook`                                             |
| Context名             | `bouncer_validator`                                           |
| User名                | `api-server`                                                  |
| テストマニフェスト            | `~/web1.yaml`                                                 |
| Backend障害時           | Imageを拒否                                                      |

# 制約

* イメージ検査サービス自体を変更してはいけません。
* Webhook Endpointを変更してはいけません。
* `~/web1.yaml`で使用されているテスト対象イメージを変更してはいけません。
* Admission Controlを無効化してWorkloadを実行してはいけません。
* イメージ検査サービスが利用できない場合に、イメージを許可する構成にしてはいけません。
* 既存のAdmission Control設定ファイルを利用してください。
* kube-apiserverの既存設定について、今回の要件と関係のない項目を変更してはいけません。

# 完了条件

以下をすべて満たした場合、作業完了とします。

* kube-apiserverでイメージ検査用のAdmission Pluginが有効になっている。
* kube-apiserverが指定されたAdmission Control設定を参照している。
* Webhookの接続先が指定されたHTTPS Endpointになっている。
* イメージ検査サービスが拒否したイメージを使用するWorkloadがAdmission段階で拒否される。
* イメージ検査サービスが利用できず安全性を判定できない場合も、イメージが許可されない。
* 設定変更後もkube-apiserverが正常に稼働している。
* クラスター内の既存Podが正常な状態を維持している。
* `~/web1.yaml`をDeployした際、対象リソースがイメージポリシーによって拒否される。

# 作業後の確認項目

設定後、次の内容を確認してください。

* kube-apiserverが正常に起動していること。
* クラスターのPodが正常な状態であること。
* イメージ検査を行うAdmission Pluginが有効になっていること。
* Admission Control設定ファイルがkube-apiserverから参照されていること。
* Webhookの接続先が指定されたEndpointになっていること。
* Backendで安全性を確認できない場合にイメージを許可しない設定になっていること。

最後に、次のテスト用マニフェストを使用してAdmission Controlの動作を確認してください。

```text
~/web1.yaml
```

このマニフェストをクラスターへ適用した際、

```text
Request
   ↓
kube-apiserver
   ↓
Admission Control
   ↓
Image検査
   ↓
拒否
```

となり、対象リソースが作成されないことを確認してください。

# 注意事項

今回の目的は、**「ImagePolicyWebhookというAdmission Pluginを有効にする」** ことだけではありません。  
コンテナイメージがクラスターへ受け入れられる前に、

```text
Workload作成要求
        ↓
使用するImageを確認
        ↓
外部の検査サービスへ問い合わせ
        ↓
安全性を判定
        ↓
許可 / 拒否
```

という制御を行うことが目的です。

そのため、まず、

```text
どのタイミングでImageを検査するのか
        ↓
Admission

どの仕組みで検査するのか
        ↓
外部Webhook

どこへ問い合わせるのか
        ↓
Webhook Endpoint
```

という構造を考えてください。

また、今回特に重要なのが、イメージ検査サービスが利用できない場合の扱いです。

例えば、

```text
ImageをDeploy
        ↓
Webhookへ問い合わせ
        ↓
Webhookから応答がない
```

という状態になったとします。

このとき、

```text
安全性を確認できない
        ↓
とりあえず許可する
```

という構成では、検査サービスの障害中に未検査のイメージがクラスターへ入る可能性があります。

今回要求されているのは、

```text
安全性を確認できない
        ↓
Imageを許可しない
```

という動作です。

つまり、**「安全であることを確認できた場合だけ受け入れる」** という考え方になります。  
さらに今回の構成では、複数の設定ファイルがそれぞれ異なる役割を持っています。

```text
kube-apiserver
        ↓
AdmissionでImage検査を行うことを決める

Admission Control設定
        ↓
Image検査をどの方針で動作させるかを決める

Webhook接続設定
        ↓
どのBackendへ問い合わせるかを決める
```

そのため、**「どのファイルへ何を書くか」ではなく、  
「それぞれの設定が何を制御しているのか」**  
を整理してから設定してください。

また、kube-apiserverはクラスターのControl Planeを構成する重要なコンポーネントです。

設定を誤ると、

```text
kube-apiserver設定変更
        ↓
Static Pod再作成
        ↓
設定エラー
        ↓
kube-apiserver起動失敗
        ↓
kubectlでクラスターを操作できない
```

という状態になる可能性があります。
変更前の設定を確認し、必要な範囲だけを修正してください。  

この問題では、**「ImagePolicyWebhookの設定値を覚える」のではなく、  
「安全性を確認できないコンテナイメージをAdmission段階でクラスターから排除する仕組みを構成する」**  
という観点から設定を判断してください。
