# 問題１：CISベンチマーク

## 【問題】
kubeadm で作成されたクラスターに対して CIS ベンチマーク ツールを実行すると、すぐに対処する必要がある複数の問題が見つかりました。
新しい設定が有効になるように、影響を受けるコンポーネントを構成して再起動し、すべての問題を修正しなさい。

```
kubeletで見つかった問題
・authentication anonymous  が false に設定されていることを確認
・authorization mode パラメータが AlwaysAllow に設定されていないことを確認
```

```
etcdで見つかった問題
・--client-cert-authパラメータが設定されていることを確認
```

## 【■事前準備】特になし


## 【▼回答】

#### ▼kubeletからの問題解決

###### ▼kubeletのconfig.yamlがあるディレクトリに移動（直接編集するのも問題無し）とコピー
```bash
cd /var/lib/kubelet
cp config.yaml /opt/
```

###### ▼configファイルの編集
```yaml
vim config.yaml

apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false  #trueからfalseへ変更
  webhook:
    cacheTTL: 0s
    enabled: true # Webhookを有効にする
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
authorization:
  mode: Webhook # Webhookを使用するモードに変更
  webhook:
    cacheAuthorizedTTL: 0s
```
#### ▼etcd.yamlを編集しての問題解決

###### ▼etcdのYamlファイルがあるディレクトリに移動（直接編集するのも問題無し）とコピー
```bash
cd /etc/kubernetes/manifests/
cp etcd.yaml /opt/
```
###### ▼etcdファイルの編集
```yaml
vim etcd.yaml

apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubeadm.kubernetes.io/etcd.advertise-client-urls: https://11.0.1.111:2379
  creationTimestamp: null
  labels:
    component: etcd
    tier: control-plane
  name: etcd
  namespace: kube-system
spec:
  containers:
  - command:
    - etcd
    - --advertise-client-urls=https://11.0.1.111:2379
    - --cert-file=/etc/kubernetes/pki/etcd/server.crt
    - --client-cert-auth=true　# クライアント証明書を有効にするため、false を true に変更
    - --data-dir=/var/lib/etcd

```

#### ▼kubeletとetcdの再起動（再起動後にPodの一覧が表示されれば成功になります）
```bash
systemctl daemon-reload && systemctl restart kubelet
kubectl get pod 
```

## ★解説と補足
```
kubeletとetcdで見つかった問題をファイル編集を実施して解決する問題になります

問題の解答を行った際に、--client-cert-auth=true に設定した後、kubectlコマンドが通らなくなりました。
調べた結果、おそらくですが下記の解決方法が必要になる可能性が高いです。

この解決方法には、設定前にetcd.yamlに下記設定内容があるかを確認して、無ければ追加を行うことになります。
--cert-file=/etc/kubernetes/pki/etcd/server.crt
--key-file=/etc/kubernetes/pki/etcd/server.key
--trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt

上記のパスを設定する必要があります。そのパスに "/etc/kubernetes/pki/etcd/" に
server.crt
server.key
ca.crt

この3つがあるかを確認した上でパスを指定するようにしてください。

この問題は改変前のCKSにも出題された問題になりますが、補足事項の内容が追加になっているのであれば
一手間必要になる問題になります。
```