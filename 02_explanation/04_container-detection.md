# 問題04：不審なコンテナの実行を検知し封じ込める ― 解説・模範解答

## この問題では何を考えればよかったのか

この問題では、Runtimeで発生している不審なアクセスをFalcoで検知し、  
その結果を根拠として停止対象を特定します。

重要なのは、

```text
怪しそうなDeploymentを探す
        ↓
停止する
```

ではありません。

今回の正しい流れは、

```text
不審なアクセスがある
        ↓
何を観測するか決める
        ↓
/dev/memへのアクセス
        ↓
Falcoで検知
        ↓
Container IDを取得
        ↓
コンテナからPodを特定
        ↓
PodからDeploymentを特定
        ↓
対象Deploymentを停止
```

です。

つまり、**検知 → 証拠取得 → 特定 → 封じ込め**  
というRuntime Securityの基本的な流れを扱う問題です。

---

# 問題文を整理する

今回わかっていることは次の通りです。

```text
不審な動作が発生している
        ↓
/dev/memへアクセスしている
        ↓
どのPodかは不明
```

また、複数のDeploymentが存在しています。

そのため、

```text
Deployment名
```

だけを見て停止対象を決めることはできません。

まずFalcoを使って、

```text
/dev/memへアクセスした実行主体
```

を観測する必要があります。

---

# なぜ `/dev/mem` が問題なのか

今回の検知対象は、

```text
/dev/mem
```

です。

`/dev/mem`は、システムの物理メモリへアクセスするためのデバイスファイルです。  
そのため、Pod内のプロセスがこのファイルへアクセスしている場合、

```text
コンテナ
   ↓
/dev/mem
   ↓
Nodeのシステムメモリ
```

という通常のアプリケーション処理では想定しにくいアクセスが発生していると考えられます。  
今回の問題では、このアクセスをRuntime上の観測点として使用します。

---

# 設計の順番

## 1. Falcoのルールファイルを確認する

Falcoの設定ディレクトリを確認します。

```bash
sudo -i

cd /etc/falco
ls
```

環境には、例えば次のファイルがあります。

```text
falco_rules.yaml
falco_rules.local.yaml
```

標準ルールファイルを直接変更するのではなく、  
ローカルルールファイルへ独自ルールを作成します。

```text
/etc/falco/falco_rules.local.yaml
```

---

## 2. 何を条件として検知するか考える

今回の観測対象は明確です。

```text
/dev/mem
```

したがって、

```text
ファイルアクセス
        ↓
fd.name
        ↓
/dev/mem
```

という条件で検知します。

元問題では、対象ファイルをFalcoのlistとして定義しています。

```yaml
- list: dev-file
  items: [/dev/mem]
```

これをconditionから参照します。

---

## 3. 検知結果に何を出力するか考える

ここがこの問題で特に重要です。

単に、

```text
/dev/memへのアクセスを検知した
```

と出力するだけでは、その後にどのPodを停止すればよいか判断できません。

今回必要なのは、

```text
検知
 ↓
コンテナ特定
 ↓
Pod特定
```

です。

そのため、Falcoの出力に、

```text
%container.id
```

を含めます。

これによって、Runtimeイベントを発生させたコンテナを後から追跡できます。

---

# 模範解答

## 1. Falcoローカルルールを作成する

`/etc/falco/falco_rules.local.yaml`を編集します。

```bash
vim /etc/falco/falco_rules.local.yaml
```

元問題に沿ったルール例は次の通りです。

```yaml
- list: dev-file
  items: [/dev/mem]

- rule: devmem
  desc: Detect access to /dev/mem
  condition: >
    fd.name in (dev-file)
  output: >
    Shell (container_id=%container.id)
  priority: NOTICE
  tags: [file]
```

---

# Falcoルールを読む

## list

```yaml
- list: dev-file
  items: [/dev/mem]
```

検知対象となるファイルを定義しています。

今回は、

```text
/dev/mem
```

だけが対象です。

---

## rule

```yaml
- rule: devmem
```

作成するFalcoルールの名前です。

ルール名自体は任意です。

---

## condition

```yaml
condition: >
  fd.name in (dev-file)
```

Falcoが検知する条件です。

今回のルールでは、

```text
アクセス対象ファイル
        ↓
/dev/mem
```

であるイベントを検知します。

---

## output

```yaml
output: >
  Shell (container_id=%container.id)
```

検知時に出力する内容です。

今回特に重要なのは、

```text
%container.id
```

です。

この情報を利用して、

```text
Falcoイベント
     ↓
Container ID
     ↓
Pod
```

と追跡します。

---

## priority

```yaml
priority: NOTICE
```

今回の問題で指定されているPriorityです。

---

# ルールを使用してRuntimeイベントを観測する

作成したルールを使用してFalcoを実行します。  
元問題では、30秒間実行しています。

```bash
falco -M 30 \
  -r /etc/falco/falco_rules.local.yaml \
  > dev.log
```

実行後、ログを確認します。

```bash
cat dev.log
```

例えば次のような出力が得られます。

```text
11:12:50.180930443: Notice Shell (container_id=8348dccac054)
11:12:50.184015793: Notice Shell (container_id=8348dccac054)
```

ここから、

```text
8348dccac054
```

というContainer IDを取得できます。

---

# Container IDからPodを特定する

次に、Container Runtime上のコンテナを確認します。

```bash
crictl ps | grep 8348dccac054
```

例えば次のように表示されます。

```text
8348dccac0549   ...   busybox   ...   cpu-65cf4d685c-lvnq
```

ここから、不審なRuntimeイベントを発生させたPodが、

```text
cpu-65cf4d685c-lvnq
```

であることを特定できます。

---

# PodからDeploymentを特定する

Pod名にはReplicaSet由来の文字列が含まれるため、ある程度推測することはできます。  
しかし、問題では観測結果を根拠として停止対象を決めるため、Owner Referenceを確認する方が確実です。

まずPodを確認します。

```bash
kubectl get pod cpu-65cf4d685c-lvnq -o wide
```

管理元を確認する場合は、

```bash
kubectl get pod cpu-65cf4d685c-lvnq \
  -o jsonpath='{.metadata.ownerReferences[0].name}'
```

などを利用できます。

PodがReplicaSetによって管理されている場合、

```text
Pod
 ↓
ReplicaSet
 ↓
Deployment
```

と管理関係をたどります。

例えばReplicaSetが、

```text
cpu-65cf4d685c
```

であれば、Deploymentを確認します。

```bash
kubectl get rs cpu-65cf4d685c \
  -o jsonpath='{.metadata.ownerReferences[0].name}'
```

今回の元問題では、最終的に、

```text
cpu
```

Deploymentが対象となっています。

---

# Deploymentを停止する

対象Deploymentが特定できたら、Replica数を`0`にします。

```bash
kubectl scale deployment cpu --replicas=0
```

この操作によって、

```text
Deployment
    ↓
desired replicas = 0
    ↓
ReplicaSet
    ↓
Pod停止
```

となります。

---

# 要件と操作の対応

| 要件                | 対応                           |
| ----------------- | ---------------------------- |
| `/dev/mem`アクセスを検知 | Falcoの`condition`            |
| 実行主体を識別           | `%container.id`              |
| コンテナを特定           | `crictl ps`                  |
| Podを特定            | Container Runtime情報          |
| Deploymentを特定     | Owner Reference              |
| 不審Workloadを停止     | `kubectl scale --replicas=0` |

この問題では、

```text
Falcoを書く
```

こと自体が目的ではありません。

Falcoは、

```text
Runtimeで起きたことを観測する
```

ために使用しています。

---

# 作業後の確認

## Deploymentの状態

```bash
kubectl get deployment
```

対象Deploymentが例えば次の状態になっていることを確認します。

```text
NAME         READY   UP-TO-DATE   AVAILABLE
amd-gpu      1/1     1            1
cpu          0/0     0            0
nvidia-gpu   1/1     1            1
```

重要なのは、

```text
対象Deploymentだけが0/0
```

になっていることです。

---

## Podの状態

```bash
kubectl get pods
```

不審なPodが停止していることを確認します。

同時に、他のDeploymentのPodが正常に稼働していることも確認します。

---

# なぜPodを直接削除しないのか

Deploymentによって管理されているPodを直接削除すると、

```text
Pod削除
   ↓
ReplicaSet
   ↓
desired replicasを維持
   ↓
新しいPodを作成
```

となります。

つまり、

```bash
kubectl delete pod
```

だけでは、不審なWorkloadが再び起動する可能性があります。

そのため今回は、

```text
Pod
```

ではなく、

```text
Deployment
```

側のReplica数を`0`にします。

これによって、管理Controller自身に、

```text
Podを起動しない状態
```

を要求します。

---

# なぜDeploymentを削除しないのか

今回の目的は、

```text
不審なWorkloadを封じ込める
```

ことです。

Deploymentそのものを削除すると、後続調査に必要な設定情報まで失われる可能性があります。

そのため、

```text
Deploymentを残す
        ↓
Replicaだけ0
        ↓
実行を停止
```

という方法を取ります。

これはインシデント対応における、

**Containment（封じ込め）**

として考えることができます。

---

# ケーススタディ

## Runtime Securityとは何を見るのか

これまでの問題では、

```text
Dockerfile
SecurityContext
Secret
```

など、主に設定された状態を扱ってきました。

Runtime Securityでは視点が変わります。

```text
設定上どうなっているか
```

だけではなく、

```text
実際にコンテナが何をしているのか
```

を観測します。

例えば、

```text
通常の設定
        ↓
アプリケーションに脆弱性
        ↓
攻撃者がコード実行
        ↓
想定外のファイルへアクセス
```

というケースでは、設定ファイルだけを確認しても攻撃を把握できないことがあります。

Falcoは、このようなRuntimeの挙動を観測するために使用できます。

---

# 検知だけではインシデント対応は終わらない

今回の重要な流れは、

```text
Falcoがアラートを出した
```

で終わらないことです。

実際の対応では、

```text
検知
 ↓
何が起きたか
 ↓
誰が起こしたか
 ↓
どのWorkloadか
 ↓
影響範囲はどこか
 ↓
封じ込め
```

と進める必要があります。

今回の問題では、そのために、

```text
%container.id
```

を出力しています。

つまりFalcoルールの`output`は、

**人間が読むメッセージを表示するだけではなく、その後の調査に必要な証拠を残す場所**

でもあります。

---

# 観測点と制御点を分けて考える

この問題は、

```text
観測点
 ↓
/dev/memへのアクセス

制御点
 ↓
DeploymentのReplica数
```

と整理できます。

Falcoで観測する場所と、実際にWorkloadを止める場所は異なります。

```text
Runtimeイベント
        ↓
Falco
        ↓
証拠取得
        ↓
Kubernetes Resourceを特定
        ↓
Deploymentを制御
```

という構造です。

Runtime Securityでは、

**「どこで異常を観測するか」と「どこで止めるか」を分けて考える**

ことが重要です。

---

# この問題で学んでほしいこと

この問題で覚えてほしいのは、

```bash
falco -M 30
```

や、

```yaml
fd.name in (dev-file)
```

といった記述そのものだけではありません。

重要なのは、

```text
不審な動作
    ↓
何を観測するのか
    ↓
Falcoで検知
    ↓
追跡可能な情報を取得
    ↓
Container
    ↓
Pod
    ↓
Deployment
    ↓
封じ込め
```

という流れです。

そして、セキュリティ対応では、

```text
怪しいから止める
```

のではなく、

```text
観測
 ↓
証拠
 ↓
特定
 ↓
制御
```

と進めます。

CKSでは、  
**「Falcoのルールを書ける」ことから一段進んで、「Runtimeで発生した異常を観測し、  
その証拠から影響範囲を特定して封じ込められる」こと**  
をこの問題の到達点とします。
