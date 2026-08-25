# 問題05：コンテナの実行時変更と権限を制限する ― 解説・模範解答

## この問題では何を考えればよかったのか

この問題では、コンテナの実行時に許可する変更範囲と権限を制限します。  
重要なのは、

```text
SecurityContext
```

を書くこと自体ではありません。

まず、コンテナが実行された後に

```text
何を変更できるのか

どのユーザーとして動作するのか

実行中に権限を上げられるのか
```

を考えます。

今回の要件を整理すると、

```text
実行ユーザーを固定する
        +
Root Filesystemを書き換えさせない
        +
実行中の権限昇格を禁止する
```

という3つの制御が必要です。

---

# 問題文を整理する

今回のDeploymentには、2つのコンテナがあります。

```text
sec-ctx-demo-1
sec-ctx-demo-2
```

両方のコンテナに同じセキュリティ要件を適用する必要があります。

| 要件                   | 対象    |
| -------------------- | ----- |
| UID `3000`で実行        | 両コンテナ |
| Root Filesystemを変更不可 | 両コンテナ |
| 権限昇格禁止               | 両コンテナ |

一方で、アプリケーションには書き込み先としてVolumeが用意されています。

```text
sec-ctx-demo-1
    ↓
/data/demo1

sec-ctx-demo-2
    ↓
/data/demo2
```

そのため、

```text
Root Filesystemを書き込み不可にする
```

ことと、

```text
アプリケーションの書き込みをすべて禁止する
```

ことは同じではありません。

---

# 設計の順番

## 1. 実行ユーザーを固定する

最初の要件は、コンテナ内のプロセスを

```text
UID 3000
```

で実行することです。

Kubernetesでは、コンテナ単位の`securityContext`で実行UIDを指定できます。

```yaml
runAsUser: 3000
```

これによって、対象コンテナのプロセスを`UID 3000`で実行します。

---

## 2. Root Filesystemを読み取り専用にする

次に、コンテナイメージ由来の

```text
Filesystem
```

これを実行中に変更させないという要件があります。

そのため、

```yaml
readOnlyRootFilesystem: true
```

を設定します。  
これによって、コンテナのRoot Filesystemを読み取り専用にします。

---

## 3. 実行中の権限昇格を禁止する

次の要件は、

```text
プロセスが実行開始後に
現在より高い権限を取得できないようにする
```

ことです。

この制御には、

```yaml
allowPrivilegeEscalation: false
```

を使用します。

これによって、コンテナ内のプロセスが権限昇格することを禁止します。

---

# Kubernetesではどの設定になるのか

今回必要になる設定は、各コンテナの`securityContext`です。

```yaml
securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  runAsUser: 3000
```

重要なのは、**2つのコンテナそれぞれへ設定すること**です。  
今回の元マニフェストでは、Pod内に複数のコンテナがあります。

そのため、一方だけ設定しても要件を満たしません。

---

# 模範解答

対象マニフェストを編集します。

```bash
vim ~/sec-ns_deployment.yaml
```

各コンテナに`securityContext`を設定します。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secdep
  namespace: sec-ns
spec:
  replicas: 1
  selector:
    matchLabels:
      app: secdep
  template:
    metadata:
      labels:
        app: secdep
    spec:
      containers:
      - name: sec-ctx-demo-1
        image: busybox:1.28
        imagePullPolicy: IfNotPresent
        command: [ "sh", "-c", "sleep 12h" ]
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsUser: 3000
        volumeMounts:
        - name: sec-ctx-vol-1
          mountPath: /data/demo1

      - name: sec-ctx-demo-2
        image: busybox
        imagePullPolicy: IfNotPresent
        command: [ "sh", "-c", "sleep 12h" ]
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsUser: 3000
        volumeMounts:
        - name: sec-ctx-vol-2
          mountPath: /data/demo2

      volumes:
      - name: sec-ctx-vol-1
        emptyDir: {}

      - name: sec-ctx-vol-2
        emptyDir: {}
```

---

# SecurityContextを読む

## runAsUser

```yaml
runAsUser: 3000
```

コンテナ内のプロセスをUID `3000`で実行します。

この設定によって、

```text
誰としてプロセスを動作させるか
```

をKubernetes側で明示できます。

---

## readOnlyRootFilesystem

```yaml
readOnlyRootFilesystem: true
```

コンテナのRoot Filesystemを読み取り専用にします。

これによって、

```text
コンテナイメージから作られたFilesystem
        ↓
実行中に書き換えない
```

という状態を作ります。

---

## allowPrivilegeEscalation

```yaml
allowPrivilegeEscalation: false
```

コンテナ内のプロセスが、実行中に現在より高い権限を取得することを禁止します。

つまり、

```text
現在の権限
    ↓
より高い権限を取得
```

という動作を制限します。

---

# 要件と設定の対応

| セキュリティ要件             | Kubernetes設定                      |
| -------------------- | --------------------------------- |
| UID `3000`で実行        | `runAsUser: 3000`                 |
| Root Filesystemを変更不可 | `readOnlyRootFilesystem: true`    |
| 権限昇格禁止               | `allowPrivilegeEscalation: false` |

この3つを個別に暗記するのではなく、

```text
誰として動くのか
        ↓
runAsUser

どこを書き換えられるのか
        ↓
readOnlyRootFilesystem

実行中に権限を上げられるのか
        ↓
allowPrivilegeEscalation
```

という対応関係で理解してください。

---

# マニフェストを適用する

編集したマニフェストを適用します。

```bash
kubectl apply -f ~/sec-ns_deployment.yaml
```

Deploymentの状態を確認します。

```bash
kubectl get deployment secdep -n sec-ns
```

Podについても確認します。

```bash
kubectl get pods -n sec-ns
```

Deploymentが正常に更新され、PodがRunningになっていることを確認します。

---

# 作成後に確認すること

## Deploymentの設定

```bash
kubectl get deployment secdep -n sec-ns -o yaml
```

各コンテナに次の設定が存在することを確認します。

```yaml
securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  runAsUser: 3000
```

---

# 実行UIDを確認する

Pod名を確認します。

```bash
kubectl get pods -n sec-ns
```

コンテナ内で実行ユーザーを確認できます。

```bash
kubectl exec -n sec-ns <Pod名> \
  -c sec-ctx-demo-1 -- id
```

同様に2つ目のコンテナも確認します。

```bash
kubectl exec -n sec-ns <Pod名> \
  -c sec-ctx-demo-2 -- id
```

UID `3000`で動作していることを確認します。

---

# Root Filesystemの読み取り専用を確認する

コンテナのRoot Filesystemへファイル作成を試します。

```bash
kubectl exec -n sec-ns <Pod名> \
  -c sec-ctx-demo-1 -- touch /testfile
```

Root Filesystemが読み取り専用であれば、書き込みに失敗します。

ここで、

```text
書き込みに失敗した
    ↓
コンテナがおかしい
```

と判断してはいけません。

今回のセキュリティ要件では、Root Filesystemへの書き込みを禁止しているため、これは期待される動作です。

---

# Volumeには書き込める

一方、今回のDeploymentにはVolumeがマウントされています。

```text
/data/demo1
/data/demo2
```

例えば、

```bash
kubectl exec -n sec-ns <Pod名> \
  -c sec-ctx-demo-1 -- touch /data/demo1/testfile
```

のように、Volume Mountされた領域への書き込みは可能です。

つまり、

```text
/
    ↓
Read Only

/data/demo1
    ↓
Volume
    ↓
書き込み可能
```

という構造になります。

---

# なぜVolumeには書き込めるのか

`readOnlyRootFilesystem: true`が制御するのは、

**コンテナのRoot Filesystem**

です。

Volumeは別のFilesystemとしてコンテナへマウントされます。

そのため、

```text
Container Image
        ↓
Root Filesystem
        ↓
Read Only

Volume
        ↓
別のStorage
        ↓
必要な場所だけWrite可能
```

という構成を作ることができます。

これは実務上非常に重要です。

アプリケーションによっては、

```text
ログ
一時ファイル
キャッシュ
アプリケーションデータ
```

などを書き込む必要があります。

その場合でもRoot Filesystem全体を書き込み可能にするのではなく、  
**書き込みが必要な場所だけVolumeとして切り出す**ことができます。

---

# コンテナの不変性として考える

今回の問題タイトルにある「不変性」は、

```text
一度起動したコンテナ内部を
自由に変更し続ける
```

運用を避ける考え方です。

例えば、

```text
コンテナ起動
    ↓
Root Filesystemへ設定を書き換える
    ↓
パッケージを追加する
    ↓
ファイルを変更する
    ↓
起動時とは違う状態になる
```

という運用では、コンテナごとに状態が変わってしまいます。

すると、

```text
同じImageなのに状態が違う

再起動すると変更が消える

障害時に状態を再現しにくい

何が変更されたか追跡しにくい
```

といった問題につながります。

そこで、

```text
Image
 ↓
変更しない

書き込みデータ
 ↓
Volume
```

と役割を分離します。

---

# ケーススタディ

## 実行中のコンテナを直接修正しない

例えば本番環境で問題が発生したとします。

```text
設定ファイルが間違っている
        ↓
Podへexec
        ↓
設定ファイルを直接編集
        ↓
一時的に直る
```

これはその場では動く可能性があります。

しかし、

```text
Pod再作成
    ↓
元のImageから起動
    ↓
修正内容がなくなる
```

可能性があります。

コンテナ運用では、

```text
修正
 ↓
ImageやManifestへ反映
 ↓
再デプロイ
```

という形にすることで、状態を再現しやすくします。

---

# 第3問との違い

第3問でもSecurityContextを扱いましたが、目的が少し異なります。

第3問では、

```text
コンテナ侵害
    ↓
侵害後に使える権限を小さくする
```

という**最小権限・多層防御**が中心でした。

今回の第5問では、

```text
コンテナ実行
    ↓
実行中の変更可能範囲を小さくする
    ↓
予測可能な状態を維持する
```

という**不変性**が中心です。

同じSecurityContextでも、

```text
どの脅威や運用上の問題を
解決するために使っているのか
```

によって意味が変わります。

---

# この問題で学んでほしいこと

この問題で覚えてほしいのは、

```yaml
runAsUser: 3000
readOnlyRootFilesystem: true
allowPrivilegeEscalation: false
```

という3行だけではありません。

重要なのは、

```text
誰として動かすのか
        ↓
実行ユーザー

実行中に何を変更できるのか
        ↓
Filesystem

実行後に権限を上げられるのか
        ↓
Privilege Escalation
```

という考え方です。

さらに、

```text
Root Filesystem
    ↓
変更させない

書き込みが必要なデータ
    ↓
Volume
```

と分離することで、**セキュリティを強化しながら、アプリケーションに必要な書き込みは維持できます。**

CKSでは、  
**「SecurityContextの設定項目を知っている」ことから一段進んで、  
「コンテナに何を変更させ、何を変更させないのかを設計できる」こと**  
を、この問題の到達点とします。
