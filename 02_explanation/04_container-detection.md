# 問題03：不審なファイルアクセスを検出する ― 解説・模範解答

## この問題では何を考えればよかったのか

この問題では、実行中のコンテナで発生する不審なファイルアクセスをFalcoで検出できるようにします。  
重要なのは、Falco Ruleの書式そのものを暗記することではありません。

まず、セキュリティ要件を整理します。

```text
検出したいこと
 ↓
/dev/memへのアクセス

何を観測するか
 ↓
アクセスされたファイル

何を条件にするか
 ↓
ファイル名

検出後に何をしたいか
 ↓
対象コンテナを調査する

何を記録するか
 ↓
Container ID
```

これをFalco Ruleへ変換します。

---

# Falcoは何をするためのものか

SecurityContextなどは、

```text
Workloadができることを制限する
```

ための設定です。

一方、FalcoはRuntime Securityの観点から、

```text
Workloadが実行中に
実際に何をしたか
        ↓
Runtime Eventを観測
        ↓
Ruleと照合
        ↓
異常な行動を検出
```

するために使用できます。

今回の問題では、

```text
/dev/memへアクセスした
```

という行動を異常として扱います。

---

# 既存ルールを確認する

Falco固有のRule構文をすべて記憶している必要はありません。  
まず既存ルールから、似た条件や出力方法を探します。

```bash
grep -n "fd.name" /etc/falco/falco_rules.yaml
```

Container IDの出力例についても確認できます。

```bash
grep -n "container.id" /etc/falco/falco_rules.yaml
```

PriorityやTagについても、既存Ruleを参考にできます。

この問題では、

```text
要件を理解する
        ↓
既存Ruleを調査する
        ↓
必要な構文を見つける
        ↓
自分のRuleへ組み合わせる
```

という進め方が重要です。

---

# なぜ既存ルールを直接編集しないのか

Falcoには既存のルールファイルがあります。

```text
/etc/falco/falco_rules.yaml
```

しかし、今回のカスタムルールは、

```text
/etc/falco/falco_rules.local.yaml
```

へ追加します。

既存ルールと独自ルールを分離することで、

```text
Falco標準ルール
        ↓
製品・配布側で管理

独自検出ルール
        ↓
利用環境側で管理
```

という責任分離ができます。

---

# Listを作成する

今回検出する対象は、

```text
/dev/mem
```

です。

これをFalcoのListとして定義します。

```yaml
- list: dev-file
  items:
    - /dev/mem
```

Listを利用することで、

```text
検出対象
 ↓
Ruleの条件へ直接埋め込む
```

のではなく、

```text
検出対象の定義
        ↓
List

検出ロジック
        ↓
Rule
```

として分離できます。

---

# Conditionを考える

今回知りたいのは、

> アクセスされたファイルが`/dev/mem`か

です。

元問題では、この判定に`fd.name`を使用しています。

したがって、

```yaml
condition: fd.name in (dev-file)
```

とします。

構造として読むと、

```text
fd.name
 ↓
アクセスされたファイル名

in
 ↓
指定Listに含まれているか

dev-file
 ↓
/dev/mem
```

となります。

---

# Outputを考える

検出するだけでは、その後の調査が難しくなります。

今回の要件では、

> どのコンテナで発生したのか後から調査したい

ため、Container IDを出力します。

Falcoでは、

```text
%container.id
```

を利用できます。

例えば、

```yaml
output: >
  Access to devmem
  (container_id=%container.id)
```

とします。

これにより、実際にイベントが発生した場合には、対象Containerを調査するための情報を残せる構成になります。

---

# 模範解答

`/etc/falco/falco_rules.local.yaml`へ、次のカスタムルールを追加します。

```yaml
- list: dev-file
  items:
    - /dev/mem

- rule: devmem
  desc: Detect access to /dev/mem
  condition: fd.name in (dev-file)
  output: >
    Access to devmem
    (container_id=%container.id)
  priority: NOTICE
  tags:
    - file
```

---

# Ruleを読む

## list

```yaml
- list: dev-file
  items:
    - /dev/mem
```

検出対象となるファイルを定義しています。

---

## rule

```yaml
- rule: devmem
```

カスタムRuleの名称です。

---

## desc

```yaml
desc: Detect access to /dev/mem
```

Ruleが何を検出するものなのかを示します。

---

## condition

```yaml
condition: fd.name in (dev-file)
```

アクセスされたファイル名が、`dev-file` Listに含まれているかを判定します。

---

## output

```yaml
output: >
  Access to devmem
  (container_id=%container.id)
```

Ruleに一致した場合に出力する情報です。

今回は、後続調査に利用できるようContainer IDを含めています。

---

## priority

```yaml
priority: NOTICE
```

問題で指定されたPriorityを設定します。

---

## tags

```yaml
tags:
  - file
```

ファイルアクセスに関するRuleであることを分類できるようにします。

---

# 要件とFalco Ruleの対応

| 要件               | Falco設定            |
| ---------------- | ------------------ |
| `/dev/mem`を対象とする | `list`             |
| アクセスされたファイルを判定   | `fd.name`          |
| List内の対象と比較      | `in (dev-file)`    |
| 検出メッセージ          | `output`           |
| Containerを後から特定  | `%container.id`    |
| 重要度              | `priority: NOTICE` |
| Rule分類           | `tags`             |

重要なのは、この対応関係です。

```text
セキュリティ要件
        ↓
観測対象
        ↓
Condition
        ↓
検出時に必要な情報
        ↓
Output
```

---

# Falcoへ反映する

作成したカスタムルールをFalcoが読み込める状態にします。  
環境に用意されているFalcoの実行・再読み込み方法を確認し、カスタムルールを反映します。

元問題ではFalcoを起動してルールを読み込ませる手順が示されていますが、  
実際のサービス管理方法は演習環境の構成に依存します。

この問題では、  
**カスタムルールがFalcoから読み込める状態になっていること**  
までを完了条件とします。

---

# 今回あえて実施しないこと

元となった問題では、この後、

```text
Falco Alert
 ↓
Container ID
 ↓
crictl
 ↓
Pod特定
 ↓
Deployment特定
 ↓
replicas=0
```

というインシデント対応まで続きます。

今回の演習では、ここを対象外とします。

理由は、学習対象を、

```text
Runtime Eventを実際に再現すること
```

ではなく、

```text
検出要件をFalco Ruleへ変換すること
```

へ絞るためです。

したがって、

```text
検出ルール設計
        ↓
Rule作成
        ↓
Falcoへ反映
────────────
ここまでが問題03
```

となります。

---

# ケーススタディ

実際のRuntime Securityでは、同じ考え方を別の異常行動にも応用できます。

例えば、

```text
コンテナ内部で
想定外のShellが起動した
```

場合でも、

```text
何が異常か
 ↓
Shell起動

何を観測するか
 ↓
実行されたProcess

何を条件にするか
 ↓
Process名など

検出後に何が必要か
 ↓
Container / Podなどの識別情報
```

と分解できます。

つまり、検出対象が変わっても、

```text
異常行動を定義
        ↓
観測できるEventへ変換
        ↓
Condition
        ↓
調査に必要な情報をOutput
```

という考え方は変わりません。

---

# SecurityContextとの違い

SecurityContextでは、

```text
危険な操作
 ↓
できないように制限する
```

という方向でセキュリティを考えます。

Falcoでは、

```text
Runtimeで発生した操作
 ↓
観測
 ↓
異常な行動を検出
```

します。

したがって、

```text
SecurityContext
        ↓
予防・制限

Falco
        ↓
観測・検出
```

という役割の違いがあります。

どちらか一方だけではなく、

```text
できることを減らす
        +
それでも発生した異常を検出する
```

という多層的な防御につながります。

---

# この問題で学んでほしいこと

この問題で最も重要なのは、Falco RuleのYAMLを暗記することではありません。

覚えてほしいのは、

```text
何を異常とするのか
        ↓
その行動は何として観測できるのか
        ↓
何を条件にすれば検出できるのか
        ↓
検出後の調査に何が必要なのか
        ↓
Falco Ruleへ変換
```

という流れです。

今回なら、

```text
/dev/memへのアクセス
        ↓
ファイルアクセスを観測
        ↓
fd.nameで判定
        ↓
Containerを特定したい
        ↓
container.idを出力
```

となります。

Falcoはツール固有の知識が必要になるため、SecurityContextのように要件だけから機能名へ到達することは難しい部分があります。  

そのためこの問題では、Falcoを使用すること自体は前提として与え、  
**「Falcoを使うと分かった後に、検出したい行動をどのようにRuleへ変換するか」**  
を学習の中心とします。
