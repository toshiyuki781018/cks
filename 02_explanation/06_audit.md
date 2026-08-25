# 問題06：Kubernetes APIの操作を監査できる状態にする ― 解説・模範解答

## この問題では何を考えればよかったのか

この問題では、Kubernetes APIに対して行われた操作を、後から追跡できる状態にします。

重要なのは、

```text
Audit Policy
```

を書くこと自体ではありません。

まず、

```text
どの操作を記録したいのか
        ↓
どこまで詳しく記録する必要があるのか
        ↓
Audit Policy
        ↓
kube-apiserverへ適用
        ↓
ログとして保存
        ↓
後から追跡
```

という順番で考えます。

監査は、攻撃や誤操作そのものを直接防止する機能ではありません。  
**「誰が、いつ、何に対して、どのようなAPI操作を行ったのか」を証跡として残す**  
ことが目的です。

---

# 問題文を整理する

今回要求されている監査内容は、対象によって異なります。

| 対象                           | 必要な監査情報            |
| ---------------------------- | ------------------ |
| `persistentvolumes`          | Request + Response |
| `front-apps`の`configmaps`    | Request            |
| すべての`configmaps` / `secrets` | Metadata           |
| その他のAPI Request              | Metadata           |

つまり、

```text
全部同じ詳細度で記録する
```

のではありません。

対象ごとに、

```text
何を追跡したいのか
        ↓
必要な情報量
        ↓
Audit Level
```

を判断する必要があります。

---

# Audit Levelを理解する

Kubernetes Auditでは、記録する情報量をLevelによって指定します。

今回重要になるのは、

```text
Metadata
Request
RequestResponse
```

です。

---

## Metadata

```yaml
level: Metadata
```

リクエストに関するメタデータを記録します。

今回では、

* `configmaps`
* `secrets`
* その他のAPI Request

に使用します。

考え方としては、

```text
API操作が行われた
        ↓
誰が
いつ
どのリソースへ
どのようなRequestを行ったか
        ↓
追跡する
```

ための基本的な監査情報です。

---

## Request

```yaml
level: Request
```

Metadataに加えて、リクエスト内容を記録します。

今回では、

```text
Namespace : front-apps
Resource  : configmaps
```

が対象です。

つまり、

```text
誰がConfigMapを操作したか
        +
どのようなRequestを送ったか
```

まで追跡したいという要求です。

---

## RequestResponse

```yaml
level: RequestResponse
```

Requestの情報に加えて、レスポンス内容まで記録します。

今回では、

```text
persistentvolumes
```

が対象です。

したがって、

```text
Request
    ↓
API Server
    ↓
Response
```

まで監査証跡として残します。

---

# 監査レベルを整理する

今回の要求は次のように変換できます。

```text
PersistentVolume
    ↓
RequestResponse

front-apps / ConfigMap
    ↓
Request

ConfigMap / Secret
    ↓
Metadata

その他
    ↓
Metadata
```

ここまで整理できれば、Audit Policyへ変換できます。

---

# Audit Policyではルールの順番にも注意する

Audit Policyでは、Requestに対して一致したルールが適用されます。  
そのため、**より具体的な条件を先に記述し、最後に一般的なルールを配置する**ことが重要です。

例えば、

```text
front-apps / ConfigMap
        ↓
Request

すべてのConfigMap
        ↓
Metadata
```

という要求があります。

先にすべてのConfigMapを対象とする一般的なルールへ一致してしまうと、  
`front-apps`に対して要求されている詳細な記録へ到達できません。

そのため、

```text
具体的なルール
    ↓
一般的なルール
    ↓
最後にCatch-all
```

という順番を意識します。

---

# 模範解答

## 1. Audit Policyを編集する

対象ファイルを確認します。

```bash
vim /etc/kubernetes/logpolicy/sample-audit.yaml
```

既存の監査除外ルールを維持し、その後に必要なルールを設定します。

例：

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy

omitStages:
  - "RequestReceived"

rules:

  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
    resources:
    - group: ""
      resources: ["endpoints", "services"]

  - level: None
    userGroups: ["system:authenticated"]
    nonResourceURLs:
    - "/api*"
    - "/version"

  - level: RequestResponse
    resources:
    - group: ""
      resources: ["persistentvolumes"]

  - level: Request
    namespaces: ["front-apps"]
    resources:
    - group: ""
      resources: ["configmaps"]

  - level: Metadata
    resources:
    - group: ""
      resources:
      - "configmaps"
      - "secrets"

  - level: Metadata
```

---

# Audit Policyを読む

## PersistentVolume

```yaml
- level: RequestResponse
  resources:
  - group: ""
    resources: ["persistentvolumes"]
```

`persistentvolumes`に対する操作について、RequestとResponseまで記録します。  
`persistentvolumes`はCore API Groupに属するため、

```yaml
group: ""
```

とします。

---

## front-apps NamespaceのConfigMap

```yaml
- level: Request
  namespaces: ["front-apps"]
  resources:
  - group: ""
    resources: ["configmaps"]
```

`front-apps` NamespaceにあるConfigMapを対象とし、Request内容まで記録します。

ここでは、

```text
Resource
+
Namespace
```

の両方を条件にしています。

---

## すべてのConfigMapとSecret

```yaml
- level: Metadata
  resources:
  - group: ""
    resources:
    - "configmaps"
    - "secrets"
```

Namespaceを指定していないため、対象となるNamespaceを限定しません。  
これによってConfigMapとSecretへの操作についてMetadataを記録します。

---

## その他すべてのRequest

```yaml
- level: Metadata
```

これまでのルールに一致しなかったRequestについてMetadataを記録します。

これは、**Catch-allルール**として機能します。  
そのため、具体的なルールより後ろに配置します。

---

# 要件とAudit Policyの対応

| 要件                             | Audit Policy             |
| ------------------------------ | ------------------------ |
| PVのRequestとResponse            | `RequestResponse`        |
| `front-apps`のConfigMapのRequest | `Request` + `namespaces` |
| ConfigMap / SecretのMetadata    | `Metadata`               |
| その他のRequest                    | 最後の`Metadata`            |

この対応を、

```text
要件
 ↓
対象
 ↓
必要な情報量
 ↓
Audit Level
 ↓
Rule
```

として理解してください。

---

# 2. kube-apiserverへAudit Policyを設定する

Audit Policyを作成しただけでは、監査ログは出力されません。

次に、

```text
kube-apiserver
    ↓
このPolicyを使用する
```

という設定が必要です。

対象ファイルを編集します。

```bash
vim /etc/kubernetes/manifests/kube-apiserver.yaml
```

`kube-apiserver`のcommandへ必要な設定を行います。

```yaml
spec:
  containers:
  - command:
    - kube-apiserver

    - --audit-policy-file=/etc/kubernetes/logpolicy/sample-audit.yaml
    - --audit-log-path=/var/log/kubernetes/audit-logs.txt
    - --audit-log-maxbackup=2
    - --audit-log-maxage=10
```

既存のkube-apiserver設定は削除せず、監査に必要な設定を追加・修正します。

---

# kube-apiserverの設定を読む

## audit-policy-file

```text
--audit-policy-file=/etc/kubernetes/logpolicy/sample-audit.yaml
```

kube-apiserverが使用するAudit Policyを指定します。

つまり、

```text
sample-audit.yaml
        ↓
kube-apiserver
        ↓
どのRequestをどのLevelで記録するか判断
```

という関係です。

---

## audit-log-path

```text
--audit-log-path=/var/log/kubernetes/audit-logs.txt
```

監査ログの出力先を指定します。

今回のログは、

```text
/var/log/kubernetes/audit-logs.txt
```

へ保存します。

---

## audit-log-maxbackup

```text
--audit-log-maxbackup=2
```

保持する古い監査ログファイルの数を指定します。

今回の要求は`2`です。

---

## audit-log-maxage

```text
--audit-log-maxage=10
```

古い監査ログを保持する日数を指定します。

今回の要求は、

```text
10日
```

です。

---

# VolumeとVolume Mountを確認する

今回の問題では、Audit PolicyとAudit Logをkube-apiserverから利用するための  
VolumeとVolume Mountは、すでに設定されている前提です。

例えば、次のような構成です。

```yaml
volumeMounts:
- mountPath: /etc/kubernetes/logpolicy/sample-audit.yaml
  name: audit
  readOnly: true

- mountPath: /var/log/kubernetes/
  name: audit-log
  readOnly: false

volumes:
- hostPath:
    path: /etc/kubernetes/logpolicy/sample-audit.yaml
    type: File
  name: audit

- hostPath:
    path: /var/log/kubernetes/
    type: DirectoryOrCreate
  name: audit-log
```

今回の制約では、これらを変更する必要はありません。  
ただし、仕組みとしては理解しておく必要があります。

---

# なぜVolume Mountが必要なのか

kube-apiserverはStatic Podとしてコンテナ内で動作しています。

一方、Audit PolicyはNode上の、

```text
/etc/kubernetes/logpolicy/sample-audit.yaml
```

にあります。

そのため、

```text
Node
 ↓
Audit Policy
 ↓
hostPath
 ↓
kube-apiserver Container
```

としてコンテナから参照できるようにします。

監査ログについても同様です。

```text
kube-apiserver Container
        ↓
Audit Log
        ↓
Volume Mount
        ↓
Node
        ↓
/var/log/kubernetes/
```

という形で保存します。

つまり、**Policyを読み込むためのMountと、ログを書き出すためのMount**が必要になります。

---

# 設定を反映する

`kube-apiserver.yaml`はStatic Pod Manifestです。  
そのため、このファイルを変更するとkubeletが変更を検知し、kube-apiserver Static Podへ反映します。  
まずAPI Serverが正常に復旧することを確認します。

```bash
kubectl get nodes
```

さらにControl Plane Podを確認します。

```bash
kubectl get pods -n kube-system
```

kube-apiserverが正常に稼働していることを確認してください。

---

# 監査ログを確認する

監査ログを確認します。

```bash
tail -f /var/log/kubernetes/audit-logs.txt
```

別のターミナルなどからKubernetes APIを操作すると、Audit Policyに従ってログが出力されます。

例えば、

```bash
kubectl get pods -A
```

などのAPI操作を行い、ログが記録されることを確認できます。

---

# 設定後にAPI Serverが起動しない場合

この問題では、

```text
Audit Policy
+
kube-apiserver Static Pod Manifest
```

というControl Planeの重要な設定を直接編集します。  
そのため、構文ミスがあるとkube-apiserverが正常に起動しなくなる可能性があります。

その場合は、

```text
設定した
 ↓
kubectlが使えない
 ↓
設定を全部消す
```

とするのではなく、変更箇所を確認します。

特に確認するのは、

```text
Audit Policy
    ↓
YAMLのインデント
スペル
Level
rulesの構造

kube-apiserver.yaml
    ↓
オプション名
パス
YAML構文
Volume Mount
```

です。

---

# ケーススタディ

## なぜ全部RequestResponseにしないのか

監査ログは詳しければ詳しいほど良いように見えます。

例えば、

```text
すべて
 ↓
RequestResponse
```

にすれば、多くの情報を取得できます。

しかし、監査ログには大量のAPI操作が記録されます。  
詳細度を上げるほど、

```text
ログ量増加
    ↓
Storage消費増加

記録内容増加
    ↓
ログ自体に機密性の高い情報が含まれる可能性
```

も考える必要があります。

そのため、

```text
何を調査する必要があるのか
        ↓
必要な証跡
        ↓
必要なLevelだけ記録
```

と設計します。

---

# Secretの監査を考える

今回、SecretについてはMetadataレベルで記録する要求になっています。

これは、

```text
Secretに対して操作があった
```

という監査証跡を残しながら、必要以上にRequest内容まで記録しない設計として考えることができます。

セキュリティログは、

```text
機密情報を守るためのログ
```

である一方、

```text
ログそのものが機密情報を持つ
```

可能性もあります。

そのため、監査ログ自体についても、

```text
誰が読めるのか
どこへ保存するのか
どのくらい保持するのか
```

を考える必要があります。

---

# 第4問のFalcoとの違い

第4問ではFalcoを使用しました。

Falcoでは、

```text
コンテナが実際に何をしたか
        ↓
Runtime Event
```

を観測しました。

今回のAuditでは、

```text
Kubernetes APIに対して
誰が何を要求したか
        ↓
API Audit
```

を観測します。

整理すると、

```text
Falco
 ↓
Runtimeの挙動を観測

Kubernetes Audit
 ↓
API操作を観測
```

となります。

どちらもセキュリティ監視ですが、**観測している場所が違います。**

---

# 観測点として考える

今回のAuditは、

```text
ユーザー
 ↓
kubectl / API Client
 ↓
kube-apiserver
 ↓
Kubernetes Resource
```

という通信の、

```text
kube-apiserver
```

を観測点にしています。

API Serverを通過する操作を監査することで、

```text
誰が
 ↓
何に
 ↓
どの操作を行ったか
```

を追跡します。

一方で、Pod内部で直接発生したRuntime挙動については、API Auditだけでは観測できません。

だからこそ、

```text
API Audit
+
Runtime Security
```

のように複数の観測点を持つことが重要になります。

---

# この問題で学んでほしいこと

この問題で覚えてほしいのは、

```yaml
level: Metadata
level: Request
level: RequestResponse
```

という設定値だけではありません。

重要なのは、

```text
後から何を追跡したいのか
        ↓
どの操作を観測するのか
        ↓
どこまで情報が必要なのか
        ↓
Audit Level
        ↓
Audit Policy
        ↓
kube-apiserver
        ↓
ログ保存
        ↓
必要なときに追跡
```

という流れです。

さらに、監査では、

```text
ログを取る
```

だけではなく、

```text
必要な情報を取る
+
必要以上の情報を取らない
+
適切に保存する
```

ことまで考える必要があります。

CKSでは、  
**「Audit Policyを書ける」ことから一段進んで、  
「どの操作について、どの程度の証跡を残せば後から調査できるのかを設計できる」こと**  
をこの問題の到達点とします。
