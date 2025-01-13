---
title: 'kubeadm 安装单master集群'
date: 2019-11-18T19:12:02+08:00
description: ''
draft: false
tags: ['kubernetes']
categories: ['kubernetes']
---

## 环境准备

### 系统及软件版本

> kubernetes version: v1.16.3

| 组件    | 版本    | 说明                |
| ------- | ------- | ------------------- |
| Centos  | 7.7     | 操作系统            |
| kubeadm | v1.16.3 | 集群部署工具        |
| etcd    | 3.3.15  | 存储数据库          |
| calico  | v3.14.0 | CNI 网络插件        |
| docker  | 18.09.9 | CRI Runtime         |
| metallb | v0.8.3  | 穷人版 LoadBalancer |

### 主机列表

| hostname   | IP Address    | components                                                                        |
| ---------- | ------------- | --------------------------------------------------------------------------------- |
| k8s-master | 10.64.144.100 | kube-apiserver,kube-controller-manager,kube-scheduler,etcd,kubelet,flannel,docker |
| k8s-node01 | 10.64.144.101 | kube-proxy,kubelet,flannel,docker                                                 |
| k8s-node02 | 10.64.144.102 | kube-proxy,kubelet,flannel,docker                                                 |
| k8s-node03 | 10.64.144.103 | kube-proxy,kubelet,flannel,docker                                                 |
| k8s-node04 | 10.64.144.104 | kube-proxy,kubelet,flannel,docker                                                 |
| k8s-node05 | 10.64.144.105 | kube-proxy,kubelet,flannel,docker                                                 |
| k8s-node06 | 10.64.144.106 | kube-proxy,kubelet,flannel,docker                                                 |

### 网络规划

| 功能         | 网段                        |
| ------------ | --------------------------- |
| Pod 网络     | 172.16.0.0/16               |
| Service 网络 | 10.96.0.0/12                |
| DNS 地址     | 10.96.0.10                  |
| LoadBalance  | 10.64.144.150~10.64.144.200 |

## 系统初始化

> 所有节点上都需要操作

### 配置 hosts 同步

```shell
$ cat > /etc/hosts <<EOF
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
10.64.144.100   k8s-master
10.64.144.101   k8s-node01
10.64.144.102   k8s-node02
10.64.144.103   k8s-node03
10.64.144.104   k8s-node04
10.64.144.105   k8s-node05
10.64.144.106   k8s-node06
EOF
```

### 设置主机名

```shell
hostnamectl set-hostname k8s-master # 分别设置所有节点的主机名
```

### 更新系统

```shell
yum update -y
```

### 更新内核

安装稳定版主线内核

```shell
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
yum install -y https://www.elrepo.org/elrepo-release-7.el7.elrepo.noarch.rpm
yum install -y --enablerepo=elrepo-kernel kernel-ml kernel-ml-headers kernel-ml-devel
```

查看系统中已安装的内核版本

```shell
$ awk -F\' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg
0 : CentOS Linux (5.6.13-1.el7.elrepo.x86_64) 7 (Core)
1 : CentOS Linux (3.10.0-1062.4.1.el7.x86_64) 7 (Core)
2 : CentOS Linux (3.10.0-957.el7.x86_64) 7 (Core)
3 : CentOS Linux (0-rescue-4097349c950a4862a8fab2587d598af7) 7 (Core)
```

切换默认内核

> `0` 为上面列出的序号，比如我这里 `0` 代表 `5.6.13` 这个版本的内核

```shell
$ grub2-set-default 0
$ grub2-mkconfig -o /boot/grub2/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-5.6.13-1.el7.elrepo.x86_64
Found initrd image: /boot/initramfs-5.6.13-1.el7.elrepo.x86_64.img
Found linux image: /boot/vmlinuz-3.10.0-1062.4.1.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-1062.4.1.el7.x86_64.img
Found linux image: /boot/vmlinuz-3.10.0-957.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-957.el7.x86_64.img
Found linux image: /boot/vmlinuz-0-rescue-4097349c950a4862a8fab2587d598af7
Found initrd image: /boot/initramfs-0-rescue-4097349c950a4862a8fab2587d598af7.img
done
```

### 配置网卡驱动(可选)

> 实验环境中由于使用的是普通 PC，板载网卡为 realtek r8168/r8169, 这个网卡在 5.x 版本内核中由于缺少加载 realtek.ko 模块会造成无法正常识别到网卡的情况

```shell
$ mkinitrd --force --preload realtek /boot/initramfs-`uname -r`.img `uname -r`
$ lsinitrd /boot/initramfs-`uname -r`.img | grep realtek
Arguments: -f --add-drivers ' realtek'
...
-rwxr--r--   1 root     root       127880 Jan  7 16:21 usr/lib/modules/5.6.13-1.el7.elrepo.x86_64/kernel/drivers/net/ethernet/realtek/r8169.ko
-rwxr--r--   1 root     root        27360 Jan  7 16:21 usr/lib/modules/5.6.13-1.el7.elrepo.x86_64/kernel/drivers/net/phy/realtek.ko
```

确保有加载 `realtek.ko` 和 `r8169.ko` 模块

### 重启后验证内核版本

```shell
$ uname -r
5.6.13-1.el7.elrepo.x86_64
```

### 禁用 SELinux

如果不禁用 selinux，挂载目录会出现 `Permission deined` 错误

```shell
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config  # 永久关闭
setenforce 0  # 临时关闭
```

### 禁用 swap

```shell
swapoff -a
sed -i -r 's/(.+ swap .*)/# \1/' /etc/fstab
sysctl -w vm.swappiness=0
cat > /etc/sysctl.d/swap.conf <<EOF
vm.swappiness=0
EOF
sysctl -p /etc/sysctl.d/swap.conf
```

### 禁用不需要的服务

如果选择不关闭`firewalld`，请参考官方文档中放行相应的端口 [check-required-ports](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#check-required-ports)

最少化安装的 Centos 中貌似没有安装 dnsmasq，如果有安装则需要将其禁用，否则它会把节点上的 DNS server 设置为 127.0.0.1,将会导致 Docker 无法解析域名

```shell
systemctl disable --now firewalld NetworkManager dnsmasq
```

### 安装必要工具

kube-proxy 的 ipvs 模式下依赖`ipvsadm`,`ipset`用于配置及管理规则

`kubectl port-forward`命令依赖`socat`做端口转发

```shell
yum install -y ipvsadm ipset sysstat conntrack libseccomp socat wget git jq curl
```

### 优化系统内核

```shell
$ cat > /etc/sysctl.d/k8s.conf <<EOF
# https://github.com/moby/moby/issues/31208
# ipvsadm -l --timout
# 修复ipvs模式下长连接timeout问题 小于900即可
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 10
# 关闭ipv6
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
net.ipv4.neigh.default.gc_stale_time = 120
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.default.rp_filter = 0
net.ipv4.conf.default.arp_announce = 2
net.ipv4.conf.lo.arp_announce = 2
net.ipv4.conf.all.arp_announce = 2
# 开启路由转发
net.ipv4.ip_forward = 1
net.ipv4.tcp_max_tw_buckets = 5000
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 1024
net.ipv4.tcp_synack_retries = 2
# 开启 bridge-netfilter, 使用iptables规则可以作用于bridge模式
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-arptables = 1
net.netfilter.nf_conntrack_max = 2310720
fs.inotify.max_user_watches=89100
fs.may_detach_mounts = 1
fs.file-max = 52706963
fs.nr_open = 52706963
vm.overcommit_memory=1
vm.panic_on_oom=0
EOF
$ sysctl --system  # 使配置生效
```

### 开启 ipvs 支持

> 如果内核版本小于 `4.19`, 需要将 `nf_conntrack` 修改为 `nf_conntrack_ipv4`

```shell
# 临时生效
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack

# 永久生效
$ cat > /etc/sysconfig/modules/ipvs.modules <<'EOF'
#!/bin/sh
modules=(
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack_ipv4
br_netfilter
)
for mod in ${modules[@]}
do
  /sbin/modinfo -F filename ${mod} > /dev/null 2>&1
  if [ $? -eq 0 ]
  then
      /sbin/modprobe ${mod}
  fi
done
EOF
chmod +x /etc/sysconfig/modules/ipvs.modules

# 检查内核模块是否已经加载
$ lsmod | grep ip_vs
ip_vs_sh               12688  0
ip_vs_wrr              12697  0
ip_vs_rr               12600  4
ip_vs                 145497  10 ip_vs_rr,ip_vs_sh,ip_vs_wrr
nf_conntrack          139224  7 ip_vs,nf_nat,nf_nat_ipv4,xt_conntrack,nf_nat_masquerade_ipv4,nf_conntrack_netlink,nf_conntrack
libcrc32c              12644  4 xfs,ip_vs,nf_nat,nf_conntrack
```

### 时间同步

所有节点配置为同一时区，并使用 `chrony` 同步节点的时间

```shell
$ ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
$ echo "Asia/Shanghai" > /etc/timezone
$ yum install chrony -y
$ cat > /etc/chrony.conf <<EOF
server ntp.aliyun.com iburst
stratumweight 0
driftfile /var/lib/chrony/drift
rtcsync
makestep 10 3
bindcmdaddress 127.0.0.1
bindcmdaddress ::1
keyfile /etc/chrony.keys
commandkey 1
generatecommandkey
logchange 0.5
logdir /var/log/chrony
EOF
$ systemctl enable --now chronyd
```

### 运行 docker 检测脚本

docker 官方提供了脚本用于检查内核相关配置

{{< highlight shell "linenos=table,hl_lines=9" >}}
\$ curl -fsSL <https://raw.githubusercontent.com/moby/moby/master/contrib/check-config.sh> | bash -s
info: reading kernel config from /boot/config-3.10.0-1062.4.3.el7.x86_64 ...

Generally Necessary:
...

Optional Features:

- CONFIG_USER_NS: enabled
  (RHEL7/CentOS7: User namespaces disabled; add 'user_namespace.enable=1' to boot command line)
- CONFIG_SECCOMP: enabled
  ...
  {{< / highlight >}}

如上高亮行所示，开启 user_namespace

```shell
$ grubby --args="user_namespace.enable=1" --update-kernel="$(grubby --default-kernel)"
$ grep user_namespace /etc/grub2.cfg
        linux16 /vmlinuz-5.6.13-1.el7.elrepo.x86_64 root=/dev/mapper/centos-root ro crashkernel=auto rd.lvm.lv=centos/root rhgb quiet user_namespace.enable=1
```

### 重启系统

重启系统以使配置生效

```shell
systemctl reboot
```

## 安装 Docker

> 所有节点上均需配置

虽然 Kubernetes 支持多种容器运行环境，如 `Docker`, `Containerd`, `CRI-O` 等，但目前 `Docker` 还是主流

关于容器运行环境可以参考官方文档 [Container Runntimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)

支持的 Docker 版本列表请查看官方 [CHANGELOG](https://github.com/kubernetes/kubernetes/blob/v1.16.3/CHANGELOG-1.16.md) 搜索 `The list of validated docker versions` 关键字

### 安装

官方推荐使用 `18.06.2`, 我这里使用的是 `18.09.9`

Docker 官方的 yum 源在国内可能会访问不到，这里使用了阿里云的镜像源

```shell
yum install -y yum-utils device-mapper-persistent-data lvm2
export VERSION=18.09
curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
```

### 修改配置文件

主要修改以下几个参数:

- `native.cgroupdriver`: 官方推荐使用 `systemd` 来管理 cgroup,且必须与 kubelet 的`--cgroup-driver`参数一致
- `storage-driver`: 存储驱动，`overlay2` 会比其他驱动更好的性能。如果文件系统为`XFS`，请确保`ftype=1`(可使用 `xfs_info` 查看),否则 docker 会无法启动，报错信息`Error starting daemon: error initializing graphdriver: overlay2: the backing xfs filesystem is formatted without d_type support, which leads to incorrect behavior. Reformat the filesystem with ftype=1 to enable d_type support. Backing filesystems without d_type support are not supported.`
- `registry-mirrors`: 加速从 dockerhub 拉取镜像

```shell
mkdir /etc/docker
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors": ["https://8jg6za8m.mirror.aliyuncs.com"],
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}
EOF
```

### 启动并检查配置是否生效

```shell
$ systemctl enable --now docker
Created symlink from /etc/systemd/system/multi-user.target.wants/docker.service to /usr/lib/systemd/system/docker.service.
$ docker version
Client:
 Version:           18.09.9
 API version:       1.39
 Go version:        go1.11.13
 Git commit:        039a7df9ba
 Built:             Wed Sep  4 16:51:21 2019
 OS/Arch:           linux/amd64
 Experimental:      false

Server: Docker Engine - Community
 Engine:
  Version:          18.09.9
  API version:      1.39 (minimum version 1.12)
  Go version:       go1.11.13
  Git commit:       039a7df
  Built:            Wed Sep  4 16:22:32 2019
  OS/Arch:          linux/amd64
  Experimental:     false
$ docker info
Containers: 0
 Running: 0
 Paused: 0
 Stopped: 0
Images: 0
Server Version: 18.09.9
Storage Driver: overlay2
 Backing Filesystem: xfs
 Supports d_type: true
 Native Overlay Diff: true
Logging Driver: json-file
Cgroup Driver: systemd
Plugins:
 Volume: local
 Network: bridge host macvlan null overlay
 Log: awslogs fluentd gcplogs gelf journald json-file local logentries splunk syslog
Swarm: inactive
Runtimes: runc
Default Runtime: runc
Init Binary: docker-init
containerd version: b34a5c8af56e510852c35414db4c1f4fa6172339
runc version: 3e425f80a8c931f88e6d94a8c831b9d5aa481657
init version: fec3683
Security Options:
 seccomp
  Profile: default
Kernel Version: 5.6.13-1.el7.elrepo.x86_64
Operating System: CentOS Linux 7 (Core)
OSType: linux
Architecture: x86_64
CPUs: 4
Total Memory: 15.39GiB
Name: k8s-master
ID: BI53:OQC7:QV6C:SP3N:FBTT:N5ST:MK3I:CGLJ:X2JX:CJOH:CBEE:4GG2
Docker Root Dir: /var/lib/docker
Debug Mode (client): false
Debug Mode (server): false
Registry: https://index.docker.io/v1/
Labels:
Experimental: false
Insecure Registries:
 127.0.0.0/8
Registry Mirrors:
 https://8jg6za8m.mirror.aliyuncs.com/
Live Restore Enabled: false
Product License: Community Engine
```

## 安装 kubernetes 相关工具

> 所有节点上操作

### 添加阿里云 YUM 源

默认官方的源在国内环境下可能会访问不到

```shell
cat > /etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

### 查看支持的版本列表

```shell
$ yum --disablerepo="*" --enablerepo="kubernetes" list available --showduplicates  | grep "1.16"
kubeadm.x86_64                       1.16.0-0                         kubernetes
kubeadm.x86_64                       1.16.1-0                         kubernetes
kubeadm.x86_64                       1.16.2-0                         kubernetes
kubeadm.x86_64                       1.16.3-0                         kubernetes
kubectl.x86_64                       1.16.0-0                         kubernetes
kubectl.x86_64                       1.16.1-0                         kubernetes
kubectl.x86_64                       1.16.2-0                         kubernetes
kubectl.x86_64                       1.16.3-0                         kubernetes
kubelet.x86_64                       1.16.0-0                         kubernetes
kubelet.x86_64                       1.16.1-0                         kubernetes
kubelet.x86_64                       1.16.2-0                         kubernetes
kubelet.x86_64                       1.16.3-0                         kubernetes
```

### 安装 kubeadm、kubectl、kubelet 工具

> 本次安装 `1.16.3` 版本

worker 节点可以不安装 kubectl

```shell
yum install kubeadm-1.16.3 kubectl-1.16.3 kubelet-1.16.3 -y
```

### 添加 kubelet 开机自启动

```shell
$ systemctl enable --now kubelet
Created symlink from /etc/systemd/system/multi-user.target.wants/kubelet.service to /usr/lib/systemd/system/kubelet.service.
$ systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
  Drop-In: /usr/lib/systemd/system/kubelet.service.d
           └─10-kubeadm.conf
   Active: activating (auto-restart) (Result: exit-code) since Thu 2019-11-21 10:27:34 CST; 898ms ago
     Docs: https://kubernetes.io/docs/
  Process: 3947 ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS (code=exited, status=255)
 Main PID: 3947 (code=exited, status=255)

Nov 21 10:27:34 k8s-master systemd[1]: Unit kubelet.service entered failed state.
Nov 21 10:27:34 k8s-master systemd[1]: kubelet.service failed.
```

如上查看服务状态你会发现启动失败，不过没关系，这里可以暂时忽略，因为我们还没开始初始化集群，所以还没有生成`kubelet`的配置文件

如果 enable 时提示 `Unit kubelet.service could not be found.`，执行`systemctl daemon-reload`重载配置就可以了

## 初始化集群

> 在 master 上操作

### 生成 kubeadm 配置文件

生成默认配置

```shell
kubeadm config print init-defaults --component-configs KubeProxyConfiguration,KubeletConfiguration > kubeadm-config.yaml
```

### 修改配置

主要改动如下:

{{< highlight yaml "linenos=table,hl_lines=5 12 32 34 37-38 49 72 97-100" >}}
apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:

- groups:
  - system:bootstrappers:kubeadm:default-node-token
    token: d7cb0e.605fb37fed0895d3 # 使用命令 echo "$(openssl rand -hex 3).$(openssl rand -hex 8)" 生成
    ttl: 24h0m0s
    usages:
  - signing
  - authentication
    kind: InitConfiguration
    localAPIEndpoint:
    advertiseAddress: 10.64.144.100 # Master 节点的本机 IP 地址
    bindPort: 6443
    nodeRegistration:
    criSocket: /var/run/dockershim.sock
    name: k8s-master
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
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers # 由于默认的`k8s.gcr.io`在国内无法正常访问，这里使用阿里云的源代替
kind: ClusterConfiguration
kubernetesVersion: v1.16.3 # kubernetes 版本
networking:
dnsDomain: cluster.local
serviceSubnet: 10.96.0.0/12 # 集群 service 的网段
podSubnet: 172.16.0.0/16 # 集群 pod 网段
scheduler: {}

---

apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
clientConnection:
acceptContentTypes: ""
burst: 10
contentType: application/vnd.kubernetes.protobuf
kubeconfig: /var/lib/kube-proxy/kubeconfig.conf
qps: 5
clusterCIDR: "172.16.0.0/16" # 集群 pod 的网段，需要与后续部署的网络插件一致
configSyncPeriod: 15m0s
conntrack:
maxPerCore: 32768
min: 131072
tcpCloseWaitTimeout: 1h0m0s
tcpEstablishedTimeout: 24h0m0s
enableProfiling: false
healthzBindAddress: 0.0.0.0:10256
hostnameOverride: ""
iptables:
masqueradeAll: false
masqueradeBit: 14
minSyncPeriod: 0s
syncPeriod: 30s
ipvs:
excludeCIDRs: null
minSyncPeriod: 0s
scheduler: ""
strictARP: false
syncPeriod: 30s
kind: KubeProxyConfiguration
metricsBindAddress: 0.0.0.0:10249
mode: "ipvs" # Kube-Proxy 使用 ipvs 模式， 1.11 之后的版本默认使用 ipvs 代替了 iptables。如果节点上没有启用 ipvs 内核模块，会自动降级为 iptables
nodePortAddresses: null
oomScoreAdj: -999
portRange: ""
udpIdleTimeout: 250ms
winkernel:
enableDSR: false
networkName: ""
sourceVip: ""

---

address: 0.0.0.0
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
anonymous:
enabled: false
webhook:
cacheTTL: 2m0s
enabled: true
x509:
clientCAFile: /etc/kubernetes/pki/ca.crt
authorization:
mode: Webhook
webhook:
cacheAuthorizedTTL: 5m0s
cacheUnauthorizedTTL: 30s
cgroupDriver: systemd # cgroup 管理方式，需要与 docker 中的`native.cgroupdriver`一致，推荐使用 systemd
cgroupsPerQOS: true
clusterDNS:

- 10.96.0.10 # 集群中 CoreDNS 的 Service IP 地址，需要在上面配置`networking.serviceSubnet`子网中
  clusterDomain: cluster.local
  configMapAndSecretChangeDetectionStrategy: Watch
  containerLogMaxFiles: 5
  containerLogMaxSize: 10Mi
  contentType: application/vnd.kubernetes.protobuf
  cpuCFSQuota: true
  cpuCFSQuotaPeriod: 100ms
  cpuManagerPolicy: none
  cpuManagerReconcilePeriod: 10s
  enableControllerAttachDetach: true
  enableDebuggingHandlers: true
  enforceNodeAllocatable:
- pods
  eventBurst: 10
  eventRecordQPS: 5
  evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
  evictionPressureTransitionPeriod: 5m0s
  failSwapOn: true
  fileCheckFrequency: 20s
  hairpinMode: promiscuous-bridge
  healthzBindAddress: 127.0.0.1
  healthzPort: 10248
  httpCheckFrequency: 20s
  imageGCHighThresholdPercent: 85
  imageGCLowThresholdPercent: 80
  imageMinimumGCAge: 2m0s
  iptablesDropBit: 15
  iptablesMasqueradeBit: 14
  kind: KubeletConfiguration
  kubeAPIBurst: 10
  kubeAPIQPS: 5
  makeIPTablesUtilChains: true
  maxOpenFiles: 1000000
  maxPods: 110
  nodeLeaseDurationSeconds: 40
  nodeStatusReportFrequency: 1m0s
  nodeStatusUpdateFrequency: 10s
  oomScoreAdj: -999
  podPidsLimit: -1
  port: 10250
  registryBurst: 10
  registryPullQPS: 5
  resolvConf: /etc/resolv.conf
  rotateCertificates: true
  runtimeRequestTimeout: 2m0s
  serializeImagePulls: true
  staticPodPath: /etc/kubernetes/manifests
  streamingConnectionIdleTimeout: 4h0m0s
  syncFrequency: 1m0s
  topologyManagerPolicy: none
  volumeStatsAggPeriod: 1m0s
  {{< / highlight >}}

### 下载所需镜像(可选)

初始化的时候也会自动下载所有的镜像,提前下载好镜像可减少后续初始化时的等待时间

```shell
$ kubeadm config images list --config kubeadm-config.yaml # 查看镜像列表
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.16.3
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.16.3
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.16.3
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.16.3
registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.1
registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.3.15-0
registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.6.2
$ kubeadm config images pull --config kubeadm-config.yaml # 下载镜像
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.16.3
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.16.3
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.16.3
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.16.3
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.1
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.3.15-0
[config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.6.2
```

### 初始化

```shell
$ kubeadm init --config kubeadm-config.yaml
[init] Using Kubernetes version: v1.16.3
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Activating the kubelet service
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [k8s-master kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 10.64.144.100]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [k8s-master localhost] and IPs [10.64.144.100 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [k8s-master localhost] and IPs [10.64.144.100 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 34.001666 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.16" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node k8s-master as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node k8s-master as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: d7cb0e.605fb37fed0895d3
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.64.144.100:6443 --token d7cb0e.605fb37fed0895d3 \
    --discovery-token-ca-cert-hash sha256:dcabbe4b86492eb5da1f22c2a24fb1b5698557185abd481d858eda25a8d926aa
```

如上提示有三条信息

- 需要配置 kubeconfig ,以便 kubectl 可以操作集群
- 需要选择一个网络插件进行安装，以使集群可以连接网络通信
- 保存最后一条命令，后续 worker 节点加入集群时会用到

### 配置 kubeconfig

按上面提示操作就行

```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

使用 kubectl 查看集群当前状态，顺便检验 kubeconfig 是否配置正确

```shell
$ kubectl get pods -n kube-system # 查看系统组件运行状态
NAME                                 READY   STATUS    RESTARTS   AGE
coredns-67c766df46-5d5pp             0/1     Pending   0          4m40s
coredns-67c766df46-kjd9f             0/1     Pending   0          4m40s
etcd-k8s-master                      1/1     Running   0          3m34s
kube-apiserver-k8s-master            1/1     Running   0          3m41s
kube-controller-manager-k8s-master   1/1     Running   0          3m44s
kube-proxy-r2w74                     1/1     Running   0          4m39s
kube-scheduler-k8s-master            1/1     Running   0          3m44s
$ kubectl get nodes  # 查看节点状态
NAME         STATUS     ROLES    AGE    VERSION
k8s-master   NotReady   master   5m6s   v1.16.3
```

以上两个 coredns 的 pod 处于 Pending 状态，以及 master 节点 NotReady 状态，是由于我们集群中还没有安装可通信的网络插件，暂时先忽略即可

## 加入 worker 节点到集群

> 在 woker 节点上操作

还记得`kubeadm init`后输出的最后一条命令吗，worker 节点上只需要执行这一条命令就行了

```shell
$ kubeadm join 10.64.144.100:6443 --token d7cb0e.605fb37fed0895d3 --discovery-token-ca-cert-hash sha256:dcabbe4b86492eb5da1f22c2a24fb1b5698557185abd481d858eda25a8d926aa
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.16" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Activating the kubelet service
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

> master 节点上操作

所有节点上都执行后，在 master 上查看节点状态

```shell
$ kubectl get nodes
NAME         STATUS     ROLES    AGE   VERSION
k8s-master   NotReady   master   18m   v1.16.3
k8s-node01   NotReady   <none>   57s   v1.16.3
k8s-node02   NotReady   <none>   20s   v1.16.3
k8s-node03   NotReady   <none>   9s    v1.16.3
k8s-node04   NotReady   <none>   2s    v1.16.3
k8s-node05   NotReady   <none>   2s    v1.16.3
k8s-node06   NotReady   <none>   2s    v1.16.3
```

到目前为止，我们集群已经创建好的，但是节点间还不能相互通信，所以状态都还是`NotReady`

接下来我们将选择一款适合自己的网络插件来部署

## 安装 CNI 网络插件

> 所有操作均在 master 节点上

注意事项及支持的网络插件列表请参考官网 [Installing a pod network add-on](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network) 及 [Networking and Network Policy](https://kubernetes.io/docs/concepts/cluster-administration/addons/#networking-and-network-policy)

关于各 CNI 插件的功能对比可以参考 [Kubernetes CNI 网络最强对比：Flannel、Calico、Canal 和 Weave](https://blog.csdn.net/RancherLabs/article/details/88885539) 和性能基准测试 [K8S 网络插件（CNI）超过 10Gbit/s 的基准测试结果](https://zhuanlan.zhihu.com/p/53296042)

由于我们实验环境是二层可达，所以选用 Calico BGP 模式，如果二层不可达，可以考虑使用 Calico IPIP 或者 Flannel VXLAN

### 下载 calico.yaml

```shell
curl -O https://docs.projectcalico.org/manifests/calico.yaml
```

默认情况下，不做修改也可以，calico 会自动发现 cluster cidr, 这里我们自行修改下

主要改动如下:

- `CALICO_IPV4POOL_IPIP` 设置为 `Never`，关闭 IPIP 模式
- `CALICO_IPV4POOL_CIDR` 设置为 `172.16.0.0/16`，对应 kubeadm 中的 `cluster-cidr`

### 部署

```shell
kubectl apply -f caclico.yml
```

### 通过 calicoctl 查看 BGP 节点状态

calicoctl 可以支持通过直接操作 `etcd` 或者对接 kubernetes apiserver 两种数据源，我这里使用 kubernetes 方式

具体的安装及配置方式请参考官网 [Install calicoctl](https://docs.projectcalico.org/getting-started/clis/calicoctl/install) 与 [Configure calicoctl](https://docs.projectcalico.org/getting-started/clis/calicoctl/configure/overview)

```shell
$ export CALICO_DATASTORE_TYPE=kubernetes
$ export CALICO_KUBECONFIG=/etc/kubernetes/admin.conf
$ calicoctl node status
Calico process is running.

IPv4 BGP status
+---------------+-------------------+-------+----------+-------------+
| PEER ADDRESS  |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+---------------+-------------------+-------+----------+-------------+
| 10.64.144.101 | node-to-node mesh | up    | 18:11:48 | Established |
| 10.64.144.102 | node-to-node mesh | up    | 18:10:48 | Established |
| 10.64.144.103 | node-to-node mesh | up    | 18:08:10 | Established |
| 10.64.144.104 | node-to-node mesh | up    | 18:22:31 | Established |
| 10.64.144.105 | node-to-node mesh | up    | 18:22:31 | Established |
| 10.64.144.106 | node-to-node mesh | up    | 18:07:43 | Established |
+---------------+-------------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.
```

### 确认查看集群状态

```shell
$ kubectl get pods -n kube-system
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-7d4d547dd6-qr98t   1/1     Running   0          22s
calico-node-8d8bm                          1/1     Running   0          22s
calico-node-hlv2h                          1/1     Running   0          22s
calico-node-nbt4b                          1/1     Running   0          22s
calico-node-p4s45                          1/1     Running   0          22s
calico-node-srb88                          1/1     Running   0          22s
calico-node-vfpfv                          1/1     Running   0          22s
coredns-67c766df46-5d5pp                   1/1     Running   0          15m
coredns-67c766df46-kjd9f                   1/1     Running   0          15m
etcd-k8s-master                            1/1     Running   0          15m
kube-apiserver-k8s-master                  1/1     Running   0          14m
kube-controller-manager-k8s-master         1/1     Running   0          15m
kube-proxy-5tx57                           1/1     Running   0          14m
kube-proxy-6zvzv                           1/1     Running   0          14m
kube-proxy-hv8zt                           1/1     Running   0          14m
kube-proxy-rfsjl                           1/1     Running   0          14m
kube-proxy-tm7jz                           1/1     Running   0          15m
kube-proxy-kkssc                           1/1     Running   0          13m
kube-scheduler-k8s-master                  1/1     Running   0          14m
$ kubectl get nodes
NAME         STATUS   ROLES    AGE   VERSION
k8s-master   Ready    master   16m   v1.16.3
k8s-node01   Ready    <none>   14m   v1.16.3
k8s-node02   Ready    <none>   14m   v1.16.3
k8s-node03   Ready    <none>   14m   v1.16.3
k8s-node04   Ready    <none>   14m   v1.16.3
k8s-node05   Ready    <none>   15m   v1.16.3
k8s-node06   Ready    <none>   14m   v1.16.3
```

此时可以发现所有节点已经是 Ready 状态，且 coredns 也已经正常 Running

## 配置节点之间流量加密 (可选)

> 所有节点上操作

calico 支持通过 `WireGuard` 对节点之间的流量进行加密传输(仅支持 BGP 模式)

> 目前处于 preview 状态，不可用于生产环境

安装 WireGuard 可参考 [WireGuard Installation](https://www.wireguard.com/install/)

```shell
yum install epel-release https://www.elrepo.org/elrepo-release-7.el7.elrepo.noarch.rpm
yum install yum-plugin-elrepo
yum install kmod-wireguard wireguard-tools
```

加载 WireGuard 内核模块

```shell
$ modprobe wireguard
$ lsmod | grep wireguard
wireguard              86016  0
curve25519_x86_64      40960  1 wireguard
libchacha20poly1305    16384  1 wireguard
ip6_udp_tunnel         16384  1 wireguard
udp_tunnel             16384  1 wireguard
libblake2s             16384  1 wireguard
libcurve25519_generic    53248  2 curve25519_x86_64,wireguard
```

添加模块开机自动加载

```shell
$ cat > /etc/sysconfig/modules/wireguard.modules <<'EOF'
#!/bin/bash
/sbin/modinfo -F filename wireguard > /dev/null 2>&1
if [ $? -eq 0 ]; then
    /sbin/modprobe wireguard
fi
EOF
$ chmod +x wireguard.modules
```

修改配置开启 wireguard 功能

```shell
$ calicoctl patch felixconfiguration default --type='merge' -p '{"spec":{"wireguardEnabled":true}}'
Successfully patched 1 'FelixConfiguration' resource
```

查看节点状态

```shell
$ calicoctl get node k8s-master -oyaml
apiVersion: projectcalico.org/v3
kind: Node
....
status:
  wireguardPublicKey: 1OP5d3RVhBK7FE9tJ6l+eG38isdiy3LHzfqPzBSwI34=
```

## 测试

### 创建 pod 测试集群是否能正常工作

尝试创建名为 busybox 的 pod

> 由于 1.28 以上版本的 busybox 镜像，使用 nslookup 的时候有些 bug 导致不能解析域名，此处使用官方提供提供的 1.28 版本 busybox 的 pod 配置

```shell
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/admin/dns/busybox.yaml
pod/busybox created
```

### 测试 coredns 能否正常工作

在 busybox pod 中测试

```shell
$ kubectl exec -it busybox -- nslookup kubernetes.default.svc.cluster.local
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes.default.svc.cluster.local
Address 1: 10.96.0.1 kubernetes.default.svc.cluster.local
```

在节点中使用 dig 命令测试

```shell
$ dig kubernetes.default.svc.cluster.local @10.96.0.10
; <<>> DiG 9.11.4-P2-RedHat-9.11.4-9.P2.el7 <<>> kubernetes.default.svc.cluster.local @10.96.0.10
;; global options: +cmd
;; Got answer:
;; WARNING: .local is reserved for Multicast DNS
;; You are currently testing what happens when an mDNS query is leaked to DNS
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 56913
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;kubernetes.default.svc.cluster.local. IN A

;; ANSWER SECTION:
kubernetes.default.svc.cluster.local. 14 IN A   10.96.0.1

;; Query time: 1 msec
;; SERVER: 10.96.0.10#53(10.96.0.10)
;; WHEN: Fri Nov 22 12:21:58 CST 2019
;; MSG SIZE  rcvd: 117
```

## 安装 Ingress

配置文件参考官方 [mandatory.yaml](https://github.com/kubernetes/ingress-nginx/blob/master/deploy/static/mandatory.yaml)

不过官方是使用 `Deployment` 来部署的，我这里改成了 `DaemonSet` 方式，同时补上了 `Service` 的部分

完整配置如下

```shell
$ cat > ingress-nginx.yaml <<'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: tcp-services
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: udp-services
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress-serviceaccount
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: nginx-ingress-clusterrole
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - endpoints
      - nodes
      - pods
      - secrets
    verbs:
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - services
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - events
    verbs:
      - create
      - patch
  - apiGroups:
      - "extensions"
      - "networking.k8s.io"
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - "extensions"
      - "networking.k8s.io"
    resources:
      - ingresses/status
    verbs:
      - update
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role
metadata:
  name: nginx-ingress-role
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - pods
      - secrets
      - namespaces
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - configmaps
    resourceNames:
      # Defaults to "<election-id>-<ingress-class>"
      # Here: "<ingress-controller-leader>-<nginx>"
      # This has to be adapted if you change either parameter
      # when launching the nginx-ingress-controller.
      - "ingress-controller-leader-nginx"
    verbs:
      - get
      - update
  - apiGroups:
      - ""
    resources:
      - configmaps
    verbs:
      - create
  - apiGroups:
      - ""
    resources:
      - endpoints
    verbs:
      - get
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: nginx-ingress-role-nisa-binding
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: nginx-ingress-role
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: nginx-ingress-clusterrole-nisa-binding
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: nginx-ingress-clusterrole
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/part-of: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
      annotations:
        prometheus.io/port: "10254"
        prometheus.io/scrape: "true"
    spec:
      # wait up to five minutes for the drain of connections
      terminationGracePeriodSeconds: 300
      serviceAccountName: nginx-ingress-serviceaccount
      nodeSelector:
        kubernetes.io/os: linux
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.26.1
          args:
            - /nginx-ingress-controller
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
            - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
            - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
            - --publish-service=$(POD_NAMESPACE)/ingress-nginx
            - --annotations-prefix=nginx.ingress.kubernetes.io
          securityContext:
            allowPrivilegeEscalation: true
            capabilities:
              drop:
                - ALL
              add:
                - NET_BIND_SERVICE
            # www-data -> 33
            runAsUser: 33
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - name: http
              containerPort: 80
            - name: https
              containerPort: 443
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 10
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 10
          lifecycle:
            preStop:
              exec:
                command:
                  - /wait-shutdown
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: 80
    - name: https
      port: 443
      protocol: TCP
      targetPort: 443
EOF
```

部署

```shell
$ kubectl apply -f ingress-nginx.yaml  # 执行部署
namespace/ingress-nginx created
configmap/nginx-configuration created
configmap/tcp-services created
configmap/udp-services created
serviceaccount/nginx-ingress-serviceaccount created
clusterrole.rbac.authorization.k8s.io/nginx-ingress-clusterrole created
role.rbac.authorization.k8s.io/nginx-ingress-role created
rolebinding.rbac.authorization.k8s.io/nginx-ingress-role-nisa-binding created
clusterrolebinding.rbac.authorization.k8s.io/nginx-ingress-clusterrole-nisa-binding created
daemonset.apps/nginx-ingress-controller created
service/nginx-ingress-controller created
$ kubectl get pods -n ingress-nginx  # 查看pod运行状态
NAME                             READY   STATUS    RESTARTS   AGE
nginx-ingress-controller-2hc8m   1/1     Running   0          4m36s
nginx-ingress-controller-69rfm   1/1     Running   0          4m36s
nginx-ingress-controller-ps7gt   1/1     Running   0          4m36s
nginx-ingress-controller-zfq56   1/1     Running   0          4m36s
$ kubectl get service -n ingress-nginx  # 查看service状态
NAME                       TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
nginx-ingress-controller   LoadBalancer   10.96.76.163   <pending>     80:31832/TCP,443:30219/TCP   6m10s
```

此时发现 pod 已经正常运行了，但是 service 的 EXTERNAL-IP 字段却是 pending 状态

这是由于我们集群中还没有能处理 `LoadBalancer` 类型 service 的程序

在云环境中都会有个额外的组件 `Cloud-Controller-Manager(CCM)` 来对接云平台的负载均衡服务，如阿里云的 SLB，AWS 的 ALB 等

那么线下环境有没有办法实现类似 CCM 的功能呢，答案是 `metallb`

## 部署 Metallb

metallb 是一款为祼机 Kubernetes 环境提供标准路由协议的开源软负载均衡器，支持基于二层网络 ARP 或基于 BGP 方式进行地址广播

更详细的介绍请参考[官方文档](https://metallb.universe.tf/)或[Github](https://github.com/danderson/metallb)

关于裸机下基于 Metallb 的负载均衡方案可参考 ingress-nginx 官方文档 [A pure software solution: MetalLB](https://kubernetes.github.io/ingress-nginx/deploy/baremetal/#a-pure-software-solution-metallb)

由于当前的物理环境中不支持 BGP，所以使用最简单的二层 ARP 方式

部署:

```shell
$ kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.8.3/manifests/metallb.yaml
namespace/metallb-system created
podsecuritypolicy.policy/speaker created
serviceaccount/controller created
serviceaccount/speaker created
clusterrole.rbac.authorization.k8s.io/metallb-system:controller created
clusterrole.rbac.authorization.k8s.io/metallb-system:speaker created
role.rbac.authorization.k8s.io/config-watcher created
clusterrolebinding.rbac.authorization.k8s.io/metallb-system:controller created
clusterrolebinding.rbac.authorization.k8s.io/metallb-system:speaker created
rolebinding.rbac.authorization.k8s.io/config-watcher created
daemonset.apps/speaker created
deployment.apps/controller created
$ kubectl get pods -n metallb-system
NAME                          READY   STATUS    RESTARTS   AGE
controller-65895b47d4-fbs7t   1/1     Running   0          3m21s
speaker-2skdk                 1/1     Running   0          3m21s
speaker-6mg76                 1/1     Running   0          3m21s
speaker-7rr8q                 1/1     Running   0          3m21s
speaker-dv2rh                 1/1     Running   0          3m21s
speaker-mwkqr                 1/1     Running   0          3m21s
```

创建配置文件

addresses 表示 LoadBalancer 可使用的 IP 池范围

```shell
$ cat > metallb-configmap.yaml <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 10.64.144.150-10.64.144.250
EOF
$ kubectl apply -f metallb-configmap.yaml
configmap/config created
```

而此时再去查看 ingree-nginx 的 service 可以发现已经 EXTERNAL-IP 字段已经能正常拿到 IP 了

```shell
$ kubectl get service -n ingress-nginx
NAME                       TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                      AGE
nginx-ingress-controller   LoadBalancer   10.96.76.163   10.64.144.150   80:31832/TCP,443:30219/TCP   50m
```

## 测试 ingress-nginx 及 metallb

前面已经创建了 ingress-nginx 及 metallb,但到目前为止也只是看到了集群中的状态，并不能说明它俩是真的可用

所以我们来创建一个简单的应用来测试下

```shell
$ cat > nginx-test.yaml <<EOF
---
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  name: nginx
  labels:
    run: nginx
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
    name: http
  selector:
    run: nginx
status:
  loadBalancer: {}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      run: nginx
  template:
    metadata:
      creationTimestamp: null
      labels:
        run: nginx
    spec:
      containers:
      - image: nginx:alpine
        name: nginx
        ports:
        - containerPort: 80
---
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: nginx
  labels:
    run: nginx
spec:
  rules:
  - host: test.nginx.com
    http:
      paths:
      - path: /
        backend:
          serviceName: nginx
          servicePort: http
EOF
$ kubectl apply -f nginx-test.yaml
service/nginx created
deployment.apps/nginx created
ingress.networking.k8s.io/nginx created
$ curl -H 'Host: test.nginx.com' http://10.64.144.150 # 在局域网中其他机器测试访问
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
```

由此 ingress-nginx 及负载均衡器 Metallb 已经部署完成
