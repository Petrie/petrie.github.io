---
title: "M1 芯片 Mac 下 Vmware 搭建 k8s 集群"
date: 2023-05-27T18:58:13+08:00
draft: false
isCJKLanguage: true
tags :
    - "k8s"
    - "kubeadm"
categories:
    - "k8s"
---

## 环境信息

Mac 硬件信息，执行 `system_profiler SPHardwareDataType` 查看

```bash
$ system_profiler SPHardwareDataType
Hardware:
    Hardware Overview:
      Model Identifier: MacBookPro18,3
      Chip: Apple M1 Pro
```

Mac OS 信息
执行 `ssw_vers` 查看

```bash
$ sw_vers
ProductName:		macOS
ProductVersion:		13.1
```

VMware 版本信息

```text
VMware Fusion
Professional Version 13.0.1 (21139760)
```

Centos 版本[下载地址](https://mirrors.centos.org/mirrorlist?path=/9-stream/BaseOS/aarch64/iso/CentOS-Stream-9-latest-aarch64-dvd1.iso&redirect=1&protocol=https)

```text
CentOS-Stream-9-latest-aarch64-dvd1.iso
```

## 准备虚拟机

注意虚拟机的网络配置需要选择`Autodetect`模式，以便虚拟机和物理机器共享网络
两台机器：

- k8s-master 192.168.1.98
- k8s-node1 192.168.1.97

### 设置 Mac hosts

在 Mac 执行以下命令

```bash
echo '192.168.1.98 k8s-master\n192.168.1.97 k8s-node1 ' | sudo tee -a /etc/hosts
```

### 设置免密登录

在 Mac 执行以下命令

```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub root@k8s-master
ssh-copy-id -i ~/.ssh/id_rsa.pub root@k8s-node1
```

### 设置两台虚拟机器 hostname

在 Mac 执行以下命令

```bash
ssh 'root@k8s-master' "hostnamectl set-hostname k8s-master"
ssh 'root@k8s-node1' "hostnamectl set-hostname k8s-node1"
```

## 安装容器运行时间

后续将使用 kubeadm 工具，进行k8s的部署。主要参考文档：[使用 kubeadm 引导集群](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
以下步骤，将在官方文档的基础上进行改进。改进目标是适配国内的网络。
如：

- 替换yum安装源为阿里源
- 替换容器镜像源为阿里镜像源

### 总览

主要有以下几个步骤

- 容器运行时安装和配置的先决条件，如禁用swap交换分区、关闭防火墙、配置网络转发等
- 安装容器运行时 containerd

### 容器运行时安装和配置的先决条件

#### 1、禁用swap交换分区

```bash
sudo swapoff -a #临时禁用
sed -i 's/.*swap.*/#&/g' /etc/fstab #更新配置文件，永久禁用
```

#### 2、转发IPv4

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# 设置所需的 sysctl 参数，参数在重新启动后保持不变
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# 应用 sysctl 参数而不重新启动
sudo sysctl --system
```

#### 3、关闭selinux

```bash
setenforce 0
sed -ri 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
```

#### 4、关闭防火墙

```bash
systemctl disable firewalld #永久禁用防火墙
systemctl stop firewalld  #关闭防火墙
firewall-cmd --state #查看状态，返回应为 not running
```

### 安装容器运行时 containerd

[参考文档](https://kubernetes.io/zh-cn/docs/setup/production-environment/container-runtimes/#containerd)

#### 1、下载后分别拷贝到两台虚拟机中

使用官方二进制的形式安装。在[release](https://github.com/containerd/containerd/releases)中下载`containerd-1.7.1-linux-arm64.tar.gz`。

```bash
scp ~/Downloads/containerd-1.7.1-linux-arm64.tar.gz root@k8s-master:~
scp ~/Downloads/containerd-1.7.1-linux-arm64.tar.gz root@k8s-node1:~
```

#### 2、解压安装containerd

```shell
ssh 'root@k8s-master' "tar Cxzvf /usr/local ~/containerd-1.7.1-linux-arm64.tar.gz"
ssh 'root@k8s-node1' "tar Cxzvf /usr/local ~/containerd-1.7.1-linux-arm64.tar.gz"
```

#### 3、设置containerd开启启动

创建文件`containerd.service`配置文件

```shell
cd /tmp; 
wget https://ghproxy.com/https://raw.githubusercontent.com/containerd/containerd/main/containerd.service 
```

这里使用了加速网站，有条件的同学可以直接访问。

这时containerd.service 文件会被下载到/tmp/containerd.service。
使用以下命令将此文件同步到两台虚拟机。
首先创建目录

```shell
ssh 'root@k8s-master' "mkdir -p /usr/local/lib/systemd/system/"
ssh 'root@k8s-node1' "mkdir -p /usr/local/lib/systemd/system/"
```

同步文件

```shell
scp /tmp/containerd.service root@k8s-master:/usr/local/lib/systemd/system/
scp /tmp/containerd.service root@k8s-node1:/usr/local/lib/systemd/system/
```

启动containerd服务，并设置开机自启动

```shell
ssh 'root@k8s-master' "systemctl daemon-reload && systemctl enable --now containerd"
ssh 'root@k8s-node1' "systemctl daemon-reload && systemctl enable --now containerd"
```
#### 4、安装runc

在[release](https://github.com/opencontainers/runc/releases)中下载对应二进制文件，这里下载`runc.arm64`。
在 Mac 上执行以下命令，将其同步至两台虚拟机。

```shell
scp ~/Downloads/runc.arm64 root@k8s-master:~
scp ~/Downloads/runc.arm64 root@k8s-node1:~
```

在 Mac 上执行以下命令，安装runc
```shell
for i in master node1;
do
ssh "root@k8s-$i" "install -m 755 ~/runc.arm64 /usr/local/sbin/runc"
done
```


#### 5、设置 containerd 配置文件

在 Mac 下执行，设置containerd默认配置文件

```shell
for i in master node1;
do
ssh "root@k8s-$i"  \
"mkdir /etc/containerd && \
containerd config default > /etc/containerd/config.toml"
done
```

生成配置文件后，需要将 /etc/containerd/config.toml 文件中的 SystemdCgroup 设置为 true
或者直接在 MAC 下执行以下命令：

```shell
for i in master node1;
do
ssh "root@k8s-$i"  \
"sed -ri 's/SystemdCgroup = false/SystemdCgroup = true/' \
/etc/containerd/config.toml"
done
```

#### 6.沙箱镜像替换

在 Mac 下，执行以下命令：

```shell
for i in master node1;
do
ssh "root@k8s-$i" "sed -ri 's#sandbox_image = \"registry.k8s.io/pause:3.8\"#sandbox_image = \"registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.8\"#' /etc/containerd/config.toml"
done
```

到这里 containred 服务就配置完成了。
最后需要在重启containerd服务
执行以下命令：

```shell
for i in master node1;
do
ssh "root@k8s-$i"  \
"systemctl restart containerd"
done
```

可以登录到虚拟机，使用`systemctl status containerd.service`查看服务状态。

## 安装 kubeadm、kubelet 和 kubectl

[参考文档](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl)

### 添加yum源

参考以上官方文档，添加相关yum源，这里需要把google的yum源替换为阿里云的。

在 Mac 执行以下命令

```bash
for i in master node1;
do
ssh "root@k8s-$i"  "cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-aarch64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF"
done
```

### yum 安装kubelet kubeadm kubectl

```bash
for i in master node1;
do
ssh "root@k8s-$i" \
"sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes"
done
```

### 设置 kubelet 开机自启动

```bash
for i in master node1;
do
ssh "root@k8s-$i" \
sudo systemctl enable --now kubelet
done
```

## 使用 kubeadm 创建集群

### 生成默认配置

登录k8s-master节点
执行以下命令将默认配置倒入到 `admin.conf`

```bash
kubeadm config print  init-defaults > admin.conf
```

### 替换镜像地址

执行以下命令替换替换容器镜像地址

```bash
sed -ri 's/imageRepository: registry.k8s.io/imageRepository: registry.cn-hangzhou.aliyuncs.com\/google_containers/g' admin.conf
```

执行以下命令替换API Server启动ip

```bash
sed -ri 's/advertiseAddress: 1.2.3.4/advertiseAddress: 192.168.1.98/g' admin.conf
```

### 创建集群

```bash
kubeadm init --config=admin.conf
```

### 查看集群状态
执行以下命令

```bash
kubectl version
```
返回以下结果，代表成功
```text
Client Version: version.Info{Major:"1", Minor:"27", GitVersion:"v1.27.2", GitCommit:"7f6f68fdabc4df88cfea2dcf9a19b2b830f1e647", GitTreeState:"clean", BuildDate:"2023-05-17T14:20:07Z", GoVersion:"go1.20.4", Compiler:"gc", Platform:"linux/arm64"}
Kustomize Version: v5.0.1
Server Version: version.Info{Major:"1", Minor:"27", GitVersion:"v1.27.0", GitCommit:"1b4df30b3cdfeaba6024e81e559a6cd09a089d65", GitTreeState:"clean", BuildDate:"2023-04-11T17:04:24Z", GoVersion:"go1.20.3", Compiler:"gc", Platform:"linux/arm64"}
```