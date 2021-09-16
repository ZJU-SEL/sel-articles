# 多云环境下的成员集群管理，看看开源项目karmada是如何做到的


在[Gartner的一份研究报告中](https://www.gartner.com/smarterwithgartner/why-organizations-choose-a-multicloud-strategy/)，公有云用户有81%都采用了多云架构。karmada是华为开源的云原生多云容器编排平台，目标是让开发者像使用单个k8s集群一样使用多k8s集群。它的第一个release（v0.1.0）出现在2020年12月，而正式发布则是2021年4月25日在深圳召开的华为开发者大会（HDC.Cloud）2021上。

karmada吸取了CNCF社区的Federation v1和v2（也称为kubefed）项目经验与教训，在保持原有k8s资源定义API不变的情况下，通过添加与多云应用资源编排相关的一套新的API和控制面组件，方便用户将应用部署到多云环境中，实现扩容、高可用等目标。

官方网站：https://karmada.io/

代码地址：https://github.com/karmada-io/karmada

使用karmada管理的多云环境包含两类集群：

1. host集群：即由karmada控制面构成的集群，接受用户提交的应用部署需求，将之同步到成员集群，并从成员集群同步应用后续的运行状况。
1. 成员集群：由一个或多个k8s集群构成，负责运行用户提交的应用

本文描述在多云环境下karmada如何管理成员集群，包括：成员集群的注册、注销，以及成员集群的状态跟踪两个方面。

使用的karmada版本为v0.8.0后的commit：0cdba3efa。

## 1. 成员集群的注册和注销

karmada的成员集群支持push和pull两种模式：push模式下由karmada控制面的execution controller将应用推送到成员集群，而pull模式下由运行在成员集群侧的karmada agent将应用下拉到本地。这两种模式下的集群注册和注销流程略有差别，本文1.1-1.4小节将分别说明。

### 1.1. push模式成员集群的注册流程（join）

karmadactl的join命令将k8s集群以push模式注册到karmada集群联邦中。如下面的例子将一个k8s集群注册到karmada联邦，其中`member1`是`$HOME/.kube/karmada.config`中该集群的context名称，也是该集群成功加入联邦后在karmada控制面中的名称。

```sh
karmadactl join member1 --cluster-kubeconfig=$HOME/.kube/karmada.config
```

karmadactl的`JoinCluster`方法实现了join命令，流程如下：

首先，karmadactl确保`karmada-cluster` namespace在karmada控制面上已经创建。该namespace用来保存lease对象，lease对象被用来辅助集群状态的跟踪，在后续2.1-2.2小节中描述。

然后，karmadactl在`member1`集群中创建service account，并为该service account配置cluster role和cluster role binding。这个service account令karmada控制面有权将应用资源对象下发到成员集群，并有权访问成员集群apiserver的`/readyz`或`/healthz` URL，从而判断成员集群的健康状态。在`member1`集群中执行`kubectl get sa karmada-member1 -n=karmada-cluster -o yaml --context=member1`可以查看karmadactl在成员集群`member1`中创建的service account内容：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: karmada-member1
  namespace: karmada-cluster
secrets:
- name: karmada-member1-token-bddfd
```

对该service account对象说明如下：

1. servie account的名称为`karmada-`加上成员集群的名称，在这里是`karmada-member1`
1. service account默认保存在`karmada-cluster` namespace中。
1. service account中的secret `karmada-member1-token-bddfd`由成员集群自动创建

执行`kubectl get clusterrole karmada-controller-manager:karmada-member1 -o yaml --context=member1`可以查看karmadactl在`member1`集群上创建的cluster role。可以看到该cluster role允许绑定的service account执行对所有类型资源的所有操作，同时授权访问`/readyz`和`/healthz`等所有非resource类型URL的get操作。
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: karmada-controller-manager:karmada-member1
rules:
- apiGroups:
  - '*'
  resources:
  - '*'
  verbs:
  - '*'
- nonResourceURLs:
  - '*'
  verbs:
  - get
```

接着，karmadactl在karmada控制面中创建cluster对象，cluster对象是`member1`在karmada集群联邦中代表，执行`kubectl get cluster member1 -o yaml --context=karmada-apiserver`可以得如下yaml：

```yaml
kind: Cluster
metadata:
  finalizers:
  - karmada.io/cluster-controller
  name: member1
spec:
  apiEndpoint: https://172.18.0.4:6443
  secretRef:
    name: member1
    namespace: karmada-cluster
  syncMode: Push
```
cluster对象的几个重要的属性说明如下:

1. `.metadata.name`为用户在karmadactl join命令中指定的成员集群名称，在这个例子中是`member1`
1. `.spec.syncMode`为push，说明这是一个push模式的成员集群
1. `.spec.apiendpoint`为成员集群的apiserver地址，在这里是用kind创建的k8s集群的apiserver的地址
1. `.sepc.secretRef`为上一步中成员集群为service account自动创建的secrets

结合cluster对象spec中的`apiendpoint`和`secretRef`，karmada控制面就可以在`member1`中创建各种资源对象，并访问`/readyz`、`/healthz`等URL。

### 1.2. pull模式成员集群的注册流程

pull模式的成员集群注册目前还在使用`hack/deploy-karmada-agent.sh`脚本，并没有集成到karmadactl中。

用户可以执行如下命令将集群`member2`集群注册到karmada集群联邦中：

```sh
hack/deploy-karmada-agent.sh ~/.kube/karmada.config karmada-apiserver ~/.kube/karmada.config member2
```

上述命令完成后，执行`kubectl get cluster --context=karmada-apiserver`可以看到`member2`集群已经成功注册，且运行在pull模式：

```sh
NAME      VERSION   MODE   READY   AGE
member2   v1.19.1   Pull   True    32s
```

`deploy-karmada-agent.sh`脚本注册成员集群的流程与karmadactl注册（join）命令的类似，流程如下：

1. 在成员集群中创建`karmada-system` namespace
1. 在成员集群的`karmada-system` namespace中创建`karmada-agent-sa` service account
1. 在成员集群中创建cluster role，允许对成员集群所有资源的get、watch、list、create、update和delete操作，以及对`/readyz`、`/healthz`等URL的get操作
1. 在成员集群中创建cluster role binding，绑定上述service account和cluster role
1. 在成员集群的`karmada-sytem` namespace中创建名为`karmada-kubeconfig`的secret，这个secret会被mount在下一步启动的karmada agent中，用来访问karmada控制面apiserver
1. 最后，在成员集群的`karmada-system` namespace中运行karmada agent，执行`kubectl get deploy karmada-agent -n=karmada-system --context=member2`可以查看karmada agent对应的deployment。在下面的yaml中我们可以看到karmada agent用来访问karmada apiserver的secret `karmada-kubeconfig`，以及用来访问成员集群的service account `karmada-agent-sa`
```yaml
kind: Deployment
metadata:
  labels:
    app: karmada-agent
  name: karmada-agent
  namespace: karmada-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: karmada-agent
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: karmada-agent
    spec:
      containers:
      - command:
        - /bin/karmada-agent
        - --karmada-kubeconfig=/etc/kubeconfig/karmada-kubeconfig
        - --karmada-context=karmada-apiserver
        - --cluster-name=member2
        - --cluster-status-update-frequency=10s
        - --v=4
        image: swr.ap-southeast-1.myhuaweicloud.com/karmada/karmada-agent:latest
        imagePullPolicy: IfNotPresent
        name: karmada-agent
        volumeMounts:
        - mountPath: /etc/kubeconfig
          name: kubeconfig
      serviceAccountName: karmada-agent-sa
      tolerations:
      - key: node-role.kubernetes.io/master
        operator: Exists
      volumes:
      - name: kubeconfig
        secret:
          secretName: karmada-kubeconfig
```

`deploy-karmada-agent.sh`注册成员集群的流程和karmadactl join命令的流程区别在于：

1. `deploy-karmada-agent.sh`脚本不负责创建cluster对象，cluster对象的创建在karmada agent的启动过程中完成（agent的`run`函数），而且创建的cluster对象是pull模式。
1. `deploy-karmada-agent.sh`脚本在成员集群中创建的service account以及karmada-agent运行在名为`karmada-system`的namespace中，而karmadactl的join命令在成员集群中创建的service account在名为`karmada-cluster`的namespace中

### 1.3. push模式成员集群的注销流程（unjoin）

karmadactl的unjoin命令将push模式的成员集群从karmada集群联邦中注销。如下面的例子注销了`member1`成员集群。

```sh
./karmadactl unjoin member1 --cluster-kubeconfig=$HOME/.kube/karmada.config
```

karamdactl的unjoin命令处理流程定义在`UnJoinCluster`函数中，流程如下：

1. 在karmada控制面中删除成员集群对应的cluster对象
1. 在成员集群中删除karmadactl的join命令创建的rbac资源，包括cluster role、cluster role binding等
1. 在成员集群中删除karmadactl的join命令创建的service account
1. 在成员集群中删除`karmada-cluster` namespace

karmada使用[finalizer](https://kubernetes.io/docs/concepts/overview/working-with-objects/finalizers/)机制确保在karmada控制面中删除cluster对象时，先删除execution space等与之相关的其他资源。

finalizer在karmada中的基本工作原理是：

当我们创建一个cluster对象时，cluster controller将`karmada.io/cluster-controller`作为finalizer写入cluster对象的metadata。执行`kubectl get cluster member1 -o yaml --context=karmada-apiserver`查看karmadactl的join命令创建的cluster对象，可以看到它的metadata中指定cluster controller为自己的finalizer。

```yaml
kind: Cluster
metadata:
  finalizers:
  - karmada.io/cluster-controller
  name: member1
spec:
  syncMode: Push
```
当我们需要删除一个cluster对象时，cluster controller作为cluster对象的finalizer，会先删除成员集群对应的execution space（execution space是karmada控制面中的一个namespace，每个成员集群都有一个对应的execution space，用以保存需要下发到该成员集群的work等资源），如果execution space成功删除，则删除cluster对象metadata中的finalizer。否则在下个循环中重新尝试删除execution space。直到execution space删除成功，才会删除cluster metadata中的finalizer。

karmada中的work等资源也带有finalizer，执行`kubectl get work -n=karmada-es-member1 -o yaml --context=karmada-apiserver`可以看到work的finalizer是execution controller，因此删除work时也会由execution controller完成对应的finalizer流程，即在删除work对象之前先删除由work转化而成的k8s原生资源对象。

```yaml
apiVersion: v1
items:
- apiVersion: work.karmada.io/v1alpha1
  kind: Work
  metadata:
    finalizers:
    - karmada.io/execution-controller
    labels:
      resourcebinding.karmada.io/name: nginx-deployment
      resourcebinding.karmada.io/namespace: default
    name: nginx-687f7fb96f
    namespace: karmada-es-member1
  spec:
    workload:
      manifests:
      ... ...
```

### 1.4. pull模式成员集群的注销流程

karmada目前没有提供类似karmadactl unjoin的命令来注销pull模式下的成员集群。不过根据push模式下的注销流程，可以猜测如下流程应该可以工作：

1. 在karmada控制面中删除成员集群对应的cluster对象
1. 在成员集群中删除`deploy-karmada-agent.sh`脚本创建的rbac资源，包括cluster role、cluster role binding等
1. 在成员集群中删除`deploy-karmada-agent.sh`脚本创建的service account
1. 在成员集群中删除`karmada-system` namespace

在这个流程中，删除cluster、work对象同样会触发上面描述的finalizer流程，这里不再复述。

经过上述注销流程，剩下的还有karmada agent作为一个deployment运行在成员集群中，它的清理流程尚不明确。

## 2. 成员集群的状态跟踪

cluster status controller、lease controller和cluster controller共同完成成员集群的状态跟踪，并将状态写入cluster对象的status。在karmada集群联邦中，cluster status controller可能运行在两个地方：

1. 对于push模式的成员集群来说，在karamada控制面中的karamada controller manager会运行一个cluster status controller
1. 对于pull模式的成员集群来说，在成员集群侧运行的karmada agent会运行一个karmada controller manager，而后者会启动一个cluster status controller

2.1-2.2小节分别说明这两种情况下的成员集群的状态跟踪的流程。

### 2.1. push模式成员集群的状态跟踪

在karmada控制面中心运行的cluster status controller基于`sigs.k8s.io/controller-runtime`框架实现，监控karmada控制面中cluster对象的增删改事件，这些事件经过`clusterPredicateFunc`中的函数过滤，使得karmada控制面中的cluster status controller只处理push模式的cluster的增删改事件。

cluster status controller默认以10秒钟为间隔（间隔时长可以用karmada controller manager的`cluster-status-update-frequency`修改）采集成员集群的以下信息，并写入cluster对象的status中（当cluster对象发生创建或更新事件时也会触发该流程）：

1. 集群的在线状态（online）和健康状态（healthy）  
cluster status controller使用`buildClusterConfig`函数，利用成员集群的cluster对象包含的apiserver地址（`.spec.apiEndpoint`），访问成员集群使用的secret（`.spec.secretRef`）等信息构建访问成员集群的rest config和clientset。cluster status controller使用clientset访问成员集群的apiserver的`/readyz`，如果`/readyz`无法访问则使用已经被deprecate的`/healthz`。如果`/readyz`或`/healthz`能够正常访问，则表示集群在线（online状态为true），如果能够得到200的HTTP响应，则表示集群健康（healthy为true）。  
根据成员集群的在线状态和健康状态，cluster status controller设置集群状态，并写入cluster对象的`.status.conditions`：
	1. 如果集群不在线，则设置集群的condition为offline
	1. 如果成员在线，但`/readyz`或`/healthz`返回的不是HTTP 200，则集群状态为not ready
	1. 如果集群在线，且`/readyz`或`/healthz`返回的是HTTP 200，则集群状态为ready
1. 成员集群的node状态，并写入cluster对象的`.status.nodeSummary`
1. 成员集群的资源使用状态，并写入cluster对象的`.status.resourceSummary`
1. 成员集群的版本，并写入cluster对象的`.status.kubernetesVersion`
1. 成员集群的api支持情况，并写入cluster对象的`.status.apiEnablements`

执行`kubectl get cluster member1 -o yaml --context=karmada-apiserver`，可以看到一个push模式成员集群的status，由于支持的API资源（`apiEnablements`）较多，下面的yaml中只保留corev1下的部分API资源：
```yaml
apiVersion: cluster.karmada.io/v1alpha1
kind: Cluster
metadata:
  finalizers:
  - karmada.io/cluster-controller
  name: member1
spec:
  apiEndpoint: https://172.18.0.4:6443
  secretRef:
    name: member1
    namespace: karmada-cluster
  syncMode: Push
status:
  apiEnablements:
  - groupVersion: v1
    resources:
    - kind: Endpoints
      name: endpoints
    - kind: Node
      name: nodes
    - kind: Pod
      name: pods
    - kind: Service
      name: services
    ...
  conditions:
  - lastTransitionTime: "2021-09-15T11:46:42Z"
    message: cluster is reachable and health endpoint responded with ok
    reason: ClusterReady
    status: "True"
    type: Ready
  kubernetesVersion: v1.19.1
  nodeSummary:
    readyNum: 1
    totalNum: 1
  resourceSummary:
    allocatable:
      cpu: "4"
      ephemeral-storage: 49805880Ki
      hugepages-1Gi: "0"
      hugepages-2Mi: "0"
      memory: 8147848Ki
      pods: "110"
    allocated:
      cpu: 850m
      memory: 190Mi
    allocating:
      cpu: "0"
      memory: "0"
```

从上面的cluster对象的yaml中，可以看到当前集群状态是ready（`/readyz`返回HTTP 200），该状态的转变时间（transition time）为`2021-09-15T11:46:42Z`，这个时间戳要在集群状态发生变化时才会更新。

cluster status controller还会启动一个lease controller（`initLeaseController`方法），定时renew一次lease，作用类似心跳检测。

但是，对push模式的成员集群来说，只要cluster status controller正常运行，那么它就一直能够尝试访问成员集群apiserver的`/readyz`或`/healthz`，无论请求成功还是失败都能据此判断成员集群的状态。lease controller和cluster controller在push模式下对判断集群状态并无太大作用，它们的作用要在pull模式下才能发挥。


### 2.2. pull模式成员集群的状态跟踪

在pull模式中，cluster status controller运行在成员集群侧，由karmada agent启动，但它使用不同的`PredicateFunc`，从而仅处理本成员集群的cluster对象增删改事件。cluster status controller同样周期性采集集群状态，并写入cluster对象的status，其流程和2.1小节中描述的类似，这里不再复述。

但pull模式下的集群状态跟踪仅依靠cluster status controller是不够的，还离不开lease controller和cluster controller的配合。

我们假设这样的场景：如果由于网络问题，运行在成员集群侧的karmada agent长期无法向karmada控制面上报集群最新状态，karmada控制面虽然能够获取karmada agent最近一次上报的集群状态，但无法判断当前集群依然处于之前的状态，还是已经失联。

这种情况下成员集群的状态判断就需要依赖运行在karmada agent中的lease controller和运行在karmada控制面中的cluster controller。lease controller类似一种心跳机制，它定期renew karmada控制面上的一个lease对象（存在`karmada-cluster` namespace中），一旦发生网络问题，lease controller就无法及时renew lease对象。karmada控制面中运行的cluster controller以5秒为间隔（间隔时长可以用karmada controller manager的`cluster-monitor-period` flag修改）运行`monitorClusterHealth`方法，当cluster controller发现某个集群的lease长期没有renew时，就将该集群的状态设置为未知状态（unknown）。
