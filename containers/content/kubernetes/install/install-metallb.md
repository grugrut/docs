---
title: MetalLBのインストール
weight: 20
---

## kube-proxyの修正
kube-proxyをIPVSモードにしたうえで、strictARPを有効にする。

`kubectl -n kube-system edit cm kube-proxy` して、 `mode: ""` になっているところを、 `mode: ipvs` に修正、 `strictARP: false` を `strictARP: true` に修正する。
修正したら、 `kubectl -n kube-system rollout restart kube-proxy` して再起動する。

## MetalLBのインストール

以下のコマンドを実行し、インストールする。

``` shell
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.3/config/manifests/metallb-native.yaml
```

## MetalLBの設定

以下のようなYamlを適用し、LB作成時に割り当てられるIPアドレスを設定する。
BGPモードではなくて、L2モードを使うのでそちらの設定も作成しておく。

``` yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: ippools
  namespace: metallb-system
spec:
  addresses:
  - 192.168.2.64/27
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2
  namespace: metallb-system
spec:
  ipAddressPools:
  - ippools
  nodeSelectors:
  - matchLabels:
      kubernetes.io/hostname: master
```

## MetalLBのテスト
以下のようなYamlを適用し、type: LoadBalancerでサービスが公開されることを確認する。

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx-test
  name: nginx-test
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-test
  template:
    metadata:
      labels:
        app: nginx-test
    spec:
      containers:
      - image: nginx
        name: nginx
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-test
  namespace: default
  annotations:
    metallb.universe.tf/address-pool: ippools
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: nginx-test
  type: LoadBalancer
```



