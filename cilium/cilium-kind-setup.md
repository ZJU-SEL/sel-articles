# kubernetes+cilium原生路由（native routing）环境搭建

## cilium路由模式介绍

cilium支持封装路由模式和原生路由模式：

1. 封装路由模式（也称为overlay模式）  
	默认的cilium路由模式。使用该模式需要在部署cilium时将`tunnel`设置为`vxlan`（默认值）或`geneve`，它们是两种基于UDP的网络封装协议。在这种模式下，所有集群节点使用基于vxlan或geneve这样的UDP 的封装协议形成隧道网格。
1. 原生路由模式（native routing）  
	使用该模式需要在部署cilium时将`tunnel`设置为`disabled`，在原生路由模式下，cilium会将所有不是发往另一个本地端点（endpoint）的数据包交给给Linux内核的路由子系统。这要求连接集群节点的网络必须能够路由pod所在CIDR的数据包。

两种路由模式的详细介绍和各自的优缺点可以查看[cilium官方文档](https://docs.cilium.io/en/stable/concepts/networking/routing/)。

其中的原生模式又可以分为两种情况：如果所有kubernetes节点都在同一个L2网络上，这时只需要在部署cilium时添加`--auto-direct-node-routes` flag即可，无需额外组件辅助；如果kubernetes节点并非在同一个L2网络上，则需要BGP daemon组件的辅助。

目前社区描述native routing搭建方法的[资料较少](https://github.com/cilium/cilium/issues/18914)，本文描述前一种情况下kubernetes和cilium的原生路由安装以及配置方法，即所有kubernetes节点都在同一个L2网络上。

## 用kind安装kubernetes

假设已经有一台安装了Ubuntu 22.04.1 LTS (Jammy Jellyfish)的机器，这里使用kind搭建kubernetes集群。

1. 根据[docker官方文档](https://docs.docker.com/engine/install/ubuntu/)，执行以下步骤安装docker
	1. `sudo apt-get remove docker docker-engine docker.io containerd runc` 
    1. `sudo apt-get update`
    1. `sudo apt-get install ca-certificates curl gnupg lsb-release`
    1. `sudo mkdir -p /etc/apt/keyrings`
    1. `curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg`
    1. `echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null`
    1. `sudo apt-get update`
    1. `sudo apt-get install docker-ce docker-ce-cli containerd.io`
    1. `sudo usermod -aG docker $USER`
    1. 重新登陆
1. 安装kind：  
    `go install sigs.k8s.io/kind@v0.17.0`
1. 创建kubernetes集群配置yaml  
    注意`kubeProxyMode: "none"`，这样就不会安装kube-proxy组件。这等同于用kubeadm安装kubernetes时加上`--skip-phases=addon/kube-proxy` flag：
    ```yaml
	kind: Cluster
	apiVersion: kind.x-k8s.io/v1alpha4
	nodes:
	- role: control-plane
	- role: worker
	- role: worker
	networking:
	  disableDefaultCNI: true
	  kubeProxyMode: "none" # --skip-phases=addon/kube-proxy
	```
1. 执行kind创建kubernetes集群，安装完成后kubeconfig文件位于`~/.kube/config`
	```bash
	kind create cluster --config="./kind-config.yaml"
	```
    > kind其他常用命令：
    > 1. `kind get clusters`：获取所有已经创建的cluster
    > 1. `kind delete cluster --name {cluster name}`：删除某个cluster。前面的create命令默认创建的集群名为kind

至此kubernetes集群已经安装完毕，执行`docker ps`可以查看当前正在运行的容器：

```bash
CONTAINER ID   IMAGE                  COMMAND                  CREATED       STATUS       PORTS                       NAMES
edd389143928   kindest/node:v1.25.3   "/usr/local/bin/entr…"   3 hours ago   Up 3 hours                               kind-worker
7608f48b238b   kindest/node:v1.25.3   "/usr/local/bin/entr…"   3 hours ago   Up 3 hours   127.0.0.1:37391->6443/tcp   kind-control-plane
7f662a356e63   kindest/node:v1.25.3   "/usr/local/bin/entr…"   3 hours ago   Up 3 hours                               kind-worker2
```

根据[官方文档](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)执行如下命令安装kubectl：

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

然后执行kubectl可获得kubernetes节点信息如下：
```bash
$ kubectl get nodes -o wide
NAME                 STATUS     ROLES           AGE    VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
kind-control-plane   NotReady   control-plane   2d3h   v1.25.3   172.18.0.4    <none>        Ubuntu 22.04.1 LTS   5.15.0-53-generic   containerd://1.6.9
kind-worker          NotReady   <none>          2d3h   v1.25.3   172.18.0.2    <none>        Ubuntu 22.04.1 LTS   5.15.0-53-generic   containerd://1.6.9
kind-worker2         NotReady   <none>          2d3h   v1.25.3   172.18.0.3    <none>        Ubuntu 22.04.1 LTS   5.15.0-53-generic   containerd://1.6.9
```

注意上面各节点显示状态为`NotReady`，这只是因为前面kind安装kubernetes集群时没有安装网络相关组件，等cilium安装完成后节点状态即恢复正常。

执行`docker network ls`命令可以看到kind创建名为`kind`的网桥，用于连接kubernetes集群各节点。三个kubernetes节点都连在这个网桥上，满足cilium原生路由模式的第一种情况，即所有kubernetes节点都在同一个L2网络上：
```bash
$ docker network ls
NETWORK ID     NAME      DRIVER    SCOPE
c063432ee4ad   kind      bridge    local
```

我们也可以看看kind是如何运行kubernetes的。执行`docker container exec -it {container name} /bin/bash`查看节点内部情况。如，执行`docker container exec -it kind-control-plane /bin/bash`连接kubernetes控制节点后，再执行`ctr -n k8s.io c ls`可以查看kind使用containerd运行的kubernetes各组件。这些容器运行在容器内部，称为嵌套容器（nested container）：

```bash
CONTAINER                                                           IMAGE                                              RUNTIME
51885bea02b150ad7fed71c93424a55ee921aa31608d4159d0421a7d8c37abb9    registry.k8s.io/etcd:3.5.4-0                       io.containerd.runc.v2
683ea9074e73c101fd62f4fcf90a5587c8f4ca5dfdf3d1f7e8315f0fe9f761b7    registry.k8s.io/kube-scheduler:v1.25.3             io.containerd.runc.v2
70eff3ca526f6e6e6f9ccb2d5361ca10a1ca41284d4cd4029617df8453f72f23    registry.k8s.io/kube-controller-manager:v1.25.3    io.containerd.runc.v2
765e366920dbd409d8e8bb86c0edfa1bd46a9b1baaff33bdecc6eeb17204209b    registry.k8s.io/pause:3.7                          io.containerd.runc.v2
776a4050d98b2e80d7ff021aea28e0b9c8c57df4d01aab254a92870b2c8bd00e    registry.k8s.io/kube-apiserver:v1.25.3             io.containerd.runc.v2
ed38372ebcbd8eccfa7a4be93d432d73eaf8501ec667ffc6b179dcf47fe2acb2    registry.k8s.io/pause:3.7                          io.containerd.runc.v2
f55d0cfa449d96a13c69bbdd0df62b76ad88bf05c7d8cf54365be88d1aee9fb7    registry.k8s.io/pause:3.7                          io.containerd.runc.v2
fa03c617589a1867f318fbf0897d490767be2989bec4fd9009688c2776df10f3    registry.k8s.io/pause:3.7                          io.containerd.runc.v2
```
执行`ip route`可以看到控制节点（kind-control-plane）内部当前只有基本的路由规则，其中的默认网关地址`172.18.0.1`为kind创建的网桥设备`br-xxx`的ip地址：

```bash
default via 172.18.0.1 dev eth0
172.18.0.0/16 dev eth0 proto kernel scope link src 172.18.0.4
```
## 安装helm

1. 根据[helm官方文档](https://helm.sh/docs/intro/install/)，安装helm 3  
    下载地址：https://github.com/helm/helm/releases
    > cilium不再支持helm 2
1. 配置helm的cilium源
    ```bash
    $ helm repo add cilium https://helm.cilium.io/
    ```

## 安装cilium

参考[cilium官方文档](https://docs.cilium.io/en/stable/gettingstarted/kind/)，执行如下命令在kind创建的集群上安装cilium：

```bash
$ docker pull quay.io/cilium/cilium:v1.12.4
$ kind load docker-image quay.io/cilium/cilium:v1.12.4
$ helm install cilium cilium/cilium --version 1.12.4 \
    --namespace kube-system \
    --set tunnel=disabled \
    --set autoDirectNodeRoutes=true \
    --set ipv4NativeRoutingCIDR="10.0.0.0/8" \
    --set kubeProxyReplacement=strict \
    --set k8sServiceHost=172.18.0.4 \
    --set k8sServicePort=6443 \
    --set bpf.masquerade=true
```

上面的helm安装指令说明：

1. `--namespace kube-system`：  
    默认安装在default namespace。由于clium cli默认cilium相关pod运行在`kube-system` namespace，于是这里在安装时指定`kube-system` namespace
1. `--set autoDirectNodeRoutes=true`：  
    等同于为cilium agent开启`--auto-direct-node-routes` flag。当所有的kubernetes节点都存在同一个L2网络上，开启该flag会让cilium调用`github.com/vishvananda/netlink`包，创建kubernetes节点之间的路由规则，使得跨kubernetes节点的pod网络数据包能够在不经过封装（encapsulation）的情况下路由到目的节点。
1. `--set ipv4NativeRoutingCIDR="10.0.0.0/8"`：  
    等同于为cilium agent设置`--ipv4-native-routing-cidr` flag，默认为空  
1. `--set k8sServiceHost=172.18.0.4`：  
    kubernetes apiserver地址
1. `--set k8sServicePort=6443`：  
    kubernetes apiserver端口

## cilium安装验证

根据[cilium官方文档](https://docs.cilium.io/en/stable/gettingstarted/kind/)，执行下面shell脚本安装cilium cli：

```bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/master/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```
执行cilium cli查看cilium各组建运行状态：

```bash
$ cilium status --wait
    /¯¯\
 /¯¯\__/¯¯\    Cilium:         OK
 \__/¯¯\__/    Operator:       OK
 /¯¯\__/¯¯\    Hubble:         disabled
 \__/¯¯\__/    ClusterMesh:    disabled
    \__/
 
Deployment        cilium-operator    Desired: 2, Ready: 2/2, Available: 2/2
DaemonSet         cilium             Desired: 3, Ready: 3/3, Available: 3/3
Containers:       cilium             Running: 3
                  cilium-operator    Running: 2
Cluster Pods:     3/3 managed by Cilium
Image versions    cilium             quay.io/cilium/cilium:v1.12.4@sha256:4b074fcfba9325c18e97569ed1988464309a5ebf64bbc79bec6f3d58cafcb8cf: 3
                  cilium-operator    quay.io/cilium/operator-generic:v1.12.4@sha256:071089ec5bca1f556afb8e541d9972a0dfb09d1e25504ae642ced021ecbedbd1: 2
```
这时执行kubectl，可以看到所有节点状态都为`Ready`：

```bash
$ kubectl get nodes -o wide
NAME                 STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
kind-control-plane   Ready    control-plane   54m   v1.25.3   172.18.0.2    <none>        Ubuntu 22.04.1 LTS   5.15.0-53-generic   containerd://1.6.9
kind-worker          Ready    <none>          54m   v1.25.3   172.18.0.3    <none>        Ubuntu 22.04.1 LTS   5.15.0-53-generic   containerd://1.6.9
kind-worker2         Ready    <none>          54m   v1.25.3   172.18.0.4    <none>        Ubuntu 22.04.1 LTS   5.15.0-53-generic   containerd://1.6.9
```
如果安装过程中遇到问题，可以查看[cilium官方的trouble shooting文档](https://docs.cilium.io/en/stable/operations/troubleshooting)。

## 测试hello world

这里使用Google cloud的[kubernetes sample application](https://github.com/GoogleCloudPlatform/kubernetes-engine-samples)中的hello-app验证跨节点网络通讯，执行如下kind命令将hello-app镜像加载到kubernetes节点中

```bash
kind load docker-image hello-app:1.0
```

执行如下kubectl命令，将容器镜像部署成deployment，扩容为2个实例，并创建对应service：

```bash
kubectl create deployment hello-app --image=hello-app:1.0
kubectl scale --replicas=2 deployment/hello-app
kubectl expose deployment hello-app --port=8080 --target-port=8080
```

执行如下kubectl命令，可以看到这时两个pod分别运行在worker和worker2节点上：

```bash
$ kubectl get pods -o wide
NAME                        READY   STATUS    RESTARTS   AGE   IP           NODE           NOMINATED NODE   READINESS GATES
hello-app-646c68546-tcjhs   1/1     Running   0          54s   10.0.2.199   kind-worker2   <none>           <none>
hello-app-646c68546-xdh5w   1/1     Running   0          15s   10.0.0.25    kind-worker    <none>           <none>
```
也可以看到service已经创建完成，对应cluster ip为`10.96.37.61`

```bash
$ kubectl get service
NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
hello-app    ClusterIP   10.96.37.61   <none>        8080/TCP   4s
```

执行如下cilium service list命令，可以看到cilium已经得知hello-app service的cluster ip，及对应的两个后端pod：

```bash
$ kubectl exec -it -n kube-system cilium-zqmdj -- cilium service list
ID   Frontend           Service Type   Backend
5    10.96.37.61:8080   ClusterIP      1 => 10.0.2.199:8080 (active)
                                       2 => 10.0.0.25:8080 (active)
```

执行如下命令，从kind-control-plane节点访问hello-app服务的cluster ip，验证跨节点通讯：

```bash
$ docker container exec -it kind-control-plane /bin/bash
root@kind-control-plane:/# curl http://10.96.37.61:8080
Hello, world!
Version: 1.0.0
Hostname: hello-app-646c68546-xdh5w
```
也可以用类似的方法在其中一个hello-app pod中curl hello-app服务，验证pod之间的跨节点通讯。这里不再赘述。至此cilium的原生路由配置及验证方法描述完毕。
