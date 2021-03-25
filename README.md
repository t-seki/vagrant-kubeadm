# 概要

以下の構成でクラスターを作成します。

host1 (Control Plane Node):

|項目|値|
|:--|:--|
|hostname|host1|
|IP|172.16.10.11|
    
host2 (Node):

|項目|値|
|:--|:--|
|hostname|host2|
|IP|172.16.10.12|

共通:

|項目|値|
|:--|:--|
|OS|CentOS 7 (CentOS Linux release 7.9.2009 (Core))|
|コンテナランタイム|containerd|
|クラスター構成ツール|kubeadm (v1.20.5)|
|CNIプラグイン|containerd (v1.4.4)|

# 01. VMの起動

[このリポジトリ](https://github.com/t-seki/vagrant-kubeadm) をクローンして、 `Vagrant up` で VM を起動します。
```
git clone vagrant-kubeadm
cd vagrant-kubeadm
vagrant up
```

VM が起動したら、ping を使って private_network で通信ができることを確認します。

```
[root@host1 vagrant]# ping -c 5 172.16.10.12
PING 172.16.10.12 (172.16.10.12) 56(84) bytes of data.
64 bytes from 172.16.10.12: icmp_seq=1 ttl=64 time=0.445 ms
64 bytes from 172.16.10.12: icmp_seq=2 ttl=64 time=0.407 ms
64 bytes from 172.16.10.12: icmp_seq=3 ttl=64 time=0.509 ms
64 bytes from 172.16.10.12: icmp_seq=4 ttl=64 time=0.444 ms
64 bytes from 172.16.10.12: icmp_seq=5 ttl=64 time=0.398 ms

--- 172.16.10.12 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4010ms
rtt min/avg/max/mdev = 0.398/0.440/0.509/0.045 ms
```

# 02. kubeadm のインストール

ここからは [Installing kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/) に沿って進めていきます。

また、root ユーザーとしてコマンドを実行していきます。

## 02-01. 前提条件の確認

最初は [Before you begin](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#before-you-begin) セクションを確認します。

Swap を無効にする必要があります。  
それ以外は VM を起動した時点で満たされていると思います。

```
# swap の確認
[root@host1 vagrant]# cat /proc/swaps
Filename                                Type            Size    Used    Priority
/swapfile                               file            2097148 264     -2
[root@host1 vagrant]# free -m
              total        used        free      shared  buff/cache   available
Mem:           1837         177         977           8         682        1475
Swap:          2047           0        2047

# swap を無効にする
[root@host1 vagrant]# swapoff -a

# 再度 swap の確認
[root@host1 vagrant]# cat /proc/swaps
Filename                                Type            Size    Used    Priority
[root@host1 vagrant]# free -m
              total        used        free      shared  buff/cache   available
Mem:           1837         175         979           8         682        1476
Swap:             0           0           0
```

## 02-02. iptables の設定

```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

## 02-03. コンテナランタイムのインストール

[こちらのページ](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd) に沿って containerd をインストールします。

```
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Setup required sysctl params, these persist across reboots.
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system

# Install containerd
yum install -y yum-utils
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
yum install containerd.io -y

# Configure containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Restart containerd
sudo systemctl restart containerd
```

containerd が起動していることを確認します。  
contaienrd のクライアントツールである `ctr` を使います。
```
[root@host1 vagrant]# ctr version
Client:
  Version:  1.4.4
  Revision: 05f951a3781f4f2c1911b05e61c160e9c30eaa8e
  Go version: go1.13.15

Server:
  Version:  1.4.4
  Revision: 05f951a3781f4f2c1911b05e61c160e9c30eaa8e
  UUID: 65476b7f-cdfb-4c13-86bd-0217cedd3ce8
```

## 02-04. cgroup ドライバーに systemd を使用する設定

cgroup の選択についてはいまいち分からないんですが(汗)  
[こちらのページ](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd-systemd) を参考に設定を行います。

`/etc/containerd/config.toml` を編集します。
```
*** 92,101 ****
--- 92,102 ----
            runtime_engine = ""
            runtime_root = ""
            privileged_without_host_devices = false
            base_runtime_spec = ""
            [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
+             SystemdCgroup = true
      [plugins."io.containerd.grpc.v1.cri".cni]
        bin_dir = "/opt/cni/bin"
        conf_dir = "/etc/cni/net.d"
        max_conf_num = 1
        conf_template = ""
```

containerd を再起動します。

```
sudo systemctl restart containerd
```

## 02-05. kubeadm, kubelet, kubectl のインストール

[こちら](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#k8s-install-1) に沿ってインストールを行います。  
Node(host2) には kubectl は無くても大丈夫です。

```
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# Set SELinux in permissive mode (effectively disabling it)
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

sudo systemctl enable --now kubelet
```

# 03 クラスタの作成

## 03-01. コントロールプレーンで kubelet の使用する cgroup ドライバーを設定する

`kubeadm init` を行う際に設定ファイルを使用することで設定します。  
詳しくは [03-02](#03-02-コントロールプレーンノードを作成する) に記載します。

## 03-02. コントロールプレーンノードを作成する

さあ、kubeadm を使ってクラスターを作成していきます。

まずは [こちら](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#initializing-your-control-plane-node) を参考にコントロールプレーンノードを作成します。

[03-01](#03-01-kubelet-の使用する-cgroup-ドライバーを設定する) で書いたように、  
cgroup の設定が含まれた形で設定ファイルを準備します。  
(必要な設定のうち CLI オプションで未対応のものがあるので設定ファイルを使用します)

まず、kubeadm で初期状態の設定ファイルを生成します。

```
[root@host1 vagrant]# kubeadm config print init-defaults --component-configs KubeletConfiguration | tee ./kubeadm-config.yaml
W0323 15:25:02.103750   25901 kubelet.go:200] cannot automatically set CgroupDriver when starting the Kubelet: cannot execute 'docker info -f {{.CgroupDriver}}': executable file not found in $PATH
W0323 15:25:02.109797   25901 kubelet.go:200] cannot automatically set CgroupDriver when starting the Kubelet: cannot execute 'docker info -f {{.CgroupDriver}}': executable file not found in $PATH
apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 1.2.3.4
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: host1
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}

dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: k8s.gcr.io
kind: ClusterConfiguration
kubernetesVersion: v1.20.0
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
scheduler: {}
---
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 0s
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 0s
    cacheUnauthorizedTTL: 0s
clusterDNS:
- 10.96.0.10
clusterDomain: cluster.local
cpuManagerReconcilePeriod: 0s
evictionPressureTransitionPeriod: 0s
fileCheckFrequency: 0s
healthzBindAddress: 127.0.0.1
healthzPort: 10248
httpCheckFrequency: 0s
imageMinimumGCAge: 0s
kind: KubeletConfiguration
logging: {}
nodeStatusReportFrequency: 0s
nodeStatusUpdateFrequency: 0s
rotateCertificates: true
runtimeRequestTimeout: 0s
shutdownGracePeriod: 0s
shutdownGracePeriodCriticalPods: 0s
staticPodPath: /etc/kubernetes/manifests
streamingConnectionIdleTimeout: 0s
syncFrequency: 0s
volumeStatsAggPeriod: 0s
```

変更するのは以下の5点です。

+ `localAPIEndpoint.advertiseAddress` に host1 の IP アドレス (172.16.10.11) を設定します。
+ `nodeRegistration.criSocket` に containerd の UNIX ソケットのパス (/run/containerd/containerd.sock) を設定します。
+ `networking.podSubnet` に Pod ネットワークの CIDR を設定します。(今回は 192.168.0.0/16 とします)
+ `cgroupDriver` に `systemd` を設定します。
+ `nodeRegistration.kubeletExtraArgs.node-ip` に host1 の Internal IP を設定します。(既定では Vagrant の NAT Network の IP が設定されてしまうため)

変更後、不要な部分を削除すると残りは次のようになります。

```
[root@host1 vagrant]# cat kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 172.16.10.11
nodeRegistration:
  criSocket: /run/containerd/containerd.sock
  kubeletExtraArgs:
    node-ip: "172.16.10.11"
---
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
networking:
  podSubnet: 192.168.0.0/16
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
```

この設定ファイルを使用して `kubeadm init` を実行します。

```
[root@host1 vagrant]# kubeadm init --config=kubeadm-config.yaml
[init] Using Kubernetes version: v1.20.5
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [host1 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 172.16.10.11]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [host1 localhost] and IPs [172.16.10.11 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [host1 localhost] and IPs [172.16.10.11 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[kubelet-check] Initial timeout of 40s passed.
[apiclient] All control plane components are healthy after 66.003491 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.20" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node host1 as control-plane by adding the labels "node-role.kubernetes.io/master=''" and "node-role.kubernetes.io/control-plane='' (deprecated)"
[mark-control-plane] Marking the node host1 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: c1adoi.uutl1ykwwc9pn5t0
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.16.10.11:6443 --token c1adoi.uutl1ykwwc9pn5t0 \
    --discovery-token-ca-cert-hash sha256:29207d9112c6a7c7a5f7860df924194570f1d7feed31ee3c7494436037252fc1
```

生成された kubeconfig を使ってクラスターにアクセスしてみます。

```
[root@host1 vagrant]# export KUBECONFIG=/etc/kubernetes/admin.conf
[root@host1 vagrant]# kubectl cluster-info
Kubernetes control plane is running at https://172.16.10.11:6443
KubeDNS is running at https://172.16.10.11:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
[root@host1 vagrant]# kubectl version
Client Version: version.Info{Major:"1", Minor:"20", GitVersion:"v1.20.5", GitCommit:"6b1d87acf3c8253c123756b9e61dac642678305f", GitTreeState:"clean", BuildDate:"2021-03-18T01:10:43Z", GoVersion:"go1.15.8", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"20", GitVersion:"v1.20.5", GitCommit:"6b1d87acf3c8253c123756b9e61dac642678305f", GitTreeState:"clean", BuildDate:"2021-03-18T01:02:01Z", GoVersion:"go1.15.8", Compiler:"gc", Platform:"linux/amd64"}
[root@host1 vagrant]# kubectl get nodes -o wide
NAME    STATUS     ROLES                  AGE    VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION           CONTAINER-RUNTIME
host1   NotReady   control-plane,master   104s   v1.20.5   172.16.10.11   <none>        CentOS Linux 7 (Core)   3.10.0-1127.el7.x86_64   containerd://1.4.4
```

アクセスすることが出来ました。

## 03-03. クラスターにノードを追加する

host2 をクラスターに追加します。

まず、kubeadm で初期状態の設定ファイルを生成します。

```
[root@host2 vagrant]# kubeadm config print join-defaults | tee kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta2
caCertPath: /etc/kubernetes/pki/ca.crt
discovery:
  bootstrapToken:
    apiServerEndpoint: kube-apiserver:6443
    token: abcdef.0123456789abcdef
    unsafeSkipCAVerification: true
  timeout: 5m0s
  tlsBootstrapToken: abcdef.0123456789abcdef
kind: JoinConfiguration
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: host2
  taints: null
```

変更する点は以下の3点です。

+ `discovery.bootstrapToken.apiServerEndpoint` に host1 の IP アドレス (172.16.10.11) を設定します。
+ `discovery.bootstrapToken.token` に生成された Token を設定します。
+ `discovery.bootstrapToken.caCertHashes` に生成された DiscoveryTokenCaCertHash を設定します。
+ `nodeRegistration.criSocket` に containerd の UNIX ソケットのパス (/run/containerd/containerd.sock) を設定します。
+ `nodeRegistration.kubeletExtraArgs.node-ip` に host2 の Internal IP を設定します。

変更後、不要な部分を削除すると残りは次のようになります。

```
apiVersion: kubeadm.k8s.io/v1beta2
discovery:
  bootstrapToken:
    apiServerEndpoint: 172.16.10.11:6443
    token: c1adoi.uutl1ykwwc9pn5t0
    caCertHashes:
    - sha256:29207d9112c6a7c7a5f7860df924194570f1d7feed31ee3c7494436037252fc1
kind: JoinConfiguration
nodeRegistration:
  criSocket: /run/containerd/containerd.sock
  kubeletExtraArgs:
    node-ip: "172.16.10.12"
```

この設定ファイルを使用して `kubeadm init` を実行します。

```
[root@host2 vagrant]# kubeadm join --config=kubeadm-config.yaml
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

ノードが追加されたことを確認します。

```
[root@host1 vagrant]# kubectl get nodes -o wide
NAME    STATUS     ROLES                  AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION           CONTAINER-RUNTIME
host1   NotReady   control-plane,master   17m   v1.20.5   172.16.10.11   <none>        CentOS Linux 7 (Core)   3.10.0-1127.el7.x86_64   containerd://1.4.4
host2   NotReady   <none>                 43s   v1.20.5   172.16.10.12   <none>        CentOS Linux 7 (Core)   3.10.0-1127.el7.x86_64   containerd://1.4.4
```

ノードが追加されていることが確認できました。

Pod ネットワークを作成して STATUS を Ready にしましょう。

## 03-04. CNI プラグインをインストールする

[こちらのページ](https://docs.projectcalico.org/getting-started/kubernetes/quickstart#install-calico) を参考にインストールを行います。

```
[root@host1 vagrant]# kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/kubecontrollersconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/imagesets.operator.tigera.io created
customresourcedefinition.apiextensions.k8s.io/installations.operator.tigera.io created
customresourcedefinition.apiextensions.k8s.io/tigerastatuses.operator.tigera.io created
namespace/tigera-operator created
podsecuritypolicy.policy/tigera-operator created
serviceaccount/tigera-operator created
clusterrole.rbac.authorization.k8s.io/tigera-operator created
clusterrolebinding.rbac.authorization.k8s.io/tigera-operator created
deployment.apps/tigera-operator created
[root@host1 vagrant]# kubectl create -f https://docs.projectcalico.org/manifests/custom-resources.yaml
installation.operator.tigera.io/default created
```

calico のリソースが作成されていることを確認します。

```
[root@host1 vagrant]# kubectl get pods -n calico-system
NAME                                      READY   STATUS    RESTARTS   AGE
calico-kube-controllers-f95867bfb-q7kp2   1/1     Running   0          72s
calico-node-njxcp                         1/1     Running   0          72s
calico-node-zzn95                         1/1     Running   0          72s
calico-typha-7b65b9686d-6hgmf             1/1     Running   0          65s
calico-typha-7b65b9686d-sgsdk             1/1     Running   0          72s
```

Node の状態を確認してみます。

```
[root@host1 vagrant]# kubectl get nodes
NAME    STATUS   ROLES                  AGE     VERSION
host1   Ready    control-plane,master   19m     v1.20.5
host2   Ready    <none>                 3m18s   v1.20.5
```

どちらのノードも Ready になりました。

# 4. 動作確認

nginx Pod を ClusterIP で公開して、busybox からアクセスしてみようと思います。

```
[root@host1 vagrant]# kubectl run nginx --image=nginx
pod/nginx created
[root@host1 vagrant]# kubectl expose pod nginx --port 80
service/nginx exposed
[root@host1 vagrant]# kubectl get svc nginx
NAME    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
nginx   ClusterIP   10.96.165.73   <none>        80/TCP    21s
[root@host1 vagrant]# kubectl run testpod --image=busybox -it --rm --command -- wget -O- -q 10.96.165.73
If you don't see a command prompt, try pressing enter.
Error attaching, falling back to logs: unable to upgrade connection: container testpod not found in pod testpod_default
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
pod "testpod" deleted
```

nginx から HTML が返却されました。