# 問題02：TLSを使用するWebアプリケーションを正常化する ― 解説・模範解答

## この問題では何を考えればよかったのか

この問題では、TLSを使用するように構成されたWebアプリケーションが正常に起動していません。  
重要なのは、最初からSecretを作成すると判断することではありません。

まず、現在の状態から原因を確認します。

```text
Deployment
    ↓
Podが起動しない
    ↓
Podのイベントを確認
    ↓
Secretが存在しない
    ↓
Deploymentが必要としている情報を確認
    ↓
証明書＋秘密鍵
    ↓
TLS用途としてKubernetesへ登録
```

つまり、  
**現象 → 観測 → 原因 → 必要なセキュリティ資材 → Kubernetesの機能**  
という順番で考えることがポイントです。

---

# 問題文を整理する

今回、すでに次のものは用意されています。

```text
Deployment
証明書
秘密鍵
```

一方、Podのイベントには次のエラーがあります。

```text
MountVolume.SetUp failed for volume "tls-certificate" :
secret "clever-cactus" not found

MountVolume.SetUp failed for volume "tls-key" :
secret "clever-cactus" not found
```

ここから、

```text
証明書がない
```

のではなく、

```text
証明書と秘密鍵は存在する
        ↓
しかしKubernetesから参照できる形になっていない
        ↓
Deploymentが参照しているSecretが存在しない
```

と判断できます。

したがって、Deploymentを変更するのではなく、不足しているSecretを作成する必要があります。

---

# 設計の順番

## 1. Deploymentの状態を確認する

まず、対象Deploymentを確認します。

```bash
kubectl get deployment -n clever-cactus
```

対象Deploymentが存在しているものの、Podが利用可能になっていないことを確認します。

---

## 2. Podが起動できない原因を確認する

Podを確認します。

```bash
kubectl get pods -n clever-cactus
```

対象Podの詳細を確認します。

```bash
kubectl describe pod -n clever-cactus <Pod名>
```

今回の問題では、

```text
secret "clever-cactus" not found
```

というイベントが重要です。

ここから、

**Deploymentが参照しているSecretが存在しない**

ことが分かります。

---

## 3. どのSecretが必要なのか判断する

次に考えるのは、

```text
Secretがない
    ↓
何を格納するSecretなのか
```

です。

今回提供されているファイルは、

```text
web.k8s.local.crt
web.k8s.local.key
```

です。

つまり、

```text
サーバー証明書
+
秘密鍵
    ↓
TLSで使用する情報
```

になります。

KubernetesにはTLS用途のSecretとして、

```text
kubernetes.io/tls
```

というSecret Typeがあります。

したがって今回は、**証明書と秘密鍵からTLS Secretを作成する**と判断します。

---

# Kubernetesではどの設定になるのか

TLS Secretでは、証明書と秘密鍵を次のキーで保持します。

```text
tls.crt
tls.key
```

Secretの構造として表すと、次のようになります。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: clever-cactus
  namespace: clever-cactus
type: kubernetes.io/tls
data:
  tls.crt: <Base64 encoded certificate>
  tls.key: <Base64 encoded private key>
```

ただし、証明書と秘密鍵を手作業でBase64エンコードしてYAMLを作成する必要はありません。

`kubectl`にはTLS Secretを作成するためのコマンドが用意されています。

---

# 模範解答

## 1. TLS Secretを作成する

提供されている証明書と秘密鍵を使用して、`clever-cactus` Secretを作成します。

```bash
kubectl create secret tls clever-cactus \
  -n clever-cactus \
  --cert=~/ca-cert/web.k8s.local.crt \
  --key=~/ca-cert/web.k8s.local.key
```

このコマンドによって、

```text
Secret名
clever-cactus

Namespace
clever-cactus

Type
kubernetes.io/tls
```

のSecretが作成されます。

---

# 作成したSecretを読む

Secretを確認します。

```bash
kubectl get secret clever-cactus -n clever-cactus
```

例えば次のように表示されます。

```text
NAME            TYPE                DATA   AGE
clever-cactus   kubernetes.io/tls   2      10s
```

ここで確認したいのは、

```text
TYPE = kubernetes.io/tls
DATA = 2
```

です。

さらに内容を確認する場合は、

```bash
kubectl describe secret clever-cactus -n clever-cactus
```

を使用できます。

TLS Secretには、

```text
tls.crt
tls.key
```

が格納されます。

---

# 要件と設定の対応

| 要件                | 対応                                   |
| ----------------- | ------------------------------------ |
| 提供された証明書を使用       | `--cert=~/ca-cert/web.k8s.local.crt` |
| 提供された秘密鍵を使用       | `--key=~/ca-cert/web.k8s.local.key`  |
| TLS用途として登録        | `kubectl create secret tls`          |
| Deploymentが参照する名前 | `clever-cactus`                      |
| 対象Namespace       | `clever-cactus`                      |
| Deploymentを変更しない  | 不足しているSecretだけを作成                    |

ここで特に重要なのが、

**Secretの名前とNamespace**

です。

Deploymentが、

```text
clever-cactus
```

というSecretを参照しているのであれば、別の名前でSecretを作成しても既存Deploymentから参照できません。  
また、SecretはNamespaceに属するリソースです。

そのため、

```text
Secretは存在する
        ↓
しかし別Namespaceに存在する
```

という状態でも、対象PodからそのSecretを利用することはできません。

---

# 適用後の確認

Secretを作成したら、まずPodの状態を確認します。

```bash
kubectl get pods -n clever-cactus
```

必要に応じて状態の変化を監視します。

```bash
kubectl get pods -n clever-cactus -w
```

Podが正常に起動すれば、

```text
Pending
   ↓
Running
```

となります。

---

# Podが正常化しない場合

Secretを作成してもPodが正常化しない場合は、再度イベントを確認します。

```bash
kubectl describe pod -n clever-cactus <Pod名>
```

`FailedMount`が継続しているのか、それとも別のエラーへ変化したのかを確認します。

必要に応じてDeploymentのPodを再作成します。

```bash
kubectl rollout restart deployment clever-cactus -n clever-cactus
```

その後、状態を確認します。

```bash
kubectl rollout status deployment clever-cactus -n clever-cactus
```

または、

```bash
kubectl get pods -n clever-cactus -w
```

で確認します。

重要なのは、

```text
Secretを作った
    ↓
とりあえず再起動
```

ではありません。

まずSecret作成後の状態を観測し、それでも正常化しない場合に、必要な対応を判断します。

---

# なぜDeploymentを変更しないのか

今回のDeploymentは、

**TLS Secretを利用するための設定そのものはすでに正しい**

という前提があります。

問題なのは、

```text
Deploymentの設定
```

ではなく、

```text
Deploymentが参照するSecretが存在しない
```

ことです。

したがって、

```text
Podが起動しない
        ↓
Deploymentを修正する
```

ではなく、

```text
Podが起動しない
        ↓
イベントを確認
        ↓
Secretが存在しない
        ↓
不足しているSecretを作成
```

と考えます。

これはKubernetesのトラブルシューティングでも重要な考え方です。

**正常な設定まで変更して問題を広げない**

ことを意識してください。

---

# ケーススタディ

## TLS Secretは何をしているのか

HTTPSなどのTLS通信では、大きく次の情報が必要になります。

```text
証明書
+
秘密鍵
```

証明書は、

```text
このサーバーが誰なのか
```

を示すために使用されます。

秘密鍵は、その証明書と組み合わせてTLS通信を成立させる重要な情報です。  
Kubernetesでは、これらをTLS SecretとしてWorkloadへ渡すことができます。

```text
証明書ファイル
秘密鍵ファイル
      ↓
TLS Secret
      ↓
Volumeなどを通じてPodへ提供
      ↓
Webサーバー
      ↓
TLS通信
```

つまりSecretは、

**TLSそのものを実行する機能ではありません。**

証明書や秘密鍵など、  
**アプリケーションがTLS通信を行うために必要な情報をKubernetes上で扱うための仕組み**です。

---

# Secretを使えば安全なのか

Secretという名前から、

```text
Secretに保存した
    ↓
完全に安全
```

と考えないようにしてください。

Secretは、パスワード、トークン、証明書、秘密鍵などをConfigMapなどの  
一般設定とは分離して扱うためのKubernetesリソースです。

そのため実際の環境では、

```text
誰がSecretを作成できるのか

誰がSecretを取得できるのか

どのPodがSecretを利用できるのか
```

といったアクセス制御も重要になります。

CKSでは、Secretというリソースだけを見るのではなく、

```text
機密情報
    ↓
どこに保存するか
    ↓
誰がアクセスできるか
    ↓
どのWorkloadへ渡すか
```

まで考えることが重要です。

---

# この問題で学んでほしいこと

この問題で覚えてほしいのは、

```bash
kubectl create secret tls
```

というコマンドだけではありません。

重要なのは、

```text
Podが起動しない
    ↓
状態を観測する
    ↓
FailedMount
    ↓
Secretが存在しない
    ↓
何のためのSecretなのか
    ↓
証明書＋秘密鍵
    ↓
TLS Secret
    ↓
既存Deploymentを変更せず不足物を提供
    ↓
再確認
```

という判断の流れです。

また、セキュリティの観点では、

```text
TLSを使う
```

だけではなく、

```text
TLSを成立させるための証明書・秘密鍵を
Workloadへどのように提供するのか
```

も重要になります。

**「コマンドを知っている」から一段進んで、「何が不足していて、どこにセキュリティ上の制御点があるのか」を判断できること**  
をこの問題の到達点とします。
