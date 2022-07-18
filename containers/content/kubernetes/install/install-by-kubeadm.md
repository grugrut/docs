---
title: kubeadmによるインストール
weight: 10
---

## 事前準備
Ubuntuをインストールする。必要なものはあとから入れるので最小構成で十分。

### タイムゾーンの設定。

``` shell
sudo timedatectl set-timezone Asia/Tokyo
```

### swapの無効化

/etc/fstabを変更して、以下の行を削除する。削除したら再起動する。
再起動後に、不要な/swap.imgを削除しておく。

```
/swap.img      none    swap    sw      0       0
```

## containerdのインストール

各サーバで以下のコマンドを実行する。

### カーネルパラメータの設定。

``` shell
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

### containerdのインストール。

``` shell
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y containerd.io
```

### CNIプラグインのインストール
containerd.ioのaptパッケージには、CNIプラグインは含まれていないので、手動でインストールする。

``` shell
sudo mkdir -p /opt/cni/bin
wget -O /tmp/cni-plugins-linux-amd64-v1.1.1.tgz https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz
sudo tar Cxzvf /opt/cni/bin /tmp/cni-plugins-linux-amd64-v1.1.1.tgz
```

### containerdの設定
/etc/containerd/config.toml を編集して、以下の内容にする。
デフォルトだとCRI統合プラグインが無効化されているので、それを有効化するのと、systemdがcgoupドライバーを利用するようにする。

``` toml
disabled_plugins = []

[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  SystemdCgroup = true
```
修正したら、containerdを再起動する。

``` shell
sudo systemctl restart containerd
```

### cgoup v1の有効化
Ubuntu 22.04の場合、cgourp v2が有効になっているが、それではこの後kubernetesをインストールした後に、kube-apiserverなどが再起動を繰り返す事象が発生するので、cgourp v1を利用するように修正する。

/etc/default/grub を修正し、以下の設定を入れる。

```
GRUB_CMDLINE_LINUX="systemd.unified_cgroup_hierarchy=0"
```
そうしたら以下のコマンドを実行し、再起動する。

``` shell
sudo update-grub
sudo reboot
```

## kubeadmのインストール

全サーバに対して、kubeadm, kubelet, kubectl をインストールする。

``` shell
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

## kubeadmによるkubernetesインストール
マスターサーバで以下コマンドを実行する。

``` shell
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```
成功すると、各ノードで以下のコマンドを実行せよ、と表示されるので各ノードで実行する。

``` shell
sudo kubeadm join 192.168.xxx.xxx:6443 --token xxxxxx.xxxxxxxxxxxxxxxx \
        --discovery-token-ca-cert-hash sha256:1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef
```

## CNIプラグインのインストール
今回はCalicoを利用する。

https://projectcalico.docs.tigera.io/getting-started/kubernetes/self-managed-onprem/onpremises

``` shell
kubectl create -f https://projectcalico.docs.tigera.io/manifests/tigera-operator.yaml
curl https://projectcalico.docs.tigera.io/manifests/custom-resources.yaml -O
```

デフォルトだとネットワークレンジが `192.168.0.0/16` になっているので、
インストール環境にあわせて修正する。

``` yaml
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  # Configures Calico networking.
  calicoNetwork:
    # Note: The ipPools section cannot be modified post-install.
    ipPools:
    - blockSize: 26
      cidr: 10.244.0.0/16
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()
```

``` shell
kubectl create -f custom-resources.yaml
```



