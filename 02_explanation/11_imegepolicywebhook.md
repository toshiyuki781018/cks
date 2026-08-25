# 問題11：安全性を確認できないコンテナイメージのDeployを拒否する ― 解説・模範解答

## この問題では何を考えればよかったのか

この問題では、コンテナイメージがクラスターへ受け入れられる前に、  
外部のイメージ検査サービスによる判定を行います。

重要なのは、

```text
ImagePolicyWebhookを有効化する
```

こと自体ではありません。

まず、

```text
Workloadの作成要求
        ↓
コンテナイメージを使用する
        ↓
そのImageをクラスターへ入れてよいか
        ↓
Admission段階で検査
        ↓
外部Webhookへ問い合わせ
        ↓
許可 / 拒否
```

という制御を考えます。

さらに今回の要件では、

```text
Webhookから応答がない
        ↓
安全性を確認できない
```

場合にも、Imageを許可してはいけません。

つまり、**「安全であることを確認できたImageだけをクラスターへ受け入れる」**  
ことが、この問題の中心です。

---

# 問題文を整理する

今回関係する設定は3つあります。

```text
/etc/kubernetes/epconfig/image-policy-config.yaml

/etc/kubernetes/epconfig/kube-config.yaml

/etc/kubernetes/manifests/kube-apiserver.yaml
```

それぞれ役割が異なります。

```text
kube-apiserver.yaml
        ↓
ImagePolicyWebhookをAdmissionで使用する

image-policy-config.yaml
        ↓
ImagePolicyWebhookを
どのような方針で動作させるか

kube-config.yaml
        ↓
どのWebhook Backendへ接続するか
```

この役割を整理してから設定します。

---

# 設計の順番

## 1. Webhook Backendへの接続先を設定する

まず、Imageの安全性を判定するサービスは次のEndpointで動作しています。

```text
https://image-bouncer-webhook.default.svc:1323/image_policy
```

kube-apiserverからこのBackendへ接続できるように、Webhook用のkubeconfigを設定します。

---

## 2. Backend障害時の動作を設定する

次に考えるのが、

```text
Webhook Backendへ問い合わせ
        ↓
応答がない
```

場合です。

今回の要件では、

```text
安全性を確認できない
        ↓
Imageを拒否
```

とします。

したがって、ImagePolicyWebhookの動作設定を、Backend障害時にImageを許可しない構成へ変更します。

---

## 3. kube-apiserverでImagePolicyWebhookを有効化する

Webhook BackendとImagePolicyWebhookの動作方針を設定しても、  
kube-apiserverがImagePolicyWebhookを使用しなければAdmission Controlは機能しません。

そのため、

```text
kube-apiserver
        ↓
ImagePolicyWebhook
        ↓
AdmissionConfiguration
```

を関連付けます。

---

## 4. 拒否対象Imageで動作確認する

最後に、

```text
~/web1.yaml
```

を使用します。

このマニフェストにはWebhookによって拒否されるImageが指定されています。

したがって、

```text
kubectl apply
        ↓
Admission
        ↓
ImagePolicyWebhook
        ↓
Webhook Backend
        ↓
拒否
        ↓
Resourceは作成されない
```

となれば、設定が機能していると判断できます。

---

# 作業前に設定をバックアップする

今回変更するファイルには、kube-apiserverのStatic Podマニフェストが含まれます。  
設定を誤るとkube-apiserverが起動できなくなる可能性があります。

元問題でも、編集前に設定ファイルをコピーする手順が示されています。  
まずrootユーザーへ変更します。

```bash
sudo -i
```

Admission Control関連のディレクトリへ移動します。

```bash
cd /etc/kubernetes/epconfig
```

現在のファイルを確認します。

```bash
ls
```

設定変更前にバックアップを作成します。

```bash
cp image-policy-config.yaml /opt/
cp kube-config.yaml /opt/
cp /etc/kubernetes/manifests/kube-apiserver.yaml /opt/
```

これによって、設定変更に失敗した場合に元の状態を確認できます。

---

# ImagePolicyWebhookの動作方針を設定する

`image-policy-config.yaml`を編集します。

```bash
vim /etc/kubernetes/epconfig/image-policy-config.yaml
```

元問題で示されている設定は次の形です。

```yaml
imagePolicy:
  kubeConfigFile: /etc/kubernetes/epconfig/kube-config.yaml
  allowTTL: 50
  denyTTL: 50
  retryBackoff: 500
  defaultAllow: false
```

今回特に重要なのは、

```yaml
defaultAllow: false
```

です。

---

# defaultAllowを考える

今回の要件では、

```text
Webhook Backendが利用可能
        ↓
判定結果に従う
```

だけでは不十分です。

Backendが利用できない場合も考える必要があります。

```text
Webhook Backendへ問い合わせ
        ↓
Backend障害
        ↓
安全性を判定できない
```

ここで、

```yaml
defaultAllow: true
```

なら、安全性を判定できなくてもImageを許可する構成になります。

今回要求されているのは逆です。

```text
安全性を確認できない
        ↓
許可しない
```

そのため、

```yaml
defaultAllow: false
```

とします。

---

# Fail OpenとFail Closed

この考え方は、一般的に次のように整理できます。

```text
判定システムが利用できない
        ↓

Fail Open
    ↓
処理を許可する

Fail Closed
    ↓
処理を拒否する
```

今回採用する考え方は、

```text
Webhook障害
        ↓
安全性不明
        ↓
拒否
```

なので、Fail Closed側の設計です。

---

# Webhook Backendへの接続を設定する

次に、

```bash
vim /etc/kubernetes/epconfig/kube-config.yaml
```

を編集します。

元問題で示されている構成は次の通りです。

```yaml
apiVersion: v1
kind: Config

clusters:
- cluster:
    certificate-authority: /etc/kubernetes/epconfig/server.pem
    server: https://image-bouncer-webhook.default.svc:1323/image_policy
  name: bouncer_webhook

contexts:
- context:
    cluster: bouncer_webhook
    user: api-server
  name: bouncer_validator

current-context: bouncer_validator

preferences: {}

users:
- name: api-server
  user:
    client-certificate: /etc/kubernetes/pki/front-proxy-client.crt
    client-key: /etc/kubernetes/pki/front-proxy-client.key
```

ここで重要なのは、

```yaml
server: https://image-bouncer-webhook.default.svc:1323/image_policy
```

です。

これによって、

```text
ImagePolicyWebhook
        ↓
kubeconfig
        ↓
image-bouncer-webhook
```

という接続先を定義します。

---

# kubeconfigの構造を読む

今回のkubeconfigには、

```text
Cluster
Context
User
```

があります。

## Cluster

```yaml
clusters:
- cluster:
    certificate-authority: /etc/kubernetes/epconfig/server.pem
    server: https://image-bouncer-webhook.default.svc:1323/image_policy
  name: bouncer_webhook
```

Webhook Backendの接続先を定義しています。

---

## User

```yaml
users:
- name: api-server
  user:
    client-certificate: /etc/kubernetes/pki/front-proxy-client.crt
    client-key: /etc/kubernetes/pki/front-proxy-client.key
```

Webhookへ接続するときに使用するクライアント証明書と秘密鍵を指定しています。

---

## Context

```yaml
contexts:
- context:
    cluster: bouncer_webhook
    user: api-server
  name: bouncer_validator
```

接続先と認証情報を関連付けています。

つまり、

```text
bouncer_webhook
    +
api-server
    ↓
bouncer_validator
```

という関係です。

---

# kube-apiserverでAdmission Pluginを有効化する

次にStatic Podマニフェストを編集します。

```bash
vim /etc/kubernetes/manifests/kube-apiserver.yaml
```

既存のAdmission Plugin設定へ`ImagePolicyWebhook`を追加します。

元問題では次の形です。

```yaml
- --enable-admission-plugins=NodeRestriction,ImagePolicyWebhook
```

また、Admission Control設定ファイルを参照します。

```yaml
- --admission-control-config-file=/etc/kubernetes/epconfig/image-policy-config.yaml
```

元問題では、この`--admission-control-config-file`はすでに存在しているため、必要な値を確認して利用します。

---

# 3つの設定の関係

ここまでの設定を整理すると、

```text
kube-apiserver
        │
        │ ImagePolicyWebhookを有効化
        ▼
image-policy-config.yaml
        │
        │ Webhookの動作方針
        │ Backend障害時は拒否
        ▼
kube-config.yaml
        │
        │ 接続先
        ▼
image-bouncer-webhook
```

となります。

つまり、

| 設定                         | 役割                           |
| -------------------------- | ---------------------------- |
| `kube-apiserver.yaml`      | Image検査をAdmission処理へ組み込む     |
| `image-policy-config.yaml` | ImagePolicyWebhookの動作方針を定義する |
| `kube-config.yaml`         | Webhook Backendへの接続方法を定義する   |

この責任分解を理解することが重要です。

---

# kube-apiserverへの反映

`/etc/kubernetes/manifests/kube-apiserver.yaml`はStatic Podマニフェストです。

設定変更後、元問題では次の操作を行っています。

```bash
systemctl daemon-reload
systemctl restart kubelet
```

続いて、クラスターの状態を確認します。

```bash
kubectl get pod -A
```

kube-apiserverを含むControl Planeコンポーネントと既存Podが正常に動作していることを確認します。

---

# 動作確認

最後に、拒否対象Imageを使用するテストマニフェストを適用します。

```bash
kubectl apply -f ~/web1.yaml
```

期待する動作は、

```text
kubectl apply
        ↓
kube-apiserver
        ↓
Admission
        ↓
ImagePolicyWebhook
        ↓
image-bouncer-webhook
        ↓
Image拒否
        ↓
Resource作成失敗
```

です。

ここでは、

```text
PodがRunningになる
```

ことを確認するのではありません。

逆に、

**拒否されるべきImageが正しく拒否されたこと**

が成功条件です。

---

# 「作成できない」が正常なテスト

通常のKubernetes演習では、

```text
Apply
 ↓
Pod Running
```

を成功と考えることが多いです。

しかし、セキュリティ制御のテストでは違います。

今回期待しているのは、

```text
危険な要求
        ↓
拒否
```

です。

つまり、

```text
~/web1.yamlがDeployできなかった
```

ことが、ImagePolicyWebhookが正しく機能した証拠になります。

---

# 要件と設定の対応

| セキュリティ要件             | 設定                                |
| -------------------- | --------------------------------- |
| ImageをAdmission段階で検査 | `ImagePolicyWebhook`              |
| Admission設定を利用       | `--admission-control-config-file` |
| Webhook Backendを指定   | `kube-config.yaml`                |
| Endpointを指定          | `server`                          |
| Backend障害時も拒否        | `defaultAllow: false`             |
| 拒否対象Imageを検証         | `~/web1.yaml`                     |

設定値だけではなく、

```text
どのタイミングで制御する？
        ↓
Admission

何を検査する？
        ↓
Container Image

誰が判定する？
        ↓
Webhook Backend

判定できなかったら？
        ↓
拒否
```

という順番で考えてください。

---

# Admission Controlとは

Kubernetes APIへリソース作成要求が送られると、すぐにリソースが保存されるわけではありません。

概念的には、

```text
API Request
    ↓
Authentication
    ↓
Authorization
    ↓
Admission Control
    ↓
Resource保存
```

という処理があります。

今回のImagePolicyWebhookは、このAdmission Controlの段階でImageを検査します。

つまり、

```text
問題のあるImageを起動した後に検出する
```

のではなく、

```text
問題のあるImageを
クラスターへ受け入れる前に止める
```

ための制御です。

---

# なぜAdmissionで止めるのか

仮にImageを実行した後で検査する構成なら、

```text
危険なImage
    ↓
Pod作成
    ↓
Container起動
    ↓
後から問題を検出
```

となります。

Admission段階で検査すれば、

```text
危険なImage
    ↓
Admission
    ↓
拒否
    ↓
Containerは起動しない
```

となります。

つまり、危険なWorkloadが実行環境へ到達する前に制御できます。

---

# ケーススタディ

## 外部のImage Scannerと組み合わせる

実際の環境では、コンテナイメージについて、

```text
脆弱性
署名
出所
利用ポリシー
```

などを検査してからDeployを許可したい場合があります。

その場合、

```text
Developer
    ↓
Deployment作成要求
    ↓
kube-apiserver
    ↓
Admission
    ↓
Image Policy
    ↓
外部判定サービス
```

という構成を取ることができます。

これによって、**「Deployした人が安全だと思ったから許可する」のではなく、  
クラスター側で一貫したポリシーを強制する**ことができます。

---

# Backend障害時の設計

ここは、この問題で特に重要です。  
Webhookを利用するセキュリティ制御では、

```text
Webhookが正常な場合
```

だけを考えてはいけません。

必ず、

```text
Webhookが落ちたらどうする？
```

を考える必要があります。

今回なら、

```text
Scanner正常
    ↓
安全性を判定
    ↓
許可 / 拒否

Scanner障害
    ↓
安全性不明
    ↓
拒否
```

です。

ただし、この構成ではScanner障害によって新しいWorkloadをDeployできなくなる可能性があります。

つまり、

```text
Security
    ↑
Fail Closed

Availability
    ↓
Scanner障害時にDeploy不可
```

というトレードオフがあります。

今回の要件では、安全性を優先してFail Closedを選択しています。

---

# Control Plane設定を変更するときの注意

今回変更する、

```text
/etc/kubernetes/manifests/kube-apiserver.yaml
```

は通常のDeploymentマニフェストではありません。  
Control Planeの重要なStatic Pod設定です。

そのため、

```text
設定ミス
    ↓
kube-apiserver起動失敗
    ↓
Kubernetes API利用不能
    ↓
kubectlも利用できない
```

という可能性があります。

元問題でも、編集前のバックアップを強く意識しています。

したがって、

```text
現在の設定確認
    ↓
Backup
    ↓
必要な箇所だけ変更
    ↓
kube-apiserver確認
    ↓
クラスター確認
```

という手順を取ることが重要です。

---

# この問題で学んでほしいこと

この問題で覚えてほしいのは、

```text
--enable-admission-plugins=ImagePolicyWebhook
```

や、

```yaml
defaultAllow: false
```

という設定値だけではありません。

重要なのは、

```text
Container ImageをDeployしたい
        ↓
クラスターへ入れる前に検査する
        ↓
Admission Control
        ↓
外部サービスへ判定を依頼する
        ↓
安全なら許可
        ↓
危険なら拒否
        ↓
判定できなくても拒否
```

というセキュリティ制御の考え方です。

さらに、

```text
kube-apiserver
        ↓
Admission機能を有効化

AdmissionConfiguration
        ↓
判定方針を定義

kubeconfig
        ↓
判定Backendへの接続を定義
```

という責任分解も理解してください。

CKSでは、**「ImagePolicyWebhookを設定できる」ことから一段進んで、  
「安全性を確認できないコンテナイメージを実行環境へ到達させない  
Admission制御を設計・設定し、実際に拒否されることまで検証できる」こと**  
をこの問題の到達点とします。
