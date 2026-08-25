# 問題10：セキュリティ基準を満たすようにWorkloadの実行権限を制限する ― 解説・模範解答

## この問題では何を考えればよかったのか

この問題では、Namespaceに適用されているPodのセキュリティ基準に対して、  
既存Deploymentを適合させます。

重要なのは、

```text
Podが作れない
    ↓
セキュリティ基準を緩和する
```

と考えることではありません。

今回修正するのはWorkload側です。

```text
Namespaceにセキュリティ基準がある
        ↓
Deploymentが基準を満たしていない
        ↓
違反内容を確認する
        ↓
必要な実行制約を整理する
        ↓
SecurityContextへ変換する
        ↓
Podを再作成する
```

つまり、

**「セキュリティ基準 → 要求される状態 → Kubernetes設定」**

へ翻訳することが、この問題のポイントです。

---

# 問題文を整理する

今回必要になる制御は4つです。

| セキュリティ要件            | 実現したい状態                      |
| ------------------- | ---------------------------- |
| 特権昇格を禁止             | プロセスが現在より高い権限を取得できない         |
| Linux Capabilityを制限 | 不要なCapabilityを保持しない          |
| root実行を禁止           | 非rootユーザーとして実行する             |
| system callを制限      | Runtime標準のseccompプロファイルを使用する |

これらはすべて、

```text
コンテナが侵害された場合に
どこまでOS側の機能を利用できるか
```

を制御する設定です。

---

# 違反内容を確認する

対象Deploymentを確認します。

```bash
kubectl get deployment -n confidential
```

対象は、

```text
nginx-unprivileged-deployment
```

です。

マニフェストを適用した際にPod Security Standardsへの違反が表示される場合、その内容を確認します。

元問題では、次の違反が確認されています。

```text
allowPrivilegeEscalation != false
unrestricted capabilities
runAsNonRoot != true
seccompProfile
```

ここから、

```text
何が足りないのか
        ↓
どの実行制約が必要なのか
```

を整理します。

---

# 設計の順番

## 1. 特権昇格を禁止する

最初の要件は、

```text
コンテナ内のプロセスが
現在より高い権限を取得できないこと
```

です。

SecurityContextでは、

```yaml
allowPrivilegeEscalation: false
```

を使用します。

これによって、コンテナ内のプロセスが権限昇格することを禁止します。

---

## 2. Linux Capabilityを削除する

Linuxでは、rootが持つ強い権限の一部をCapabilityという単位で分割しています。  
今回のセキュリティ基準では、不要なCapabilityを保持させません。

そのため、

```yaml
capabilities:
  drop:
  - ALL
```

とします。

考え方としては、

```text
必要か分からないCapabilityを残す
        ↓
×

一度すべて削除する
        ↓
必要なものだけ追加する
        ↓
○
```

です。

今回のWorkloadでは追加Capabilityの指定は要求されていないため、`ALL`を削除します。

---

## 3. rootユーザーで実行しない

次の要件は、

```text
コンテナをrootユーザーとして実行しない
```

ことです。

そのため、

```yaml
runAsNonRoot: true
```

を設定します。

これによって、コンテナを非rootユーザーとして実行することを要求します。

---

## 4. seccompプロファイルを設定する

最後の要件は、コンテナから利用できるsystem callを制限することです。

今回指定されているのは、

```text
コンテナランタイムの標準的なseccompプロファイル
```

です。

そのため、

```yaml
seccompProfile:
  type: RuntimeDefault
```

を設定します。

---

# Kubernetesではどの設定になるのか

今回必要になるSecurityContextは次の形です。

```yaml
securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop:
    - ALL
  runAsNonRoot: true
  seccompProfile:
    type: RuntimeDefault
```

この4つを、

```text
特権昇格
Capability
実行ユーザー
system call
```

という異なる制御点として理解してください。

---

# 模範解答

対象マニフェストを編集します。

```bash
vim ~/nginx-unprivileged.yaml
```

修正例は次の通りです。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-unprivileged-deployment
  namespace: confidential
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginxinc/nginx-unprivileged
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          runAsNonRoot: true
          seccompProfile:
            type: RuntimeDefault
```

今回変更するのは、コンテナの実行権限に関する部分です。

既存の、

```text
Deployment名
Image
Container Port
Label
```

などは変更しません。

---

# SecurityContextを読む

## allowPrivilegeEscalation

```yaml
allowPrivilegeEscalation: false
```

コンテナ内のプロセスが現在より高い権限を取得することを禁止します。

つまり、

```text
現在の権限
    ↓
より高い権限を取得
    ↓
禁止
```

です。

---

## capabilities.drop

```yaml
capabilities:
  drop:
  - ALL
```

コンテナが保持するLinux Capabilityを削除します。

これによって、

```text
コンテナ侵害
    ↓
OS側で利用できる特殊な権限
    ↓
できるだけ小さくする
```

という状態を作ります。

---

## runAsNonRoot

```yaml
runAsNonRoot: true
```

コンテナをrootユーザーとして実行しないことを要求します。

ここで重要なのは、

```text
非rootで実行したい
```

というセキュリティ要件を、

```text
runAsNonRoot
```

へ変換していることです。

---

## seccompProfile

```yaml
seccompProfile:
  type: RuntimeDefault
```

コンテナランタイムが提供する標準的なseccompプロファイルを使用します。  
seccompは、プロセスが利用できるLinux system callを制限する仕組みです。

つまり、

```text
コンテナプロセス
    ↓
Linux system call
    ↓
すべて自由に利用させない
```

という制御です。

---

# 要件と設定の対応

| セキュリティ要件         | Kubernetes設定                          |
| ---------------- | ------------------------------------- |
| 特権昇格禁止           | `allowPrivilegeEscalation: false`     |
| 不要なCapabilityを削除 | `capabilities.drop: ALL`              |
| root実行禁止         | `runAsNonRoot: true`                  |
| 標準seccompプロファイル  | `seccompProfile.type: RuntimeDefault` |

この対応を、

```text
何を禁止したいのか
        ↓
どのOS機能を制御するのか
        ↓
SecurityContext
```

として理解してください。

---

# マニフェストを適用する

修正後のDeploymentを適用します。

```bash
kubectl apply -f ~/nginx-unprivileged.yaml
```

Deploymentを確認します。

```bash
kubectl get deployment -n confidential
```

Podも確認します。

```bash
kubectl get pods -n confidential
```

Podが、

```text
Running
```

になっていることを確認します。

---

# Pod Security Standardsへの違反が解消したことを確認する

マニフェスト適用時に、以前表示されていた違反が出力されないことを確認します。  
また、Deploymentの実際の設定を確認する場合は、

```bash
kubectl get deployment nginx-unprivileged-deployment \
  -n confidential \
  -o yaml
```

を使用できます。

各コンテナに次の設定が存在することを確認します。

```yaml
securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop:
    - ALL
  runAsNonRoot: true
  seccompProfile:
    type: RuntimeDefault
```

---

# Pod Security StandardsとSecurityContextの違い

ここは、この問題で特に重要です。

Pod Security Standardsは、

```text
どのようなPodを安全とみなすか
```

というセキュリティ基準です。

一方、SecurityContextは、

```text
PodやContainerを
実際にどの権限で実行するか
```

を設定します。

整理すると、

```text
Pod Security Standards
        ↓
ルール・基準

SecurityContext
        ↓
Workload側の実装
```

です。

今回の問題では、

```text
セキュリティ基準を下げる
```

のではなく、

```text
Workloadをセキュリティ基準へ適合させる
```

ことが求められています。

---

# なぜNamespace側を変更しないのか

セキュリティPolicyに適合しないWorkloadがある場合、

```text
Podを動かしたい
    ↓
Policyを緩める
```

という対応をすると、同じNamespace内の他のWorkloadについても  
セキュリティ基準が弱くなる可能性があります。

今回必要なのは、

```text
Security Policy
    ↓
維持

Workload
    ↓
適合させる
```

という対応です。

つまり、**安全基準をWorkloadに合わせるのではなく、  
Workloadを安全基準に合わせる**という考え方です。

---

# Container Hardeningとして考える

今回設定している4つは、すべてコンテナのHardeningに関係します。

```text
allowPrivilegeEscalation
        ↓
権限昇格を制限

capabilities
        ↓
OS側の特殊権限を制限

runAsNonRoot
        ↓
実行ユーザーを制限

seccomp
        ↓
system callを制限
```

これらはそれぞれ別のレイヤーを制御しています。  
そのため、1つだけ設定すれば安全になるわけではありません。

---

# Defense in Depthとして考える

例えば、アプリケーションに脆弱性があり、攻撃者がコンテナ内部でコードを実行できたとします。

セキュリティ制約が弱い場合、

```text
Application侵害
        ↓
Container内部でコード実行
        ↓
root権限
        ↓
Capability利用
        ↓
権限昇格
        ↓
広いsystem callを利用
```

という可能性があります。

一方で今回の設定では、

```text
Application侵害
        ↓
Non-root
        ↓
Capabilityなし
        ↓
Privilege Escalation不可
        ↓
seccomp制限
```

となります。

つまり、**侵入そのものを完全に防ぐのではなく、  
侵入された後に利用できる能力を小さくする**  
という考え方です。

---

# 第3問との接続

第3問でもSecurityContextを扱いました。

第3問では、

```text
Dockerfile
+
Kubernetes SecurityContext
        ↓
イメージと実行環境の両方をHardening
```

という問題でした。

今回の問題では、

```text
Pod Security Standards
        ↓
要求される基準
        ↓
SecurityContext
        ↓
Workloadを基準へ適合
```

という構造です。

つまり、

```text
第3問
 ↓
自分でセキュリティ要件を実装する

第10問
 ↓
既存のセキュリティ基準へ適合する
```

という違いがあります。

---

# 第7問・第9問との接続

これまでの問題を並べると、

```text
NetworkPolicy
    ↓
侵害後に通信できる範囲を小さくする

ServiceAccount Token
    ↓
侵害後に取得できる認証情報を小さくする

SecurityContext
    ↓
侵害後に利用できるOS権限を小さくする
```

となります。

それぞれ制御している対象は違いますが、**コンテナが侵害された場合の可動域を小さくする**  
という方向は共通しています。

---

# この問題で学んでほしいこと

この問題で覚えてほしいのは、

```yaml
allowPrivilegeEscalation: false
capabilities:
  drop:
  - ALL
runAsNonRoot: true
seccompProfile:
  type: RuntimeDefault
```

という設定そのものだけではありません。

重要なのは、

```text
セキュリティ基準
        ↓
何が禁止されているのか
        ↓
コンテナのどの能力を制御するのか
        ↓
SecurityContext
        ↓
Workloadへ適用
        ↓
基準を満たすことを確認
```

という判断の流れです。

Pod Security StandardsのようなPolicyが存在する環境では、  
**「Podを動かすためにPolicyを緩める」のではなく、  
「要求される安全基準を読み取り、Workloadをその基準へ適合させる」** ことが重要です。

CKSでは、**「SecurityContextの設定値を知っている」ことから一段進んで、  
「セキュリティ基準が要求している状態を読み取り、それをWorkloadの実行制約へ翻訳できる」こと**  
をこの問題の到達点とします。
