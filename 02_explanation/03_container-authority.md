# 問題03：コンテナ実行権限を最小化する ― 解説・模範解答

## この問題では何を考えればよかったのか

この問題では、コンテナが持つ権限を最小化するために、  
2つのレイヤーを確認する必要があります。

```text
コンテナイメージ
    ↓
Dockerfile
    ↓
どのユーザーで実行するのか

Kubernetes
    ↓
Deployment
    ↓
実行時にどの権限を与えるのか
```

重要なのは、**「コンテナをrootで動かさない」だけで終わらない**ことです。

仮にコンテナ内のアプリケーションが侵害された場合でも、

```text
どのユーザーとして動けるのか

どこへ書き込めるのか

どのLinux Capabilityを利用できるのか

特権コンテナとして動作していないか
```

によって、その後の影響範囲が変わります。

そのため、

```text
侵害される可能性
    ↓
侵害されたプロセスが利用できる権限
    ↓
必要以上の権限を与えない
    ↓
Dockerfile + SecurityContext
```

という考え方が、この問題のポイントです。

---

# 問題文を整理する

今回のセキュリティ要件を整理すると、次のようになります。

| 対象               | 要件               | 確認する場所     |
| ---------------- | ---------------- | ---------- |
| 実行ユーザー           | rootで実行しない       | Dockerfile |
| 実行UID            | UID `65535`で実行する | Deployment |
| Root Filesystem  | 書き込み不可           | Deployment |
| 特権モード            | 使用しない            | Deployment |
| Linux Capability | 不要なものを削除         | Deployment |
| 必要なCapability    | Webサーバー用の権限を維持   | Deployment |

さらに今回の問題には、**設定項目を追加・削除してはいけない**  
という制約があります。

したがって、新しい設定を考える問題ではなく、

```text
既存設定を確認
    ↓
セキュリティ上問題のある値を特定
    ↓
要件を満たす値へ変更
```

という問題になります。

---

# 設計の順番

## 1. Dockerfileの実行ユーザーを確認する

まずDockerfileを確認します。

```bash
cat /cks/docker/Dockerfile
```

今回問題となるのは、コンテナのプロセスを特権ユーザーで実行する設定です。

```dockerfile
USER root
```

rootはコンテナ内部で大きな権限を持ちます。  
そのため、アプリケーションがroot権限を必要としないのであれば、

```text
root
 ↓
非特権ユーザー
```

へ変更します。

今回の設定情報では、非特権ユーザーとして`anyone`を使用することになっています。

---

## 2. Kubernetes側の実行ユーザーを確認する

Dockerfileだけではなく、Kubernetes側の設定も確認します。

```bash
cat /cks/docker/deployment.yaml
```

Kubernetesでは、コンテナの実行時セキュリティ設定を`securityContext`で制御できます。  
今回指定されているUIDは、

```text
65535
```

です。

したがって、コンテナをUID `65535`で実行するように既存値を変更します。

---

## 3. Root Filesystemへの書き込みを制限する

次に、

```text
侵害されたプロセスが
コンテナ内部のファイルを書き換えられる状態でよいのか
```

を考えます。

不要な書き込みを防ぐため、コンテナのRoot Filesystemを読み取り専用にします。  
これによって、

```text
コンテナプロセス
    ↓
Root Filesystem
    ↓
読み取り可能
    ↓
書き込み不可
```

という境界を作ります。

---

## 4. 特権コンテナとして実行させない

特権コンテナは通常のコンテナよりも強い権限を持ちます。  
今回のWebアプリケーションには、そのような権限は必要ありません。

そのため、

```text
privileged
    ↓
false
```

とします。

---

## 5. Linux Capabilityを最小化する

Linuxでは、rootが持つ強い権限をCapabilityという単位に分割できます。  
今回の要件では、不要なCapabilityを保持させません。

考え方としては、

```text
現在持っているCapability
        ↓
一度不要なものをすべて削除
        ↓
アプリケーションに必要なものだけ戻す
```

となります。

そのため、

```yaml
capabilities:
  drop:
  - ALL
```

として不要なCapabilityを削除します。  
そのうえで、Webサーバーに必要なCapabilityとして、

```yaml
add:
- NET_BIND_SERVICE
```

を追加します。

つまり、

```text
多くのCapabilityを保持
        ↓
ALLを削除
        ↓
NET_BIND_SERVICEだけを付与
```

という最小権限の構成です。

---

# 模範解答

## 1. Dockerfileを修正する

対象ファイルを編集します。

```bash
vim /cks/docker/Dockerfile
```

実行ユーザーを変更します。

```dockerfile
USER anyone
```

今回の問題では、既存の`USER`ディレクティブの値だけを変更します。  
新しく`USER`を追加するのではありません。

---

## 2. Deploymentを修正する

対象ファイルを編集します。

```bash
vim /cks/docker/deployment.yaml
```

既存の`securityContext`を、要件を満たす値へ変更します。

```yaml
securityContext:
  runAsUser: 65535
  readOnlyRootFilesystem: true
  privileged: false
  capabilities:
    drop:
    - ALL
    add:
    - NET_BIND_SERVICE
```

今回の問題では、これらのフィールドは既に存在している前提です。

したがって、**フィールドを追加・削除するのではなく、既存値を変更してください。**

---

# SecurityContextを読む

## runAsUser

```yaml
runAsUser: 65535
```

コンテナ内のプロセスをUID `65535`で実行します。  
今回指定されている非特権ユーザーで実行するための設定です。

---

## readOnlyRootFilesystem

```yaml
readOnlyRootFilesystem: true
```

コンテナのRoot Filesystemを読み取り専用にします。  
これによって、コンテナ内のプロセスがRoot Filesystemへ自由に書き込むことを防ぎます。

---

## privileged

```yaml
privileged: false
```

コンテナを特権コンテナとして実行しません。  
通常のアプリケーションコンテナでは、必要がない限り特権モードを使用しないことが重要です。

---

## capabilities.drop

```yaml
capabilities:
  drop:
  - ALL
```

コンテナが保持するLinux Capabilityを削除します。  
考え方として重要なのは、

```text
必要かどうか分からない権限を残す
```

のではなく、

```text
一度削除する
    ↓
本当に必要なものだけを付与する
```

ことです。

---

## capabilities.add

```yaml
add:
- NET_BIND_SERVICE
```

削除したCapabilityの中から、アプリケーションに必要なものだけを追加します。  
今回残すのは`NET_BIND_SERVICE`です。

---

# 要件と設定の対応

| セキュリティ要件                | 設定                             |
| ----------------------- | ------------------------------ |
| rootユーザーで実行しない          | Dockerfileの`USER anyone`       |
| 指定されたUIDで実行             | `runAsUser: 65535`             |
| Root Filesystemへの書き込み禁止 | `readOnlyRootFilesystem: true` |
| 特権コンテナを禁止               | `privileged: false`            |
| 不要なCapabilityを削除        | `drop: ALL`                    |
| 必要なCapabilityのみ付与       | `add: NET_BIND_SERVICE`        |

設定を個別に暗記するのではなく、

```text
何を禁止したいのか
        ↓
どの権限を制御するのか
        ↓
Dockerfileなのか
SecurityContextなのか
        ↓
具体的な設定
```

という対応関係で理解してください。

---

# 修正後の確認

## Dockerfile

実行ユーザーを確認します。

```bash
grep "^USER" /cks/docker/Dockerfile
```

期待する状態は、

```text
USER anyone
```

です。

---

## Deployment

SecurityContextを確認します。

```bash
grep -A 10 "securityContext:" /cks/docker/deployment.yaml
```

次の設定になっていることを確認します。

```yaml
securityContext:
  runAsUser: 65535
  readOnlyRootFilesystem: true
  privileged: false
  capabilities:
    drop:
    - ALL
    add:
    - NET_BIND_SERVICE
```

今回の問題ではファイルの編集が目的なので、問題文で適用まで要求されていない場合は、  
指定されたファイルを正しく修正できていることを確認します。

---

# DockerfileとSecurityContextは何が違うのか

ここは、この問題で特に理解してほしいポイントです。

Dockerfileでは、

```dockerfile
USER anyone
```

として、**コンテナイメージ側のデフォルト実行ユーザー**を設定しています。

一方、Deploymentの、

```yaml
securityContext:
  runAsUser: 65535
```

は、

**Kubernetesでコンテナを実行するときのユーザー**  
を指定しています。

つまり、

```text
Dockerfile
    ↓
イメージとしての実行ユーザー

Deployment
    ↓
Kubernetesで実行するときのユーザー
```

という違いがあります。  
セキュリティを考える場合、

```text
イメージは安全か
+
実行環境は安全か
```

の両方を見る必要があります。

---

# ケーススタディ

## コンテナが侵害されたらどうなるのか

コンテナセキュリティでは、

```text
侵害されないようにする
```

ことはもちろん重要です。

しかし、それだけではありません。

```text
侵害されたとしても
どこまで操作できるのか
```

という視点も重要になります。

例えば、

```text
Webアプリケーションに脆弱性
        ↓
攻撃者がコンテナ内でコード実行
        ↓
コンテナがroot
        ↓
Filesystemへ書き込み可能
        ↓
多くのCapabilityを保持
```

という構成では、侵害後に利用できる権限が大きくなります。

そこで、

```text
非root
+
Read Only Root Filesystem
+
非特権コンテナ
+
Capability最小化
```

によって、侵害後に利用できる権限を制限します。

---

# Defense in Depthとして考える

今回の設定は、一つの設定だけですべてを守るものではありません。

```text
Dockerfile
    ↓
非root

Kubernetes
    ↓
UIDを制限

Filesystem
    ↓
書き込み制限

Linux Capability
    ↓
不要な権限を削除

Privileged
    ↓
使用しない
```

複数のレイヤーで制限することで、一つの防御が突破された場合でも影響を小さくします。  
これは、**Defense in Depth（多層防御）**というセキュリティの基本的な考え方です。

---

# この問題で学んでほしいこと

この問題で覚えてほしいのは、

```yaml
runAsUser: 65535
readOnlyRootFilesystem: true
privileged: false
```

という値そのものではありません。

重要なのは、

```text
コンテナが侵害される可能性
        ↓
侵害された場合に何ができるのか
        ↓
実行ユーザー
        ↓
Filesystem
        ↓
Privilege
        ↓
Linux Capability
        ↓
必要な権限だけを残す
```

という考え方です。

そして、コンテナの権限はKubernetesだけで決まるわけではありません。

```text
コンテナイメージ
      +
Kubernetes実行環境
      ↓
実際のコンテナの権限
```

という複数のレイヤーから確認する必要があります。

CKSでは、  
**「このSecurityContextを書けば安全」ではなく、「侵害されたときに、このコンテナには何ができるのか」**  
という視点から設定を判断できることを、この問題の到達点とします。
