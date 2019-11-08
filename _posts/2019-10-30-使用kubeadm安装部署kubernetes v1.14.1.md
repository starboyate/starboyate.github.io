---
layout: post
title: "安装部署kubernetes v1.14.1"
subtitle: 'kubeadm安装部署k8s简单集群'
author: "Starboyate"
header-img: "/img/singleton.jpg"
multilingual: true
tags:
  - kubernetes
  - kubeadm
---


## 前言
> 使用**kubeadm**的方式搭建kubernetes集群,前提是所有节点都必须安装了docker，docker详细安装文档可以参考[这里](https://help.aliyun.com/document_detail/60742.html?spm=5176.11065259.1996646101.searchclickresult.261ebdd3Qy7wLJ)，
如果想要安装其它版本的kubernetes也可以参考本教程，然后下载安装对应想要版本就行了。

<br/>

## 前置准备
#### 1.服务器配置
我们这里使用的是三台centos-7.2的虚拟机，具体信息如下表：

| 系统类型 | IP地址 | 节点角色 | CPU | Memory | Hostname |
| :------: | :--------: | :-------: | :-----: | :---------: | :-----: |
| centos-7.2 | 192.168.1.221 | master |   \>=6    | \>=16G | k8s-node1 |
| centos-7.2 | 192.168.1.222 | worker |   \>=6    | \>=16G | k8s-node2|
| centos-7.2 | 192.168.1.223 | worker |   \>=6    | \>=16G | k8s-node3 |

#### 2.系统设置
修改所有节点主机名，保证节点之间能通过hostname互相访问
```bash
# 查看hostname
$ hostname
# 修改hostname
$ hostnamectl set-hostname <your-hostname>
# 配置host，使所有节点之间可以通过hostname互相访问
$ vi /etc/hosts
# <node-ip> <node-hostname>
```

关闭防火墙、swap，重置iptables，禁用Selinux
```bash
# 关闭防火墙
$ systemctl stop firewalld && systemctl disable firewalld
# 重置iptables
$ iptables -F && iptables -X && iptables -F -t nat && iptables -X -t nat && iptables -P FORWARD ACCEPT
# 关闭swap
$ swapoff -a
$ sed -i '/swap/s/^\(.*\)$/#\1/g' /etc/fstab
# 关闭selinux
$ setenforce 0
# 关闭dnsmasq(否则可能导致docker容器无法解析域名)
$ systemctl stop dnsmasq && systemctl disable dnsmasq
```

配置内核参数
```bash
$ cat > /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
vm.swappiness=0
vm.overcommit_memory=1
vm.panic_on_oom=0
fs.inotify.max_user_watches=89100
EOF
# 生效文件
$ sysctl -p /etc/sysctl.d/kubernetes.conf
```

#### 3.安装必要工具
- **kubeadm:**  部署集群用的命令
- **kubelet:** 在集群中每台机器上都要运行的组件，负责管理pod、容器的生命周期
- **kubectl:** 集群管理工具（只要在master上安装即可） 

安装步骤：
```bash
# 配置yum源
$ cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
       http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

# 安装工具
# 找到要安装的版本号
$ yum list kubeadm --showduplicates | sort -r

# 安装指定版本（这里用的是1.14.1）
$ yum install -y kubeadm-1.14.1-0 kubelet-1.14.1-0 kubectl-1.14.1-0 --disableexcludes=kubernetes

# 设置kubelet的cgroupdriver（kubelet的cgroupdriver默认为systemd，如果上面没有设置docker的exec-opts为systemd，这里就需要将kubelet的设置为cgroupfs）
$ sed -i "s/cgroup-driver=systemd/cgroup-driver=cgroupfs/g" /lib/systemd/system/kubelet.service.d/10-kubeadm.conf

# 启动kubelet (worker节点不要启动kubelet)
$ systemctl enable kubelet && systemctl start kubelet

```

<br/>

## 2.master节点部署
```bash
# 初始化
$ kubeadm init \
--apiserver-advertise-address=192.168.1.221 \
--image-repository registry.aliyuncs.com/google_containers \
--kubernetes-version v1.14.1 \
--service-cidr=10.68.0.0/16 \
--pod-network-cidr=172.22.0.0/16

# copy kubectl配置（上一步会有提示）
$ mkdir -p ~/.kube
$ cp -i /etc/kubernetes/admin.conf ~/.kube/config

# 测试一下kubectl
$ kubectl get pods --all-namespaces

# **备份init打印的join命令**
```
- --apiserver-advertise-address：这里放master节点的ip
- --image-repository：这里是拉取镜像地址
- --kubernetes-version: 这里是指定kubernetes版本号
- --service-cidr：指定Service网络的范围，即负载均衡VIP使用的IP地址段
- --pod-network-cidr：pod的ip地址段

<br/>

## 3.部署calico网络插件
```bash
# 创建calico文件夹
$ mkdir -p /usr/local/calico

# 拉取calico文件
$ cd /usr/local/calico
$ wget https://docs.projectcalico.org/v3.1/getting-started/\
kubernetes/installation/hosted/rbac-kdd.yaml
$ wget https://docs.projectcalico.org/v3.1/getting-started/\
kubernetes/installation/hosted/kubernetes-datastore/calico-networking/1.7/calico.yaml

# 修改calico.yaml中的CALICO_IPV4POOL_CIDR值为刚才kubeadm init的pod网段172.22.0.0/16

# 不能科学上网的，需要将calico.yaml里面的image镜像地址替换成以下内容:
registry.cn-hangzhou.aliyuncs.com/kubernetes-base/node:v3.1.7
registry.cn-hangzhou.aliyuncs.com/kubernetes-base/cni:v3.1.7
registry.cn-hangzhou.aliyuncs.com/kubernetes-base/typha:v3.1.7

# 部署calico
$ kubectl apply -f rbac-kdd.yaml 
$ kubectl apply -f calico.yaml 

# 查看状态
$ kubectl get pods -n kube-system
```

<br/>

## 4.加入worker节点
```bash
# 使用之前kubeadm init最后生成最后一段命令，然后join命令加入集群
$ kubeadm join ...

# 耐心等待一会，并观察日志
$ journalctl -f

# 查看节点
$ kubectl get nodes
```

## 5.测试
#### 5.1 测试DNS
```bash
$ kubectl run curl --image=radial/busyboxplus:curl -it

# 然后进入容器执行nslookup kubernetes确认解析正常，如果正确显示server，address那就代表成功了
$ nslookup kubernetes

```

#### 5.2 测试calico网络插件
```bash
# 所有节点执行calicoctl node status,会出现一个表格，里面会显示其它节点，看到state如果是up那就代表没问题
$ calicoctl node status
```

## 6.总结
> 至此，一个kubernetes集群已经搭建成功了，后续会整合一些其它插件，第三方应用来慢慢完善kubernetes体系
