---
title: '使用Kubeadm更新集群'
date: 2020-05-20T19:17:26+08:00
description: ''
draft: false
tags: ['kubernetes', 'kubeadm']
categories: ['kubernetes']
---

前面我们已经通过 [kubeadm 安装了一套集群](/2019/11/kubeadm-安装单master集群/)

当时安装的版本为 v1.16.3，现在版本已经更新到 1.18.x 了，所以我们需要对它进行升级

> k8s 不支持跨多个大版本升级，所以我们如果想要升级到 1.18.x，就得先升级到 1.17.x 版本

升级步骤参考官方文档 [Upgrading kubeadm clusters](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)

## 查看可升级的版本

如下可以看到，当前 1.17.5 为最新可升级版本

```shell
$ yum list --showduplicates kubeadm --disableexcludes=kubernetes | grep 1.17
kubeadm.x86_64                       1.17.0-0                        kubernetes
kubeadm.x86_64                       1.17.1-0                        kubernetes
kubeadm.x86_64                       1.17.2-0                        kubernetes
kubeadm.x86_64                       1.17.3-0                        kubernetes
kubeadm.x86_64                       1.17.4-0                        kubernetes
kubeadm.x86_64                       1.17.5-0                        kubernetes
```

## 升级 master

### 安装 1.17.5 版本 kubeadm

```shell
$ yum install -y kubeadm-1.17.5-0 --disableexcludes=kubernetes
$ kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"17", GitVersion:"v1.17.5", GitCommit:"e0fccafd69541e3750d460ba0f9743b90336f24f", GitTreeState:"clean", BuildDate:"2020-04-16T11:41:38Z", GoVersion:"go1.13.9", Compiler:"gc", Platform:"linux/amd64"}
```

### 给 master 节点打上污点并驱逐所有 pod

```shell
$ kubectl drain k8s-master --ignore-daemonsets --delete-local-data
node/k8s-master already cordoned
node/k8s-master evicted
```

### 升级前的检查

```shell
$ kubeadm upgrade plan
[upgrade/config] Making sure the configuration is correct:
[upgrade/config] Reading configuration from the cluster...
[upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[preflight] Running pre-flight checks.
[preflight] Some fatal errors occurred:
	[ERROR CoreDNSUnsupportedPlugins]: there are unsupported plugins in the CoreDNS Corefile
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
To see the stack trace of this error execute with --v=5 or higher
```

如上出现警告信息 `[ERROR CoreDNSUnsupportedPlugins]: there are unsupported plugins in the CoreDNS Corefile`

通过如下命令查看 CoreDNS 的配置文件，如果确认没有问题可以通过参数 `--ignore-preflight-errors=CoreDNSUnsupportedPlugins`忽略，github 上也有相关[issue](https://github.com/kubernetes/kubernetes/issues/82889)

```shell
$ kubectl get configmap -n kube-system coredns -oyaml
```

此处选择忽略再次重试

```shell
$ kubeadm upgrade plan --ignore-preflight-errors=CoreDNSUnsupportedPlugins
[upgrade/config] Making sure the configuration is correct:
[upgrade/config] Reading configuration from the cluster...
[upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[preflight] Running pre-flight checks.
	[WARNING CoreDNSUnsupportedPlugins]: there are unsupported plugins in the CoreDNS Corefile
[upgrade] Making sure the cluster is healthy:
[upgrade] Fetching available versions to upgrade to
[upgrade/versions] Cluster version: v1.16.3
[upgrade/versions] kubeadm version: v1.17.5
I0520 19:40:07.623240    6896 version.go:251] remote version is much newer: v1.18.2; falling back to: stable-1.17
[upgrade/versions] Latest stable version: v1.17.5
[upgrade/versions] Latest version in the v1.16 series: v1.16.9

Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   CURRENT       AVAILABLE
Kubelet     6 x v1.16.3   v1.16.9

Upgrade to the latest version in the v1.16 series:

COMPONENT            CURRENT   AVAILABLE
API Server           v1.16.3   v1.16.9
Controller Manager   v1.16.3   v1.16.9
Scheduler            v1.16.3   v1.16.9
Kube Proxy           v1.16.3   v1.16.9
CoreDNS              1.6.2     1.6.5
Etcd                 3.3.15    3.3.17-0

You can now apply the upgrade by executing the following command:

	kubeadm upgrade apply v1.16.9

_____________________________________________________________________

Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   CURRENT       AVAILABLE
Kubelet     6 x v1.16.3   v1.17.5

Upgrade to the latest stable version:

COMPONENT            CURRENT   AVAILABLE
API Server           v1.16.3   v1.17.5
Controller Manager   v1.16.3   v1.17.5
Scheduler            v1.16.3   v1.17.5
Kube Proxy           v1.16.3   v1.17.5
CoreDNS              1.6.2     1.6.5
Etcd                 3.3.15    3.4.3-0

You can now apply the upgrade by executing the following command:

	kubeadm upgrade apply v1.17.5

_____________________________________________________________________
```

### 执行升级

```shell
$ kubeadm upgrade apply v1.17.5 --ignore-preflight-errors=CoreDNSUnsupportedPlugins
[upgrade/config] Making sure the configuration is correct:
[upgrade/config] Reading configuration from the cluster...
[upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[preflight] Running pre-flight checks.
	[WARNING CoreDNSUnsupportedPlugins]: there are unsupported plugins in the CoreDNS Corefile
[upgrade] Making sure the cluster is healthy:
[upgrade/version] You have chosen to change the cluster version to "v1.17.5"
[upgrade/versions] Cluster version: v1.16.3
[upgrade/versions] kubeadm version: v1.17.5
[upgrade/confirm] Are you sure you want to proceed with the upgrade? [y/N]: y
[upgrade/prepull] Will prepull images for components [kube-apiserver kube-controller-manager kube-scheduler etcd]
[upgrade/prepull] Prepulling image for component kube-apiserver.
[upgrade/prepull] Prepulling image for component kube-controller-manager.
[upgrade/prepull] Prepulling image for component etcd.
[upgrade/prepull] Prepulling image for component kube-scheduler.
[apiclient] Found 1 Pods for label selector k8s-app=upgrade-prepull-kube-controller-manager
[apiclient] Found 0 Pods for label selector k8s-app=upgrade-prepull-kube-scheduler
[apiclient] Found 0 Pods for label selector k8s-app=upgrade-prepull-etcd
[apiclient] Found 1 Pods for label selector k8s-app=upgrade-prepull-kube-apiserver
[apiclient] Found 1 Pods for label selector k8s-app=upgrade-prepull-kube-scheduler
[apiclient] Found 1 Pods for label selector k8s-app=upgrade-prepull-etcd
[upgrade/prepull] Prepulled image for component kube-apiserver.
[upgrade/prepull] Prepulled image for component kube-controller-manager.
[upgrade/prepull] Prepulled image for component etcd.
[upgrade/prepull] Prepulled image for component kube-scheduler.
[upgrade/prepull] Successfully prepulled the images for all the control plane components
[upgrade/apply] Upgrading your Static Pod-hosted control plane to version "v1.17.5"...
Static pod: kube-apiserver-k8s-master hash: e541261a584becf7ccd0ebc447ad6965
Static pod: kube-controller-manager-k8s-master hash: 83e58d3548072b341a6faa0bbf8427dc
Static pod: kube-scheduler-k8s-master hash: e8486b59c2c8408b07026a560746b02c
[upgrade/etcd] Upgrading to TLS for etcd
Static pod: etcd-k8s-master hash: 7d86fcdbd639f4fd16605f99fbffa6e4
[upgrade/staticpods] Preparing for "etcd" upgrade
[upgrade/staticpods] Renewing etcd-server certificate
[upgrade/staticpods] Renewing etcd-peer certificate
[upgrade/staticpods] Renewing etcd-healthcheck-client certificate
[upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/etcd.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2020-05-20-19-53-29/etcd.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
Static pod: etcd-k8s-master hash: 7d86fcdbd639f4fd16605f99fbffa6e4
Static pod: etcd-k8s-master hash: 0f55aa1d6eb94a8c4eca4c5f027fb643
[apiclient] Found 1 Pods for label selector component=etcd
[upgrade/staticpods] Component "etcd" upgraded successfully!
[upgrade/etcd] Waiting for etcd to become available
[upgrade/staticpods] Writing new Static Pod manifests to "/etc/kubernetes/tmp/kubeadm-upgraded-manifests198557268"
W0520 19:54:11.882414   28625 manifests.go:214] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[upgrade/staticpods] Preparing for "kube-apiserver" upgrade
[upgrade/staticpods] Renewing apiserver certificate
[upgrade/staticpods] Renewing apiserver-kubelet-client certificate
[upgrade/staticpods] Renewing front-proxy-client certificate
[upgrade/staticpods] Renewing apiserver-etcd-client certificate
[upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-apiserver.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2020-05-20-19-53-29/kube-apiserver.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
Static pod: kube-apiserver-k8s-master hash: e541261a584becf7ccd0ebc447ad6965
Static pod: kube-apiserver-k8s-master hash: e541261a584becf7ccd0ebc447ad6965
Static pod: kube-apiserver-k8s-master hash: e541261a584becf7ccd0ebc447ad6965
Static pod: kube-apiserver-k8s-master hash: e541261a584becf7ccd0ebc447ad6965
Static pod: kube-apiserver-k8s-master hash: e541261a584becf7ccd0ebc447ad6965
Static pod: kube-apiserver-k8s-master hash: e541261a584becf7ccd0ebc447ad6965
Static pod: kube-apiserver-k8s-master hash: e541261a584becf7ccd0ebc447ad6965
Static pod: kube-apiserver-k8s-master hash: e541261a584becf7ccd0ebc447ad6965
Static pod: kube-apiserver-k8s-master hash: e541261a584becf7ccd0ebc447ad6965
Static pod: kube-apiserver-k8s-master hash: e541261a584becf7ccd0ebc447ad6965
Static pod: kube-apiserver-k8s-master hash: e541261a584becf7ccd0ebc447ad6965
Static pod: kube-apiserver-k8s-master hash: e541261a584becf7ccd0ebc447ad6965
Static pod: kube-apiserver-k8s-master hash: cb0329ceca81a08d2613d2eafe96927f
[apiclient] Found 1 Pods for label selector component=kube-apiserver
[upgrade/staticpods] Component "kube-apiserver" upgraded successfully!
[upgrade/staticpods] Preparing for "kube-controller-manager" upgrade
[upgrade/staticpods] Renewing controller-manager.conf certificate
[upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-controller-manager.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2020-05-20-19-53-29/kube-controller-manager.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
Static pod: kube-controller-manager-k8s-master hash: 83e58d3548072b341a6faa0bbf8427dc
Static pod: kube-controller-manager-k8s-master hash: 57f625031b63d8f30d971821233b1d59
[apiclient] Found 1 Pods for label selector component=kube-controller-manager
[upgrade/staticpods] Component "kube-controller-manager" upgraded successfully!
[upgrade/staticpods] Preparing for "kube-scheduler" upgrade
[upgrade/staticpods] Renewing scheduler.conf certificate
[upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-scheduler.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2020-05-20-19-53-29/kube-scheduler.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
Static pod: kube-scheduler-k8s-master hash: e8486b59c2c8408b07026a560746b02c
Static pod: kube-scheduler-k8s-master hash: 9d8e79ec2b72cfad2458fb03205bc653
[apiclient] Found 1 Pods for label selector component=kube-scheduler
[upgrade/staticpods] Component "kube-scheduler" upgraded successfully!
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.17" in namespace kube-system with the configuration for the kubelets in the cluster
[kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.17" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[addons]: Migrating CoreDNS Corefile
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.17.5". Enjoy!

[upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
```

### 解除 master 节点污点

```shell
$ kubectl uncordon k8s-master
node/k8s-master uncordoned
```

### 升级其他 master 节点

如果是高可用版本还需要升级其他的 master 节点

> 本次集群只有单个 master,以下操作纯做记录

```shell
$ yum install -y kubeadm-1.17.5-0 --disableexcludes=kubernetes
$ kubeadm upgrade node
$ kubeadm upgrade apply
```

### 升级所有 master 节点的 kubectl 与 kubelet

```shell
$ yum install -y kubectl-1.17.5-0 kubelet-1.17.5-0 --disableexcludes=kubernetes
```

### 重启 master 节点 kubelet 服务

```shell
$ systemctl daemon-reload
$ systemctl restart kubelet
$ systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
  Drop-In: /usr/lib/systemd/system/kubelet.service.d
           └─10-kubeadm.conf
   Active: active (running) since Wed 2020-05-20 20:11:47 CST; 12s ago
     Docs: https://kubernetes.io/docs/
 Main PID: 2904 (kubelet)
    Tasks: 15
   Memory: 25.9M
   CGroup: /system.slice/kubelet.service
           └─2904 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --cgroup-driver=systemd --network-plugin=cni --pod-infra...

May 20 20:11:47 k8s-master kubelet[2904]: I0520 20:11:47.297946    2904 docker_service.go:260] Docker Info: &{ID:BI53:OQC7:QV6C:SP3N:FBTT:N5ST:MK3I:CGLJ:X2JX:CJOH:CBEE:4GG2 Containers:30 ContainersRunning:22 Containe...rts d_type true]
May 20 20:11:47 k8s-master kubelet[2904]: I0520 20:11:47.298023    2904 docker_service.go:273] Setting cgroupDriver to systemd
May 20 20:11:47 k8s-master kubelet[2904]: I0520 20:11:47.305372    2904 remote_runtime.go:59] parsed scheme: ""
May 20 20:11:47 k8s-master kubelet[2904]: I0520 20:11:47.305387    2904 remote_runtime.go:59] scheme "" not registered, fallback to default scheme
May 20 20:11:47 k8s-master kubelet[2904]: I0520 20:11:47.305408    2904 passthrough.go:48] ccResolverWrapper: sending update to cc: {[{/var/run/dockershim.sock 0  <nil>}] <nil>}
May 20 20:11:47 k8s-master kubelet[2904]: I0520 20:11:47.305415    2904 clientconn.go:577] ClientConn switching balancer to "pick_first"
May 20 20:11:47 k8s-master kubelet[2904]: I0520 20:11:47.305433    2904 remote_image.go:50] parsed scheme: ""
May 20 20:11:47 k8s-master kubelet[2904]: I0520 20:11:47.305438    2904 remote_image.go:50] scheme "" not registered, fallback to default scheme
May 20 20:11:47 k8s-master kubelet[2904]: I0520 20:11:47.305446    2904 passthrough.go:48] ccResolverWrapper: sending update to cc: {[{/var/run/dockershim.sock 0  <nil>}] <nil>}
May 20 20:11:47 k8s-master kubelet[2904]: I0520 20:11:47.305450    2904 clientconn.go:577] ClientConn switching balancer to "pick_first"
Hint: Some lines were ellipsized, use -l to show in full.
```

### 确认 master 版本

```shell
$ kubectl get node k8s-master
NAME         STATUS   ROLES    AGE    VERSION
k8s-master   Ready    master   180d   v1.17.5
```

## 升级 worker 节点

> 非特别说明，都是在 worker 节点上操作

需要在所有 woker 节点上进行操作, 此处以 k8s-node01 节点为例

### 更新 kubeadm

```shell
yum install -y kubeadm-1.17.5-0 --disableexcludes=kubernetes
```

### 给 worker 节点打上污点并驱逐所有 pod

> 在 Master 节点上执行

```shell
$ kubectl drain k8s-node01 --ignore-daemonsets --delete-local-data
node/k8s-node01 cordoned
evicting pod "tekton-pipelines-webhook-694dc67fbb-kjbqv"
...
pod/tekton-pipelines-webhook-694dc67fbb-kjbqv evicted
...
node/k8s-node01 evicted
```

### 升级 woker 节点 kubelet 配置

```shell
$ kubeadm upgrade node
[upgrade] Reading configuration from the cluster...
[upgrade] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[upgrade] Skipping phase. Not a control plane node.
[kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.17" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[upgrade] The configuration for this node was successfully updated!
[upgrade] Now you should go ahead and upgrade the kubelet package using your package manager.
```

### 升级 woker 节点 kubelet 与 kubectl 版本

```shell
yum install -y kubectl-1.17.5-0 kubelet-1.17.5-0 --disableexcludes=kubernetes
```

### 重启 kubelet 服务

```shell
$ systemctl daemon-reload
$ systemctl restart kubelet
$ systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
  Drop-In: /usr/lib/systemd/system/kubelet.service.d
           └─10-kubeadm.conf
   Active: active (running) since Wed 2020-05-20 20:27:07 CST; 21s ago
     Docs: https://kubernetes.io/docs/
 Main PID: 4202 (kubelet)
    Tasks: 18
   Memory: 38.0M
   CGroup: /system.slice/kubelet.service
           └─4202 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/et...

May 20 20:27:28 k8s-node01 kubelet[4202]: I0520 20:27:28.637150    4202 kuberuntime_manager.go:981] updating....0/24
May 20 20:27:28 k8s-node01 kubelet[4202]: I0520 20:27:28.637283    4202 docker_service.go:355] docker cri re...4,},}
May 20 20:27:28 k8s-node01 kubelet[4202]: I0520 20:27:28.637397    4202 kubelet_network.go:77] Setting Pod C....0/24
May 20 20:27:28 k8s-node01 kubelet[4202]: I0520 20:27:28.638934    4202 kubelet_node_status.go:70] Attemptin...ode01
May 20 20:27:28 k8s-node01 kubelet[4202]: I0520 20:27:28.653131    4202 kubelet_node_status.go:112] Node k8s...tered
May 20 20:27:28 k8s-node01 kubelet[4202]: I0520 20:27:28.653222    4202 kubelet_node_status.go:73] Successfu...ode01
May 20 20:27:28 k8s-node01 kubelet[4202]: E0520 20:27:28.657774    4202 kubelet.go:1844] skipping pod synchr...d yet
May 20 20:27:28 k8s-node01 kubelet[4202]: I0520 20:27:28.659901    4202 setters.go:537] Node became not ready: {T...
May 20 20:27:28 k8s-node01 kubelet[4202]: W0520 20:27:28.842260    4202 docker_sandbox.go:394] failed to read pod...
May 20 20:27:28 k8s-node01 kubelet[4202]: E0520 20:27:28.857932    4202 kubelet.go:1844] skipping pod synchr...d yet
Hint: Some lines were ellipsized, use -l to show in full.
```

### 解除 worker 节点污点

> 在 Master 节点上操作

```shell
$ kubectl uncordon k8s-node01
node/k8s-node01 uncordoned
```

### 验证 worker 节点版本

```shell
$ kubectl get nodes
NAME         STATUS   ROLES    AGE    VERSION
k8s-master   Ready    master   180d   v1.17.2
k8s-node01   Ready    <none>   180d   v1.17.5
k8s-node02   Ready    <none>   180d   v1.17.5
k8s-node03   Ready    <none>   180d   v1.17.5
k8s-node04   Ready    <none>   180d   v1.17.5
k8s-node05   Ready    <none>   180d   v1.17.5
k8s-node06   Ready    <none>   180d   v1.17.5
```

如上，再以相同的方法更新到 1.18.x 版本即可

## 验证集群状态

### 检查系统组件

```shell
$ kubectl get pods -n kube-system
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-7d4d547dd6-59xqn   1/1     Running   0          17m
calico-node-8d8bm                          1/1     Running   0          19h
calico-node-hlv2h                          1/1     Running   0          19h
calico-node-nbt4b                          1/1     Running   0          19h
calico-node-p4s45                          1/1     Running   4          19h
calico-node-srb88                          1/1     Running   0          19h
calico-node-vfpfv                          1/1     Running   0          19h
coredns-546565776c-54gxr                   1/1     Running   0          12m
coredns-546565776c-zqhcb                   1/1     Running   0          20m
etcd-k8s-master                            1/1     Running   0          20m
istio-cni-node-42ngc                       2/2     Running   0          10h
istio-cni-node-6qngn                       2/2     Running   0          10h
istio-cni-node-8pzlj                       2/2     Running   0          10h
istio-cni-node-9p47g                       2/2     Running   0          10h
istio-cni-node-hqp75                       2/2     Running   0          10h
istio-cni-node-vmnbp                       2/2     Running   10         10h
kube-apiserver-k8s-master                  1/1     Running   0          20m
kube-controller-manager-k8s-master         1/1     Running   0          20m
kube-proxy-86845                           1/1     Running   0          19m
kube-proxy-dnccs                           1/1     Running   0          20m
kube-proxy-grn6t                           1/1     Running   0          19m
kube-proxy-ltwdr                           1/1     Running   0          19m
kube-proxy-p5mn8                           1/1     Running   0          19m
kube-proxy-qrnb7                           1/1     Running   0          19m
kube-scheduler-k8s-master                  1/1     Running   0          20m
metrics-server-747dfbd64b-5nnlz            1/1     Running   0          23s
```

### 测试 DNS 功能

```shell
$ dig www.baidu.com @10.96.0.10 +short
www.a.shifen.com.
112.80.248.75
112.80.248.76
$ dig kubernetes.default.svc.cluster.local @10.96.0.10 +short
10.96.0.1
$ kubectl run -n kube-ops -i --rm --restart=Never dummy --image=busybox:1.28 -- nslookup kubernetes.default
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
Name:      kubernetes.default
Address 1: 10.96.0.1 kubernetes.default.svc.cluster.local
pod "dummy" deleted
```

### 测试访问集群中的服务

```shell
$ kubectl get service -n default
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
whoami       ClusterIP   10.96.179.57   <none>        80/TCP    21d
$ curl http://10.96.179.57
Hostname: whoami-5b4bb9c787-xz87j
IP: 127.0.0.1
IP: 172.16.217.76
RemoteAddr: 127.0.0.1:33338
GET / HTTP/1.1
Host: 10.96.179.57
User-Agent: curl/7.29.0
Accept: */*
$ kubectl run curl -n kube-ops -it --image curlimages/curl --restart Never --rm -- curl whoami.default
Hostname: whoami-5b4bb9c787-xz87j
IP: 127.0.0.1
IP: 172.16.217.76
RemoteAddr: 127.0.0.1:35506
GET / HTTP/1.1
Host: whoami.default
User-Agent: curl/7.70.0-DEV
Accept: */*
pod "curl" deleted
```
