# 第3章 部署Kubernetes Cluster

目标:部署3个节点的Kubernetes Cluster

## 3.1 事前准备工作

### 3.1.1 安装Docker

- step1. [安装Docker引擎](https://github.com/rayallen20/DockerPrimer/blob/main/%E7%AC%AC2%E7%AB%A0%20%E6%A0%B8%E5%BF%83%E6%A6%82%E5%BF%B5%E4%B8%8E%E5%AE%89%E8%A3%85%E9%85%8D%E7%BD%AE/%E7%AC%AC2%E7%AB%A0%20%E6%A0%B8%E5%BF%83%E6%A6%82%E5%BF%B5%E4%B8%8E%E5%AE%89%E8%A3%85%E9%85%8D%E7%BD%AE.md#22-%E5%AE%89%E8%A3%85docker%E5%BC%95%E6%93%8E)

- step2. [配置国内镜像源](https://github.com/rayallen20/DockerPrimer/blob/main/%E7%AC%AC3%E7%AB%A0%20%E4%BD%BF%E7%94%A8Docker%E9%95%9C%E5%83%8F/%E7%AC%AC3%E7%AB%A0%20%E4%BD%BF%E7%94%A8Docker%E9%95%9C%E5%83%8F.md#%E9%85%8D%E7%BD%AE%E5%9B%BD%E5%86%85%E9%95%9C%E5%83%8F%E6%BA%90)

- step3. 调整docker的cgroup驱动

kebernetes默认设置cgroup驱动是systemd,而docker的cgroup驱动是cgroupfs.调整docker的cgroup驱动为systemd.

```
root@k8s-master:/home/soap# vim /etc/docker/daemon.json
root@k8s-master:/home/soap# cat /etc/docker/daemon.json
{
    "registry-mirrors": ["https://sb6xpp51.mirror.aliyuncs.com"],
    "exec-opts": ["native.cgroupdriver=systemd"]
}
```

- step4. 重启docker

```
root@k8s-master:/home/soap# systemctl restart docker
root@k8s-master:/home/soap# systemctl status docker
● docker.service - Docker Application Container Engine
     Loaded: loaded (/lib/systemd/system/docker.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2022-02-09 10:06:34 UTC; 4h 33min ago
TriggeredBy: ● docker.socket
       Docs: https://docs.docker.com
   Main PID: 6616 (dockerd)
      Tasks: 15
     Memory: 43.0M
     CGroup: /system.slice/docker.service
             └─6616 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock

Feb 09 10:06:34 k8s-master dockerd[6616]: time="2022-02-09T10:06:34.055228315Z" level=warning msg="Your kernel does not support CPU realtime scheduler"
Feb 09 10:06:34 k8s-master dockerd[6616]: time="2022-02-09T10:06:34.055325447Z" level=warning msg="Your kernel does not support cgroup blkio weight"
Feb 09 10:06:34 k8s-master dockerd[6616]: time="2022-02-09T10:06:34.055416770Z" level=warning msg="Your kernel does not support cgroup blkio weight_device"
Feb 09 10:06:34 k8s-master dockerd[6616]: time="2022-02-09T10:06:34.055655669Z" level=info msg="Loading containers: start."
Feb 09 10:06:34 k8s-master dockerd[6616]: time="2022-02-09T10:06:34.132966016Z" level=info msg="Default bridge (docker0) is assigned with an IP address 172.17.0.0/16. Daemon option --bip can be used to set a preferred IP address"
Feb 09 10:06:34 k8s-master dockerd[6616]: time="2022-02-09T10:06:34.165836370Z" level=info msg="Loading containers: done."
Feb 09 10:06:34 k8s-master dockerd[6616]: time="2022-02-09T10:06:34.188900324Z" level=info msg="Docker daemon" commit=459d0df graphdriver(s)=overlay2 version=20.10.12
Feb 09 10:06:34 k8s-master dockerd[6616]: time="2022-02-09T10:06:34.189145989Z" level=info msg="Daemon has completed initialization"
Feb 09 10:06:34 k8s-master systemd[1]: Started Docker Application Container Engine.
Feb 09 10:06:34 k8s-master dockerd[6616]: time="2022-02-09T10:06:34.205025511Z" level=info msg="API listen on /run/docker.sock"
```

这一步不做的话,kubelet启动会失败,错误码为1

### 3.1.2 禁用swap交换分区

此处设置方式为永久禁用交换分区

- step1. 设置根目录文件系统为可读写

```
root@k8s-master:/home/soap# sudo mount -n -o remount,rw /
```

- step2. 注释掉`/etc/fstab`中有swap信息的那行

```
root@k8s-master:/home/soap# vim /etc/fstab
root@k8s-master:/home/soap# cat /etc/fstab
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
# / was on /dev/ubuntu-vg/ubuntu-lv during curtin installation
/dev/disk/by-id/dm-uuid-LVM-GGdc6yC1ASeW555Ug2Tx2USlIACVo3bSJmovhNbeNftUrVCxw0aIEpDiIamTZnv9 / ext4 defaults 0 1
# /boot was on /dev/sda2 during curtin installation
/dev/disk/by-uuid/812168d2-97ac-4863-b9ab-da66de22633a /boot ext4 defaults 0 1
# /swap.img	none	swap	sw	0	0
```

- step3. 重启

```
root@k8s-master:/home/soap# reboot
```

- step4. 查看交换分区情况

```
root@k8s-master:/home/soap# free -m
              total        used        free      shared  buff/cache   available
Mem:          11977         650       10290           1        1036       11065
Swap:             0           0           0
```

这一步不做的话,kubelet启动会失败,错误码为255

## 3.2 安装kubelet、kubeadm和kubectl

- kubelet运行在Cluster所有节点上,负责启动Pod和容器
- kubeadm用于初始化Cluster
- kubectl是Kubernetes命令行工具.通过kubectl可以部署和管理应用,查看各种资源,创建、删除和更新各种组件

- step1. 更新依赖安装https

```
root@k8s-master:/home/soap# apt-get update && apt-get install -y apt-transport-https
Hit:1 https://download.docker.com/linux/ubuntu focal InRelease
Hit:2 http://cn.archive.ubuntu.com/ubuntu focal InRelease
Hit:3 http://cn.archive.ubuntu.com/ubuntu focal-updates InRelease
Hit:4 http://cn.archive.ubuntu.com/ubuntu focal-backports InRelease
Hit:5 http://cn.archive.ubuntu.com/ubuntu focal-security InRelease
Reading package lists... Done
Reading package lists... Done
Building dependency tree       
Reading state information... Done
apt-transport-https is already the newest version (2.0.6).
0 upgraded, 0 newly installed, 0 to remove and 39 not upgraded.
```

- step2. 获取密钥

```
root@k8s-master:/home/soap# curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - 
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  2537  100  2537    0     0   5637      0 --:--:-- --:--:-- --:--:--  5637
OK
```

- step3. 修改Kubernetes源

```
root@k8s-master:/home/soap# cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
> deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
> EOF
```

- step4. 升级依赖

```
root@k8s-master:/home/soap# apt-get update
Hit:1 https://download.docker.com/linux/ubuntu focal InRelease
Hit:2 http://cn.archive.ubuntu.com/ubuntu focal InRelease                                 
Hit:3 http://cn.archive.ubuntu.com/ubuntu focal-updates InRelease                         
Hit:4 http://cn.archive.ubuntu.com/ubuntu focal-backports InRelease
Get:5 https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial InRelease [9,383 B]
Hit:6 http://cn.archive.ubuntu.com/ubuntu focal-security InRelease
Ign:7 https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial/main amd64 Packages
Get:7 https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial/main amd64 Packages [53.6 kB]
Fetched 63.0 kB in 2s (29.5 kB/s)   
Reading package lists... Done
```

- step5. 安装

```
root@k8s-master:/home/soap# apt-get install -y kubelet kubeadm kubectl
Reading package lists... Done
Building dependency tree       
Reading state information... Done
The following additional packages will be installed:
  conntrack cri-tools ebtables kubernetes-cni socat
Suggested packages:
  nftables
The following NEW packages will be installed:
  conntrack cri-tools ebtables kubeadm kubectl kubelet kubernetes-cni socat
0 upgraded, 8 newly installed, 0 to remove and 39 not upgraded.
Need to get 73.6 MB of archives.
After this operation, 321 MB of additional disk space will be used.
Get:1 https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial/main amd64 cri-tools amd64 1.19.0-00 [11.2 MB]
Get:2 http://cn.archive.ubuntu.com/ubuntu focal/main amd64 conntrack amd64 1:1.4.5-2 [30.3 kB]
Get:3 http://cn.archive.ubuntu.com/ubuntu focal/main amd64 ebtables amd64 2.0.11-3build1 [80.3 kB]
Get:4 http://cn.archive.ubuntu.com/ubuntu focal/main amd64 socat amd64 1.7.3.3-2 [323 kB]
Get:5 https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial/main amd64 kubernetes-cni amd64 0.8.7-00 [25.0 MB]
Get:6 https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial/main amd64 kubelet amd64 1.23.3-00 [19.5 MB]                                                                                                                                                                 
Get:7 https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial/main amd64 kubectl amd64 1.23.3-00 [8,929 kB]                                                                                                                                                                
Get:8 https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial/main amd64 kubeadm amd64 1.23.3-00 [8,580 kB]                                                                                                                                                                
Fetched 73.6 MB in 22s (3,391 kB/s)                                                                                                                                                                                                                                            
Selecting previously unselected package conntrack.
(Reading database ... 71881 files and directories currently installed.)
Preparing to unpack .../0-conntrack_1%3a1.4.5-2_amd64.deb ...
Unpacking conntrack (1:1.4.5-2) ...
Selecting previously unselected package cri-tools.
Preparing to unpack .../1-cri-tools_1.19.0-00_amd64.deb ...
Unpacking cri-tools (1.19.0-00) ...
Selecting previously unselected package ebtables.
Preparing to unpack .../2-ebtables_2.0.11-3build1_amd64.deb ...
Unpacking ebtables (2.0.11-3build1) ...
Selecting previously unselected package kubernetes-cni.
Preparing to unpack .../3-kubernetes-cni_0.8.7-00_amd64.deb ...
Unpacking kubernetes-cni (0.8.7-00) ...
Selecting previously unselected package socat.
Preparing to unpack .../4-socat_1.7.3.3-2_amd64.deb ...
Unpacking socat (1.7.3.3-2) ...
Selecting previously unselected package kubelet.
Preparing to unpack .../5-kubelet_1.23.3-00_amd64.deb ...
Unpacking kubelet (1.23.3-00) ...
Selecting previously unselected package kubectl.
Preparing to unpack .../6-kubectl_1.23.3-00_amd64.deb ...
Unpacking kubectl (1.23.3-00) ...
Selecting previously unselected package kubeadm.
Preparing to unpack .../7-kubeadm_1.23.3-00_amd64.deb ...
Unpacking kubeadm (1.23.3-00) ...
Setting up conntrack (1:1.4.5-2) ...
Setting up kubectl (1.23.3-00) ...
Setting up ebtables (2.0.11-3build1) ...
Setting up socat (1.7.3.3-2) ...
Setting up cri-tools (1.19.0-00) ...
Setting up kubernetes-cni (0.8.7-00) ...
Setting up kubelet (1.23.3-00) ...
Created symlink /etc/systemd/system/multi-user.target.wants/kubelet.service → /lib/systemd/system/kubelet.service.
Setting up kubeadm (1.23.3-00) ...
Processing triggers for man-db (2.9.1-1) ...
```

- step6. 添加参数

在文件`/etc/systemd/system/kubelet.service.d/10-kubeadm.conf`中添加一行:`Environment="KUBELET_SYSTEM_PODS_ARGS=--pod-manifest-path=/etc/kubernetes/manifests --allow-privileged=true --fail-swap-on=false"`

```
root@k8s-master:/home/soap# vim /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
root@k8s-master:/home/soap# cat /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
# Note: This dropin only works with kubeadm and kubelet v1.11+
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
# This is a file that "kubeadm init" and "kubeadm join" generates at runtime, populating the KUBELET_KUBEADM_ARGS variable dynamically
EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
# This is a file that the user can use for overrides of the kubelet args as a last resort. Preferably, the user should use
# the .NodeRegistration.KubeletExtraArgs object in the configuration files instead. KUBELET_EXTRA_ARGS should be sourced from this file.
EnvironmentFile=-/etc/default/kubelet
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS

# 以下为手动添加内容
Environment="KUBELET_SYSTEM_PODS_ARGS=--pod-manifest-path=/etc/kubernetes/manifests --allow-privileged=true --fail-swap-on=false"
```

以上步骤在Kubernetes的所有node上都要做

## 3.3 用kubeadm创建Cluster

### 3.3.1 初始化Master

```
root@k8s-master:/home/soap# kubeadm init --image-repository registry.aliyuncs.com/google_containers --kubernetes-version v1.22.2 --ignore-preflight-errors=all --apiserver-advertise-address 192.168.0.154 --pod-network-cidr=10.244.0.0/16 
[init] Using Kubernetes version: v1.22.2
[preflight] Running pre-flight checks
	[WARNING Port-6443]: Port 6443 is in use
	[WARNING Port-10259]: Port 10259 is in use
	[WARNING Port-10257]: Port 10257 is in use
	[WARNING FileAvailable--etc-kubernetes-manifests-kube-apiserver.yaml]: /etc/kubernetes/manifests/kube-apiserver.yaml already exists
	[WARNING FileAvailable--etc-kubernetes-manifests-kube-controller-manager.yaml]: /etc/kubernetes/manifests/kube-controller-manager.yaml already exists
	[WARNING FileAvailable--etc-kubernetes-manifests-kube-scheduler.yaml]: /etc/kubernetes/manifests/kube-scheduler.yaml already exists
	[WARNING FileAvailable--etc-kubernetes-manifests-etcd.yaml]: /etc/kubernetes/manifests/etcd.yaml already exists
	[WARNING KubeletVersion]: the kubelet version is higher than the control plane version. This is not a supported version skew and may lead to a malfunctional cluster. Kubelet version: "1.23.3" Control plane version: "1.22.2"
	[WARNING Port-10250]: Port 10250 is in use
	[WARNING Port-2379]: Port 2379 is in use
	[WARNING Port-2380]: Port 2380 is in use
	[WARNING DirAvailable--var-lib-etcd]: /var/lib/etcd is not empty
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Using existing ca certificate authority
[certs] Using existing apiserver certificate and key on disk
[certs] Using existing apiserver-kubelet-client certificate and key on disk
[certs] Using existing front-proxy-ca certificate authority
[certs] Using existing front-proxy-client certificate and key on disk
[certs] Using existing etcd/ca certificate authority
[certs] Using existing etcd/server certificate and key on disk
[certs] Using existing etcd/peer certificate and key on disk
[certs] Using existing etcd/healthcheck-client certificate and key on disk
[certs] Using existing apiserver-etcd-client certificate and key on disk
[certs] Using the existing "sa" key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Using existing kubeconfig file: "/etc/kubernetes/admin.conf"
[kubeconfig] Using existing kubeconfig file: "/etc/kubernetes/kubelet.conf"
[kubeconfig] Using existing kubeconfig file: "/etc/kubernetes/controller-manager.conf"
[kubeconfig] Using existing kubeconfig file: "/etc/kubernetes/scheduler.conf"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 0.016572 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.22" in namespace kube-system with the configuration for the kubelets in the cluster
NOTE: The "kubelet-config-1.22" naming of the kubelet ConfigMap is deprecated. Once the UnversionedKubeletConfigMap feature gate graduates to Beta the default name will become just "kubelet-config". Kubeadm upgrade will handle this transition transparently.
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node k8s-master as control-plane by adding the labels: [node-role.kubernetes.io/master(deprecated) node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node k8s-master as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: 0pzfmn.gviozccniu2hp1dj
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

kubeadm join 192.168.0.154:6443 --token 0pzfmn.gviozccniu2hp1dj \
	--discovery-token-ca-cert-hash sha256:66882cc6f755d8597bcc6db4de3537ab5b88094c4cfc3950983974737abca05d 
```

- `--apiserver-advertise-address`:Master的IP地址.该选项指明用Master的那个interface与Cluster的其他节点通信.若Master有多个interface,建议明确指定,若不指定,kubeadm会自动选择有默认网关的interface


- `--pod-network-cidr=10.244.0.0/16`:指定Pod的网络范围.Kubernetes支持多种网络方案,每种方案对`--pod-network-cidr`有自己的要求,此处设置为`10.244.0.0/16`是因为我们将要使用flannel网络方案,必须设置成这个CIDR,后续可以替换为Canal

- `--image-repository`:指定镜像仓库

- `--kubernetes-version`:指定Kubernetes版本

- `--ignore-preflight-errors`:忽略preflight级报错信息.此处由于之前有过初始化失败的情况,故需加此选项

- `--v=5`:查看详细初始化过程可使用该选项

此命令主要做的操作有:

- 1. kubeadm执行初始化前的检查
- 2. 生成token和证书
- 3. 生成KubeConfig文件(默认为`/var/lib/kubelet/config.yaml`),kubelet需要用这个文件与Master通信
- 4. 安装Master组件
- 5. 安装附加组件kube-proxy和kube-dns
- 6. Kubernetes Master初始化成功
- 7. 提示如何配置kubectl
- 8. 提示如何安装Pod网络
- 9. 提示如何注册其他节点到Cluster

### 3.3.2 配置kubectl

kubectl是管理Kubernetes Cluster的命令行工具,前面我们已经在所有节点安装了kubectl.Master初始化完成后需要做一些配置工作,然后kubectl就能使用了

按照`kubeadm init`输出的第7步信息提示,推荐使用Linux普通用户执行kubectl(root用户会有一些问题)

```
su - soap
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

- step1. 切换用户

```
root@k8s-master:/home/soap# su - soap
soap@k8s-master:~$ who
soap     pts/0        2022-02-09 09:56 (192.168.0.101)
```

- step2. 创建用于保存配置文件的目录

```
soap@k8s-master:~$ mkdir -p $HOME/.kube
soap@k8s-master:~$ ls -a $HOME
.  ..  .bash_history  .bash_logout  .bashrc  .cache  .kube  .profile  .sudo_as_admin_successful
```

- step3. 复制配置文件到该目录

```
soap@k8s-master:~$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
soap@k8s-master:~$ tree $HOME/.kube
/home/soap/.kube
└── config

0 directories, 1 file
```

- step4. 修改配置文件的属组属主为普通用户

```
soap@k8s-master:~$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
soap@k8s-master:~$ ll $HOME/.kube/config
-rw------- 1 soap soap 5637 Feb  9 15:22 /home/soap/.kube/config
```

- step5. 启用kubectl命令的自动补全功能

```
soap@k8s-master:~$ echo "source <(kubectl completion bash)" >> ~/.bashrc
```

### 3.3.3 安装Pod网络

要让Kubernetes Cluster能够工作,必须先安装Pod网络,否则Pod之间无法通信.

Kubernetes支持多种网络方案,此处先使用flannel.

可以翻墙的情况下直接执行如下命令即可:

```
soap@k8s-master:~$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
Warning: policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
podsecuritypolicy.policy/psp.flannel.unprivileged created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.apps/kube-flannel-ds created
```

### 3.3.4 添加k8s-node1和k8s-node2

在k8s-node1和k8s-node2上分别执行如下命令,将其注册到Cluster中:

- 在k8s-node1上执行

```
root@k8s-node1:/home/allen# kubeadm join 192.168.0.154:6443 --token 0pzfmn.gviozccniu2hp1dj \
> --discovery-token-ca-cert-hash sha256:66882cc6f755d8597bcc6db4de3537ab5b88094c4cfc3950983974737abca05d
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
W0209 15:53:07.200031   12934 utils.go:69] The recommended value for "resolvConf" in "KubeletConfiguration" is: /run/systemd/resolve/resolv.conf; the provided value is: /run/systemd/resolve/resolv.conf
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

- 在k8s-node2上执行

```
root@k8s-node2:/home/nikolai# kubeadm join 192.168.0.154:6443 --token 0pzfmn.gviozccniu2hp1dj \
> --discovery-token-ca-cert-hash sha256:66882cc6f755d8597bcc6db4de3537ab5b88094c4cfc3950983974737abca05d
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
W0209 15:54:01.105342   10064 utils.go:69] The recommended value for "resolvConf" in "KubeletConfiguration" is: /run/systemd/resolve/resolv.conf; the provided value is: /run/systemd/resolve/resolv.conf
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

- 添加完成后,在k8s-master上查看结果

```
soap@k8s-master:~$ kubectl get nodes
NAME         STATUS     ROLES                  AGE     VERSION
k8s-master   NotReady   control-plane,master   5h48m   v1.23.3
k8s-node1    NotReady   <none>                 2m11s   v1.23.3
k8s-node2    NotReady   <none>                 78s     v1.23.3
```

可以看到所有节点此时都是NotReady状态.这是因为每个节点都需要启动若干组件,这些组件都是在Pod中运行,需要首先从Google下载镜像.

- 查看Pod状态

```
soap@k8s-master:~$ kubectl get pod --all-namespaces
NAMESPACE     NAME                                 READY   STATUS                  RESTARTS   AGE
kube-system   coredns-55dffbd598-8rcfq             0/1     Pending                 0          5h51m
kube-system   coredns-55dffbd598-w4c4z             0/1     Pending                 0          5h51m
kube-system   etcd-k8s-master                      1/1     Running                 0          5h52m
kube-system   kube-apiserver-k8s-master            1/1     Running                 0          5h52m
kube-system   kube-controller-manager-k8s-master   1/1     Running                 0          5h52m
kube-system   kube-flannel-ds-7w96q                0/1     Init:ImagePullBackOff   0          10m
kube-system   kube-flannel-ds-dxwnk                0/1     Init:1/2                0          5m2s
kube-system   kube-flannel-ds-tx967                0/1     Init:1/2                0          5m56s
kube-system   kube-proxy-5wvtm                     1/1     Running                 0          5m56s
kube-system   kube-proxy-n52bm                     1/1     Running                 0          5m2s
kube-system   kube-proxy-qzxsp                     1/1     Running                 0          5h51m
kube-system   kube-scheduler-k8s-master            1/1     Running                 0          5h52m
```

ImagePullBackOff表明Pod没有就绪.Running才是就绪状态.

- 查看Pod具体情况

```
soap@k8s-master:~$ kubectl describe pod kube-flannel-ds-tx967 --namespace=kube-system
Name:                 kube-flannel-ds-tx967
Namespace:            kube-system
Priority:             2000001000
Priority Class Name:  system-node-critical
Node:                 k8s-node1/192.168.0.155
Start Time:           Wed, 09 Feb 2022 15:53:09 +0000
Labels:               app=flannel
                      controller-revision-hash=cd4db5bc6
                      pod-template-generation=1
                      tier=node
Annotations:          <none>
Status:               Pending
IP:                   192.168.0.155
IPs:
  IP:           192.168.0.155
Controlled By:  DaemonSet/kube-flannel-ds
Init Containers:
  install-cni-plugin:
    Container ID:  docker://518725687ef5f69dcf26bfc59bde4fd2848960f201cd2830942ed2591b0f16e0
    Image:         rancher/mirrored-flannelcni-flannel-cni-plugin:v1.0.1
    Image ID:      docker-pullable://rancher/mirrored-flannelcni-flannel-cni-plugin@sha256:5dd61f95e28fa7ef897ff2fa402ce283e5078d334401d2f62d00a568f779f2d5
    Port:          <none>
    Host Port:     <none>
    Command:
      cp
    Args:
      -f
      /flannel
      /opt/cni/bin/flannel
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Wed, 09 Feb 2022 15:58:29 +0000
      Finished:     Wed, 09 Feb 2022 15:58:29 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /opt/cni/bin from cni-plugin (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-sv89r (ro)
  install-cni:
    Container ID:  
    Image:         rancher/mirrored-flannelcni-flannel:v0.16.3
    Image ID:      
    Port:          <none>
    Host Port:     <none>
    Command:
      cp
    Args:
      -f
      /etc/kube-flannel/cni-conf.json
      /etc/cni/net.d/10-flannel.conflist
    State:          Waiting
      Reason:       PodInitializing
    Ready:          False
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /etc/cni/net.d from cni (rw)
      /etc/kube-flannel/ from flannel-cfg (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-sv89r (ro)
Containers:
  kube-flannel:
    Container ID:  
    Image:         rancher/mirrored-flannelcni-flannel:v0.16.3
    Image ID:      
    Port:          <none>
    Host Port:     <none>
    Command:
      /opt/bin/flanneld
    Args:
      --ip-masq
      --kube-subnet-mgr
    State:          Waiting
      Reason:       PodInitializing
    Ready:          False
    Restart Count:  0
    Limits:
      cpu:     100m
      memory:  50Mi
    Requests:
      cpu:     100m
      memory:  50Mi
    Environment:
      POD_NAME:       kube-flannel-ds-tx967 (v1:metadata.name)
      POD_NAMESPACE:  kube-system (v1:metadata.namespace)
    Mounts:
      /etc/kube-flannel/ from flannel-cfg (rw)
      /run/flannel from run (rw)
      /run/xtables.lock from xtables-lock (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-sv89r (ro)
Conditions:
  Type              Status
  Initialized       False 
  Ready             False 
  ContainersReady   False 
  PodScheduled      True 
Volumes:
  run:
    Type:          HostPath (bare host directory volume)
    Path:          /run/flannel
    HostPathType:  
  cni-plugin:
    Type:          HostPath (bare host directory volume)
    Path:          /opt/cni/bin
    HostPathType:  
  cni:
    Type:          HostPath (bare host directory volume)
    Path:          /etc/cni/net.d
    HostPathType:  
  flannel-cfg:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      kube-flannel-cfg
    Optional:  false
  xtables-lock:
    Type:          HostPath (bare host directory volume)
    Path:          /run/xtables.lock
    HostPathType:  FileOrCreate
  kube-api-access-sv89r:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   Burstable
Node-Selectors:              <none>
Tolerations:                 :NoSchedule op=Exists
                             node.kubernetes.io/disk-pressure:NoSchedule op=Exists
                             node.kubernetes.io/memory-pressure:NoSchedule op=Exists
                             node.kubernetes.io/network-unavailable:NoSchedule op=Exists
                             node.kubernetes.io/not-ready:NoExecute op=Exists
                             node.kubernetes.io/pid-pressure:NoSchedule op=Exists
                             node.kubernetes.io/unreachable:NoExecute op=Exists
                             node.kubernetes.io/unschedulable:NoSchedule op=Exists
Events:
  Type     Reason     Age                   From               Message
  ----     ------     ----                  ----               -------
  Normal   Scheduled  7m46s                 default-scheduler  Successfully assigned kube-system/kube-flannel-ds-tx967 to k8s-node1
  Warning  Failed     4m21s                 kubelet            Failed to pull image "rancher/mirrored-flannelcni-flannel-cni-plugin:v1.0.1": rpc error: code = Unknown desc = context canceled
  Warning  Failed     4m21s                 kubelet            Error: ErrImagePull
  Normal   BackOff    4m20s                 kubelet            Back-off pulling image "rancher/mirrored-flannelcni-flannel-cni-plugin:v1.0.1"
  Warning  Failed     4m20s                 kubelet            Error: ImagePullBackOff
  Normal   Pulling    4m6s (x2 over 7m41s)  kubelet            Pulling image "rancher/mirrored-flannelcni-flannel-cni-plugin:v1.0.1"
  Normal   Pulled     2m25s                 kubelet            Successfully pulled image "rancher/mirrored-flannelcni-flannel-cni-plugin:v1.0.1" in 1m40.880912888s
  Normal   Created    2m25s                 kubelet            Created container install-cni-plugin
  Normal   Started    2m25s                 kubelet            Started container install-cni-plugin
  Normal   Pulling    2m25s                 kubelet            Pulling image "rancher/mirrored-flannelcni-flannel:v0.16.3"
```

可以看到,此时该Pod正在拉取镜像,但是失败了.在对应的节点上手动拉取镜像即可

```
root@k8s-node2:/home/nikolai# docker pull rancher/mirrored-flannelcni-flannel:v0.16.3
v0.16.3: Pulling from rancher/mirrored-flannelcni-flannel
5758d4e389a3: Already exists 
2550df1e0230: Already exists 
d05af167a36f: Already exists 
4d1ce2758b2c: Pull complete 
7ed6eda0612a: Pull complete 
980bc40a6f66: Pull complete 
54c6f5f413fd: Pull complete 
Digest: sha256:645f782e024986db3a3ce255d7cb004f851b03a7b0abecb145a8ce96659b05c5
Status: Downloaded newer image for rancher/mirrored-flannelcni-flannel:v0.16.3
docker.io/rancher/mirrored-flannelcni-flannel:v0.16.3
```

- 拉取完毕后,从k8s-master上查看结果

```
soap@k8s-master:~$ kubectl get pod --all-namespaces
NAMESPACE     NAME                                 READY   STATUS    RESTARTS   AGE
kube-system   coredns-55dffbd598-8rcfq             1/1     Running   0          6h48m
kube-system   coredns-55dffbd598-w4c4z             1/1     Running   0          6h48m
kube-system   etcd-k8s-master                      1/1     Running   0          6h49m
kube-system   kube-apiserver-k8s-master            1/1     Running   0          6h49m
kube-system   kube-controller-manager-k8s-master   1/1     Running   0          6h49m
kube-system   kube-flannel-ds-7w96q                1/1     Running   0          66m
kube-system   kube-flannel-ds-dxwnk                1/1     Running   0          61m
kube-system   kube-flannel-ds-tx967                1/1     Running   0          62m
kube-system   kube-proxy-5wvtm                     1/1     Running   0          62m
kube-system   kube-proxy-n52bm                     1/1     Running   0          61m
kube-system   kube-proxy-qzxsp                     1/1     Running   0          6h48m
kube-system   kube-scheduler-k8s-master            1/1     Running   0          6h49m
```

可以看到,所有节点现在均处于Running状态

## 附:查看kubelet的日志以确定报错信息的方式

查看报错信息:`journalctl -xeu kubelet`

此处以没有修改docker的cgroup驱动为例,查看日志

```
root@k8s-master:/home/soap# journalctl -xeu kubelet
Feb 09 10:04:25 k8s-master kubelet[5466]: E0209 10:04:25.388272    5466 server.go:302] "Failed to run kubelet" err="failed to run Kubelet: misconfiguration: kubelet cgroup driver: \"systemd\" is different from docker cgroup driver: \"cgroupfs\""
Feb 09 10:04:25 k8s-master systemd[1]: kubelet.service: Main process exited, code=exited, status=1/FAILURE
-- Subject: Unit process exited
-- Defined-By: systemd
-- Support: http://www.ubuntu.com/support
-- 
-- An ExecStart= process belonging to unit kubelet.service has exited.
-- 
-- The process' exit code is 'exited' and its exit status is 1.
Feb 09 10:04:25 k8s-master systemd[1]: kubelet.service: Failed with result 'exit-code'.
-- Subject: Unit failed
-- Defined-By: systemd
-- Support: http://www.ubuntu.com/support
-- 
-- The unit kubelet.service has entered the 'failed' state with result 'exit-code'.
Feb 09 10:04:35 k8s-master systemd[1]: kubelet.service: Scheduled restart job, restart counter is at 41.
-- Subject: Automatic restarting of a unit has been scheduled
-- Defined-By: systemd
-- Support: http://www.ubuntu.com/support
-- 
-- Automatic restarting of the unit kubelet.service has been scheduled, as the result for
-- the configured Restart= setting for the unit.
Feb 09 10:04:35 k8s-master systemd[1]: Stopped kubelet: The Kubernetes Node Agent.
-- Subject: A stop job for unit kubelet.service has finished
-- Defined-By: systemd
-- Support: http://www.ubuntu.com/support
-- 
-- A stop job for unit kubelet.service has finished.
-- 
-- The job identifier is 4472 and the job result is done.
Feb 09 10:04:35 k8s-master systemd[1]: Started kubelet: The Kubernetes Node Agent.
-- Subject: A start job for unit kubelet.service has finished successfully
-- Defined-By: systemd
-- Support: http://www.ubuntu.com/support
-- 
-- A start job for unit kubelet.service has finished successfully.
-- 
-- The job identifier is 4472.
```