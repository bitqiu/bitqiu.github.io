---
title: 使用 kubeadm 安装 kubernetes
date: 2020-03-11 10:58:20
tags:
    - kubernetes
    - ops
---


## 配置要求
- 至少2台 2核4G 的服务器
- Cent OS 7.6

### 安装后的软件版本为

- Kubernetes v1.17.x
    - calico 3.13.1
    - nginx-ingress 1.5.5
- Docker 19.03.8
  
## 修改 hostname
```shell
# 修改 hostname
hostnamectl set-hostname your-new-host-name
# 查看修改结果
hostnamectl status
# 设置 hostname 解析
echo "127.0.0.1   $(hostname)" >> /etc/hosts

# 此处 hostname 的输出将会是该机器在 Kubernetes 集群中的节点名字
# 不能使用 localhost 作为节点的名字
cat /etc/redhat-release
hostname
```

## 检查网络

在所有节点执行命令

```shell
[root@k8s-master ~]# ip route show
default via 10.1.2.2 dev ens33 proto static metric 100
10.1.2.0/24 dev ens33 proto kernel scope link src 10.1.2.100 metric 100

[root@k8s-master ~]# ip address
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:2e:35:42 brd ff:ff:ff:ff:ff:ff
    inet 10.1.2.100/24 brd 10.1.2.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::a3e8:7936:50da:d341/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
```



> kubelet使用的IP地址
> - `ip route show` 命令中，可以知道机器的默认网卡，通常是 `eth0`，如 ***default via 10.1.2.2 dev eth0***
> - `ip address` 命令中，可显示默认网卡的 IP 地址，Kubernetes 将使用此 IP 地址与集群内的其他节点通信，如 `10.1.2.100`
> - 所有节点上 Kubernetes 所使用的 IP 地址必须可以互通（无需 NAT 映射、无安全组或防火墙隔离）

## 安装docker及kubelet

使用 root 身份在所有节点执行如下代码，以安装软件：

- docker
- nfs-utils
- kubectl / kubeadm / kubelet



手动执行以下代码，结果与快速安装相同。***请将脚本第79行（已高亮）的 ${1} 替换成您需要的版本号，例如 1.17.4***

> docker hub 镜像请根据自己网络的情况任选一个

```shell
# 在 master 节点和 worker 节点都要执行
# 最后一个参数 1.17.4 用于指定 kubenetes 版本，支持所有 1.17.x 版本的安装
# 腾讯云 docker hub 镜像
# export REGISTRY_MIRROR="https://mirror.ccs.tencentyun.com"
# DaoCloud 镜像
# export REGISTRY_MIRROR="http://f1361db2.m.daocloud.io"
# 阿里云 docker hub 镜像
# export REGISTRY_MIRROR=https://registry.cn-hangzhou.aliyuncs.com
# 微软Azure docker hub 镜像
export REGISTRY_MIRROR=https://dockerhub.azk8s.cn
```

**在 master 节点和 worker 节点都要执行**

```shell
# 卸载旧版本
yum remove -y docker \
docker-client \
docker-client-latest \
docker-common \
docker-latest \
docker-latest-logrotate \
docker-logrotate \
docker-selinux \
docker-engine-selinux \
docker-engine

# 设置 yum repository
yum install -y yum-utils \
device-mapper-persistent-data \
lvm2 \
wget
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# 安装并启动 docker
yum install -y docker-ce-19.03.8 docker-ce-cli-19.03.8 containerd.io
systemctl enable docker
systemctl start docker

# 安装 nfs-utils
# 必须先安装 nfs-utils 才能挂载 nfs 网络存储
yum install -y nfs-utils

# 关闭 防火墙
systemctl stop firewalld
systemctl disable firewalld

# 关闭 SeLinux
setenforce 0
sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config

# 关闭 swap
swapoff -a
yes | cp /etc/fstab /etc/fstab_bak
cat /etc/fstab_bak |grep -v swap > /etc/fstab

# 修改 /etc/sysctl.conf
# 如果有配置，则修改
sed -i "s#^net.ipv4.ip_forward.*#net.ipv4.ip_forward=1#g"  /etc/sysctl.conf
sed -i "s#^net.bridge.bridge-nf-call-ip6tables.*#net.bridge.bridge-nf-call-ip6tables=1#g"  /etc/sysctl.conf
sed -i "s#^net.bridge.bridge-nf-call-iptables.*#net.bridge.bridge-nf-call-iptables=1#g"  /etc/sysctl.conf
sed -i "s#^net.ipv6.conf.all.disable_ipv6.*#net.ipv6.conf.all.disable_ipv6=1#g"  /etc/sysctl.conf
sed -i "s#^net.ipv6.conf.default.disable_ipv6.*#net.ipv6.conf.default.disable_ipv6=1#g"  /etc/sysctl.conf
sed -i "s#^net.ipv6.conf.lo.disable_ipv6.*#net.ipv6.conf.lo.disable_ipv6=1#g"  /etc/sysctl.conf
sed -i "s#^net.ipv6.conf.all.forwarding.*#net.ipv6.conf.all.forwarding=1#g"  /etc/sysctl.conf

# 可能没有，追加
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-ip6tables = 1" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables = 1" >> /etc/sysctl.conf
echo "net.ipv6.conf.all.disable_ipv6 = 1" >> /etc/sysctl.conf
echo "net.ipv6.conf.default.disable_ipv6 = 1" >> /etc/sysctl.conf
echo "net.ipv6.conf.lo.disable_ipv6 = 1" >> /etc/sysctl.conf
echo "net.ipv6.conf.all.forwarding = 1"  >> /etc/sysctl.conf

# 执行命令以应用
sysctl -p

# 配置K8S的yum源
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
       http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

# 卸载旧版本
yum remove -y kubelet kubeadm kubectl

# 安装kubelet、kubeadm、kubectl
# 将 ${1} 替换为 kubernetes 版本号，例如 1.17.4
yum install -y kubelet-${1} kubeadm-${1} kubectl-${1}

# 修改docker Cgroup Driver为systemd
# # 将/usr/lib/systemd/system/docker.service文件中的这一行 ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
# # 修改为 ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --exec-opt native.cgroupdriver=systemd
# 如果不修改，在添加 worker 节点时可能会碰到如下错误
# [WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". 
# Please follow the guide at https://kubernetes.io/docs/setup/cri/
sed -i "s#^ExecStart=/usr/bin/dockerd.*#ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --exec-opt native.cgroupdriver=systemd#g" /usr/lib/systemd/system/docker.service

# 设置 docker 镜像，提高 docker 镜像下载速度和稳定性
# 如果您访问 https://hub.docker.io 速度非常稳定，亦可以跳过这个步骤
mkdir -p /etc/docker
tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://dockerhub.azk8s.cn"]
}
EOF
systemctl daemon-reload
systemctl restart docker
systemctl enable kubelet && systemctl start kubelet
```

## 初始化 master 节点

```shell
# 只在 master 节点执行
# 替换 x.x.x.x 为 master 节点的内网IP
# export 命令只在当前 shell 会话中有效，开启新的 shell 窗口后，如果要继续安装过程，请重新执行此处的 export 命令
export MASTER_IP=x.x.x.x
# 替换 apiserver.demo 为 您想要的 dnsName
export APISERVER_NAME=apiserver.demo
# Kubernetes 容器组所在的网段，该网段安装完成后，由 kubernetes 创建，事先并不存在于您的物理网络中
export POD_SUBNET=10.100.0.1/16
echo "${MASTER_IP}    ${APISERVER_NAME}" >> /etc/hosts
```

**只在 master 节点执行**

```shell
# 查看完整配置选项 https://godoc.org/k8s.io/kubernetes/cmd/kubeadm/app/apis/kubeadm/v1beta2
rm -f ./kubeadm-config.yaml
cat <<EOF > ./kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: v${1}
imageRepository: gcr.azk8s.cn/google_containers
controlPlaneEndpoint: "${APISERVER_NAME}:6443"
networking:
  serviceSubnet: "10.96.0.0/16"
  podSubnet: "${POD_SUBNET}"
  dnsDomain: "cluster.local"
EOF

# kubeadm init
# 根据您服务器网速的情况，您需要等候 3 - 10 分钟
kubeadm init --config=kubeadm-config.yaml --upload-certs

# 配置 kubectl
rm -rf /root/.kube/
mkdir /root/.kube/
cp -i /etc/kubernetes/admin.conf /root/.kube/config

# 安装 calico 网络插件
# 参考文档 https://docs.projectcalico.org/v3.13/getting-started/kubernetes/self-managed-onprem/onpremises
echo "安装calico-3.13"
rm -f calico.yaml
curl https://docs.projectcalico.org/v3.13/manifests/calico.yaml -O
kubectl apply -f calico.yaml
```

**检查 master 初始化结果**

```shell
# 只在 master 节点执行

# 执行如下命令，等待 3-10 分钟，直到所有的容器组处于 Running 状态
watch kubectl get pod -n kube-system -o wide

# 查看 master 节点初始化结果
kubectl get nodes -o wide
```

## 初始化 worker节点

### 获得 join命令参数

**在 master 节点上执行**

```shell
kubeadm token create --print-join-command
```
可获取kubeadm join 命令及参数，如下所示


```shell
kubeadm join apiserver.demo:6443 --token mpfjma.4vjjg8flqihor4vt     --discovery-token-ca-cert-hash sha256:6f7a8e40a810323672de5eee6f4d19aa2dbdb38411845a1bf5dd63485c43d303
```

### 初始化worker

**针对所有的 worker 节点执行**

```shell
# 只在 worker 节点执行
# 替换 x.x.x.x 为 master 节点的内网 IP
export MASTER_IP=x.x.x.x
# 替换 apiserver.demo 为初始化 master 节点时所使用的 APISERVER_NAME
export APISERVER_NAME=apiserver.demo
echo "${MASTER_IP}    ${APISERVER_NAME}" >> /etc/hosts

# 替换为 master 节点上 kubeadm token create 命令的输出
kubeadm join apiserver.demo:6443 --token mpfjma.4vjjg8flqihor4vt     --discovery-token-ca-cert-hash sha256:6f7a8e40a810323672de5eee6f4d19aa2dbdb38411845a1bf5dd63485c43d303
 
```

### 检查初始化结果

在 master 节点上执行

```shell
kubectl get nodes -o wide
```

输出结果如下所示：


```shell
[root@k8s-master ~]# kubectl get nodes
NAME         STATUS   ROLES    AGE     VERSION
k8s-master   Ready    master   3m46s   v1.17.4
k8s-node     Ready    <none>   2m20s   v1.17.4
```