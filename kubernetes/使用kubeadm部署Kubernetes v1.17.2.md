# 使用kubeadm部署Kubernetes v1.17.2
本文使用kubeadm部署Kubernetes v1.17.2，同时也可被参考部署Kubernetes v1.17.x的其他版本。如果需要部署v1.17.x其他版本，只需要在第3步、第4步中更改相应的Kubernetes版本号即可。

## 环境

### 主机环境
准备两台主机或者虚拟机用于部署集群，后续节点可以根据需要进行添加，下面是此次部署Kubernetes的主机基本信息。

|     主机IP     | 主机名  | 系统      |
| :------------: | ------- | --------- |
| 192.168.11.136 | master  | Centos7.8 |
| 192.168.11.137 | nodeone | Centos7.8 |
## 0. 注意事项

注意第1-3步骤在每个主机上都需要进行；**第4步只在集群的Master节点运行**；**第5步在想要加入集群的其他节点上运行（除了Master节点的其他节点）**。

同时检查确保每个主机的如下状态：

- 任意节点 hostname（主机名） 不是 localhost，且不包含下划线、小数点、大写字母；

- 任意节点都有固定的内网 IP 地址；

- 任意节点都能访问外网，且相互之间可以ping通。

如果需要修改 hostname，可在需要修改的主机中执行如下指令：

```powershell
# 修改 hostname
hostnamectl set-hostname [你的新主机名]
# 查看修改结果
hostnamectl status
# 设置 hostname 解析
echo "127.0.0.1   $(hostname)" >> /etc/hosts
```

## 1. 系统基础环境配置
注意: 以下命令均在root模式下运行
### 禁用防火墙
```powershell
$ systemctl stop firewalld
$ systemctl disable firewalld
```
### 禁用SELINUX
```powershell
$ setenforce 0
$ sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
```
### 关闭交换区

Kubernetes 从1.8开始要求关闭系统的 Swap ，如果不关闭，默认配置的 kubelet 将无法启动，虽然可以通过 kubelet 的启动参数`--fail-swap-on=false`更改这个限制，但是最好还是将 swap 给关掉，这样能提高 kubelet 的性能。

```powershell
$ swapoff -a
$ yes | cp /etc/fstab /etc/fstab_bak
$ cat /etc/fstab_bak |grep -v swap > /etc/fstab
```

### 修改 /etc/sysctl.conf

将桥接的IPv4流量传递到iptables的链

```powershell
# 在/etc/sysctl.conf文件中追加如下内容
$ echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
$ echo "net.bridge.bridge-nf-call-ip6tables = 1" >> /etc/sysctl.conf
$ echo "net.bridge.bridge-nf-call-iptables = 1" >> /etc/sysctl.conf
# 执行命令使之生效
$ sysctl -p
```

## 2. 安装Docker

### 卸载旧版本Docker（可选）
如果曾经安装过Docker，可用以下命令卸载旧版本（没有安装过则可忽略该命令)

```powershell
$ yum remove -y docker \
docker-client \
docker-client-latest \
docker-common \
docker-latest \
docker-latest-logrotate \
docker-logrotate \
docker-selinux \
docker-engine-selinux \
docker-engine
```
### 设置Docker yum源
```powershell
$ yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2
$ yum-config-manager --add-repo \
http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```
### 安装最新版本的Docker

```powershell
# 本文安装时最新Docker版本为19.03.13, 可以选择安装这个版本, 命令为:
# yum install -y docker-ce-19.03.13 docker-ce-cli-19.03.13 containerd.io
$ yum install docker-ce docker-ce-cli containerd.io
$ systemctl enable docker && systemctl start docker
```

## 3. 安装kubeadm、kubelet、kubectl

### 设置Kubernetes yum源

```powershell
$ cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```
### 安装kubeadm、kubelet、kubectl

```powershell
# 可根据实际版本修改命令,将以下命令中的${1}替换为自己想安装的版本号，本文安装1.17.2版本
# yum install -y kubelet-${1} kubeadm-${1} kubectl-${1}
$ yum install -y kubelet-1.17.2 kubeadm-1.17.2 kubectl-1.17.2
```

### 配置Docker

安装完成后，我们还需要对`docker`进行配置，因为用`yum`源的方式安装的`kubelet`生成的配置文件将参数`--cgroup-driver`改成了`systemd`，而 docker 的`cgroup-driver`是`cgroupfs`，这二者必须一致才行，修改docker Cgroup Driver为systemd。

```powershell
$ sed -i "s#^ExecStart=/usr/bin/dockerd.*#ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --exec-opt native.cgroupdriver=systemd#g" /usr/lib/systemd/system/docker.service
# 重启docker
$ systemctl daemon-reload
$ systemctl restart docker
```
修改完成后可以通过`docker info`命令查看修改结果：

```powershell
$ docker info |grep Cgroup
Cgroup Driver: systemd
```
### 启动kubelet

```powershell
$ systemctl enable kubelet && systemctl start kubelet
```


## 4. Master节点初始化

**该步骤只在Master节点上运行！**
### 初始化命令
- `--apiserver-advertise-address`指定的是 apiserver 的通信地址，这里我们应该选择 Master 节点的 IP 地址
- 因为我们之后选择`flannel`作为 Pod 的网络插件，所以需要指定`–-pod-network-cidr=10.244.0.0/16`。

```powershell
$ kubeadm init \
--apiserver-advertise-address=192.168.11.136 \
--image-repository registry.cn-hangzhou.aliyuncs.com/google_containers \
--kubernetes-version v1.17.2 \
--service-cidr=10.1.0.0/16 \
--pod-network-cidr=10.244.0.0/16
```

运行完，返回的信息包含下面的信息即代表初始化成功：

```
Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join 192.168.11.136:6443 --token ivyfuh.31jiaci3rwjkcyx6     --discovery-token-ca-cert-hash sha256:5ee2a16681e694f439a431b69d58fd162efc317bc43b2066dfed6ac035d2168d
```

其中这个信息：`kubeadm join 192.168.11.136:6443 --token ivyfuh.31jiaci3rwjkcyx6     --discovery-token-ca-cert-hash sha256:5ee2a16681e694f439a431b69d58fd162efc317bc43b2066dfed6ac035d2168d`是在其他节点上运行用于加入集群中的命令，忘了也没关系，后面可以再通过其他命令获得。

### 配置kubectl

```powershell
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 安装Pod Network

接下来我们来安装`flannel`网络插件，对于Kubernetes v1.17.x和更高版本的Kubernetes可以直接使用以下命令

```powershell
$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

### 查看集群状态

安装完成后使用 kubectl get nodes命令查看各节点状态，如果节点状态为Ready，则master节点安装成功。

```powershell
$ kubectl get nodes
```

输出结果如下所示，master节点状态为Ready则初始化成功

```shell
[root@master ~]# kubectl get nodes
NAME     STATUS   ROLES    AGE     VERSION
master   Ready    master   5m3s    v1.17.2
```

### 获得join命令参数

在Master节点上运行以下命令，返回的join命令及参数在工作节点上运行，使工作节点加入集群。

```powershell
$ kubeadm token create --print-join-command
```

## 5. 工作节点加入集群

在工作节点上运行第1-3步后，根据第4步在Master节点上运行`kubeadm token create --print-join-command` 命令获得join命令参数，并在工作节点上运行join命令即可加入集群：

```powershell
# 替换为 master 节点上 kubeadm token create --print-join-command 命令的输出
$ kubeadm join 192.168.11.136:6443 --token ivyfuh.31jiaci3rwjkcyx6     --discovery-token-ca-cert-hash sha256:5ee2a16681e694f439a431b69d58fd162efc317bc43b2066dfed6ac035d2168d
```
## 6. 检查集群部署结果

在Master节点上运行以下命令

```powershell
# 只在 master 节点执行
$ kubectl get nodes
```

输出结果如下所示：

```powershell
[root@master ~]# kubectl get nodes
NAME     STATUS   ROLES    AGE     VERSION
master   Ready    master   5m3s    v1.17.2
nodeone  Ready    <none>   2m26s   v1.17.2
```

## 7. 其他

### Master节点初始化设置错误后如何重置

首先在Master节点上删除所有已经加入的节点，命令如下所示：

```powershell
# kubectl delete node [工作节点名称]
$ kubectl delete node nodeone
```

接着使用kubeadm reset命令**在Master节点和所有工作节点上都运行**来重置kubeadm

```powershell
$ kubeadm reset
```

最后清理kubelet环境，**仅仅在Master节点上运行**

```powershell
$ systemctl stop kubelet
$ docker rm -f -v $(docker ps  -a -q)

$ rm -rf /etc/kubernetes
$ rm -rf  /var/lib/etcd
$ rm -rf   /var/lib/kubelet
$ rm -rf  $HOME/.kube/config
$ iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X

$ yum reinstall -y kubelet
$ systemctl daemon-reload
$ systemctl restart docker
$ systemctl enable kubelet
```

重新初始化Master节点从第4步开始即可。