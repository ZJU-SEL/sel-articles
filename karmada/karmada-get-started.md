# karmada上手指南

华为的karmada项目第一个release（v0.1.0）出现在2020年12月，而其正是发布则是在2021年4月25日，在深圳召开的华为开发者大会（HDC.Cloud）2021上，华为云CTO张宇昕在主题演讲中正式宣布其开源。

karmada吸取了CNCF社区的Federation v1和v2（也称为kubefed）项目经验与教训，在保持原有k8s应用资源定义API不变的情况下，通过添加与分布式应用部署管理相关的一套新的API和控制面组件，方便用户将应用部署到多云环境中，实现应用扩容、高可用等目标。

官方网站：https://karmada.io/

代码地址：https://github.com/karmada-io/karmada

使用karmada管理的多云环境包含两类集群：
1. host集群：即由karmada控制面构成的集群，接受用户提交的工作负载部署需求，并将之同步到member集群
1. member集群：由一个或多个k8s集群构成，负责运行用户提交的工作负载

本文描述karmada的上手流程，使用的karmada版本为v0.7.0后的commit：c4835e1f。

## 1. karmada安装

### 1.1. 安装docker

按照docker官网[文档](https://docs.docker.com/engine/install/debian/)在本机安装docker，对debian来说流程如下：

1. 安装基础工具
	```sh
	 sudo apt-get update
	 sudo apt-get install \
		apt-transport-https \
		ca-certificates \
		curl \
		gnupg \
		lsb-release
	```
1. 安装docker的gpg key：
	```sh
	curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
	```
1. 安装docker源
	```sh
	echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
	```
1. 安装docker
	```sh
	apt-get update
	sudo apt-get install docker-ce docker-ce-cli containerd.io
	```
### 1.2. 安装简易k8s开发集群管理工具：kind

> kind官网对其描述：kind is a tool for running local Kubernetes clusters using Docker container “nodes”. kind was primarily designed for testing Kubernetes itself, but may be used for local development or CI.

在本地已经安装好go (1.11以上) 和docker的情况下，执行下面指令安装kind：

```sh
GO111MODULE=on GOPROXY=goproxy.cn go get sigs.k8s.io/kind@v0.11.1
```

> 0.11.1是当前kind最新的release的，也是后续部署karmada过程中指定要用的版本

kind使用一个容器来模拟一个node，在容器内部使用systemd托管kubelet和containerd（不是docker），然后通过被托管的kubelet启动其他k8s组件，比如kube-apiserver、etcd、CNI等跑起来。

后续部署karmada环境时会调用kind创建好一个单node的k8s集群，可以通过执行`docker exec karmada-host-control-plane ctr --namespace k8s.io containers ls`查看该容器内部运行的作为“嵌套容器”运行的k8s组件的运行情况：如kube-controller-manager、etcd、kube-apiserver、kube-proxy、kube-scheduler等

### 1.3. 启动本地k8s集群，安装karmada控制面

1. 确保已经安装make、gcc工具
1. 确保已经安装kubectl，可以参考[官方文档](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)采用手动安装或包管理器的方式安装
1. clone karmada源码：
	`git clone https://github.com/karmada-io/karmada.git karmada-io/karmada`
1. `cd karmada-io/karmada`
1. `hack/local-up-karmada.sh`，这里包含一系列步骤：  
	1. 检查go、kind等工具是否已经存在
	1. 调用kind创建host k8s集群，集群版本默认为1.19.1，与karmada重用的k8s组件（kube-apiserver、kube-controllermanager）版本一致
		> 创建k8s集群需要的kindest/node:v1.19.1镜像较大，可以提前下载好，防止后续的`local-up-karmada`等待集群启动超时（默认5分钟）
	1. build karmada控制面可执行文件及容器镜像，包括karmada-controller-manager、karmada-scheduler、karmada-webhook。build结束后本地可以找到如下镜像：
		```sh
		REPOSITORY                                                                TAG       IMAGE ID       CREATED              SIZE
		swr.ap-southeast-1.myhuaweicloud.com/karmada/karmada-agent                latest    e708d0c5b514   30 seconds ago       65.1MB
		swr.ap-southeast-1.myhuaweicloud.com/karmada/karmada-webhook              latest    bc2890d489c0   56 seconds ago       60.5MB
		swr.ap-southeast-1.myhuaweicloud.com/karmada/karmada-scheduler            latest    617603429a89   About a minute ago   63.3MB
		swr.ap-southeast-1.myhuaweicloud.com/karmada/karmada-controller-manager   latest    b3a7b2254394   About a minute ago   66.1MB
		```
	1. 部署karmada控制面组件到host集群
	1. 创建CRD
		```
		customresourcedefinition.apiextensions.k8s.io/propagationpolicies.policy.karmada.io created
		customresourcedefinition.apiextensions.k8s.io/clusterpropagationpolicies.policy.karmada.io created
		customresourcedefinition.apiextensions.k8s.io/overridepolicies.policy.karmada.io created
		customresourcedefinition.apiextensions.k8s.io/clusteroverridepolicies.policy.karmada.io created
		customresourcedefinition.apiextensions.k8s.io/replicaschedulingpolicies.policy.karmada.io created
		customresourcedefinition.apiextensions.k8s.io/works.work.karmada.io created
		customresourcedefinition.apiextensions.k8s.io/resourcebindings.work.karmada.io created
		customresourcedefinition.apiextensions.k8s.io/clusterresourcebindings.work.karmada.io created
		customresourcedefinition.apiextensions.k8s.io/serviceexports.multicluster.x-k8s.io created
		customresourcedefinition.apiextensions.k8s.io/serviceimports.multicluster.x-k8s.io created
		```
	1. 创建webhook：
		```
		mutatingwebhookconfiguration.admissionregistration.k8s.io/mutating-config created
		validatingwebhookconfiguration.admissionregistration.k8s.io/validating-config created
		```
	1. 部署完成后，形成kubeconfig文件`$HOME/kube/karmada.config`，其中定义如下context
		  | Context Name      | Purpose                        |
		  |-------------------|--------------------------------|
		  | karmada-host      | the cluster karmada install in |
		  | karmada-apiserver | karmada control plane          |


> 注意：karmada还提供了remote-up-karmada.sh脚本，用以把一个现有的k8s集群加入联邦。感兴趣的读者可以阅读karmada项目的readme尝试

## 2. karmada控制面构成

部署karmada完成后，在切换到`karmada-host` context后，执行`kubectl get po --all-namespaces`可以得到已经部署的组件列表：

```sh
NAMESPACE            NAME                                                 READY   STATUS    RESTARTS   AGE
karmada-system       etcd-0                                               1/1     Running   0          98m
karmada-system       karmada-apiserver-75b5dc6fb7-l6hdv                   1/1     Running   0          98m
karmada-system       karmada-controller-manager-7d66968445-nnnpp          1/1     Running   0          98m
karmada-system       karmada-kube-controller-manager-5456fd756d-sf9xk     1/1     Running   0          98m
karmada-system       karmada-scheduler-7c8d678979-bgq4f                   1/1     Running   0          98m
karmada-system       karmada-webhook-5bfd9fb89d-msqnw                     1/1     Running   0          98m
kube-system          coredns-f9fd979d6-4bc2l                              1/1     Running   0          99m
kube-system          coredns-f9fd979d6-s7jc6                              1/1     Running   0          99m
kube-system          etcd-karmada-host-control-plane                      1/1     Running   0          99m
kube-system          kindnet-cq6kv                                        1/1     Running   0          99m
kube-system          kube-apiserver-karmada-host-control-plane            1/1     Running   0          99m
kube-system          kube-controller-manager-karmada-host-control-plane   1/1     Running   0          99m
kube-system          kube-proxy-ld9t8                                     1/1     Running   0          99m
kube-system          kube-scheduler-karmada-host-control-plane            1/1     Running   0          99m
local-path-storage   local-path-provisioner-78776bfc44-d9fvv              1/1     Running   0          99m
```
可以看到部署在两个namespace中的两个k8s控制面：

`karmada-host` context对应的控制面运行在`kube-system` namespace中，用来运行管理karmada控制面，是个由kind启动的标准的k8s管理节点。

`karmada-apiserver` context对应的控制面运行在`karmada-system` namespace中，是karmada控制面（对应readme中提到的host集群）。它由`local-up-karmada.sh`脚本部署到`karmada-host`集群中，包行两个部分：
1. 重用了k8s的两个组件：`kube-apiserver`和`kube-controllermanager`以及etcd
1. 其他3个为karmada组件，包括`kamada-controller-manager`、`karmada-scheduler`、`karmada-webhook`


前一个k8s集群只是为了支撑karmada控制面的运行。所有后续集群联邦相关的操作，包括用karmadactl发出的member集群管理请求，以及用kubectl发出的工作负载管理请求都发往karmada控制面。这些请求中创建的API资源也保存在karmada控制面自身的etcd中（对应上面列表中的`etcd-0` pod）。

需要注意的是，虽然karmada控制面重用了部分k8s组件，被重用的kube-controller-mamager通过启动flag限制其只运行namespace、garbagecollector、serviceaccount-token、serviceaccount这几个controller，所以当用户把`Deployment`等k8s标准资源提交给karmada apiserver时，它们只是被记录在karmada控制面的etcd中，并在后续同步到member集群中，这些`Deployment`资源并不会在karmada控制面管理的集群中发生reconcile（如创建pod）。

## 3. karmada使用

用户可以用karmadactl和kubectl两个cli使用karmada，其中：
1. karmadactl用来执行member集群的加入(joint)/离开（unjoin）、标志一个member集群不可调度（cordon）或解除不可调度的标志（uncordon）
1. kubectl用来向karmada集群提交标准的k8s资源请求，或由karmada定义的CR请求。用以管理karmada集群中的工作负载。

### 3.1 使用karmadactl管理member集群

1. 执行`hack/create-cluster.sh member1 $HOME/.kube/karmada.config`创建新的集群member1
1. 执行`kubectl config use-context karmada-apiserver`切换到karmada控制面
1. 执行`karmadactl join member1 --cluster-kubeconfig=$HOME/.kube/karmada.config`以push的方式把member1加入karmada集群  
	> 注意：如果还没有编译过karmadactl可执行文件，可以在代码根目录执行make karmadactl  
	> 注意：karmada中的host集群和member集群之间的同步方式有push和pull两种，这里的上手过程采用push的方式，感兴趣的读者可以参考karmada的readme尝试pull同步方式

目前karmadactl没有list member集群的功能，对于已经加入的member集群，可以通过获取`Cluster`类型的CR实现：
```sh
kubectl get clusters
```
得到输出为：
```sh
NAME      VERSION   MODE   READY   AGE
member1   v1.19.1   Push   True    88s
```

> 上面的create-cluster.sh脚本默认创建最新版的k8s集群，为了避免再次拉下一个大镜像，可以修改create-cluster.sh脚本，为kind create cluster命令加上`--image="kindest/node:v1.19.1"`参数

### 3.2 使用kubectl管理工作负载

karmada代码里自带一个nginx应用可以用来体验基于karmada的分布式工作负载管理：

1. 执行`kubectl config use-context karmada-apiserver`切换到karmada控制面
1. 执行`kubectl create -f samples/nginx/deployment.yaml`创建deployment资源  
	如前面所述，由于kamada控制面没有部署deployment controller，nginx不会在karmada控制面所在集群跑起来，而是仅仅保存在etcd里  
	这时候如果去member1集群查看pod资源的情况，可以发现nginx也没有在member1集群中运行起来
1. 执行`kubectl create -f samples/nginx/propagationpolicy.yaml`，定义如下的propgation policy：
	```yaml
    apiVersion: policy.karmada.io/v1alpha1
    kind: PropagationPolicy
    metadata:
      name: nginx-propagation
    spec:
      resourceSelectors:
        - apiVersion: apps/v1
          kind: Deployment
          name: nginx
      placement:
        clusterAffinity:
          clusterNames:
            - member1
	```
	这个progation policy将之前部署的nginx deployment资源（由`resourceSelectors`指定）同步到member1集群（由`placement`指定）中
1. 这时不用切换到member1 context，对karmada控制面执行`kubectl get deploy`可以看到名叫nginx的deployment已经正常运行：
	```sh
	NAME    READY   UP-TO-DATE   AVAILABLE   AGE
	nginx   1/1     1            1           21m
	```
	上述结果说明karmada有能力从member集群同步工作负载状态到host集群。作为验证，我们可以切换到member1集群，执行`kubectl get po`可以看到deployment对应的nginx pod已经在member1集群内正常运行：
	```sh
	NAME                     READY   STATUS    RESTARTS   AGE
	nginx-6799fc88d8-7tgmb   1/1     Running   0          8m27s
	```

## 4. 结尾并非结束

在[Gartner的一份研究报告中](https://www.gartner.com/smarterwithgartner/why-organizations-choose-a-multicloud-strategy/)，公有云用户有81%都采用了多云架构。近年来蓬勃发展的云原生社区对多云挑战给也几次给出自己的思考和解决方案，远有CNCF社区sig multicluster提出的Federation v1和v2，近有华为开源的karmada以及Red Hat开源的Open Cluster Management（OCM）。虽然尚未在API层面达成一致，但各开源项目都在吸取前人的经验教训的基础上优化演进。百家争鸣而非闭门造车，这正是开源的魅力所在。

后续我们会对karmada项目进行更为深入的分析。
