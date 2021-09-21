# 多云环境下的资源调度：karmada scheduler的框架和实现

karmada是华为开源的云原生多云容器编排平台，目标是让开发者像使用单个k8s集群一样使用多k8s集群。它的第一个release（v0.1.0）出现在2020年12月，而正式发布则是2021年4月25日在深圳召开的华为开发者大会（HDC.Cloud）2021上。

karmada吸取了CNCF社区的Federation v1和v2（也称为kubefed）项目经验与教训，在保持原有k8s资源定义API不变的情况下，通过添加与多云应用资源编排相关的一套新的API和控制面组件，方便用户将应用部署到多云环境中，实现扩容、高可用等目标。

官方网站：https://karmada.io/

代码地址：https://github.com/karmada-io/karmada

使用karmada管理的多云环境包含两类集群：

1. host集群：即由karmada控制面构成的集群，接受用户提交的应用部署需求，将之同步到成员集群，并从成员集群同步应用后续的运行状况。
1. 成员集群：由一个或多个k8s集群构成，负责运行用户提交的应用

本文分析karmada scheduler的框架和实现。

相对于工作在单个k8s集群层面的kube-scheduler，karmada scheduler工作在多云层面，负责从集群联邦中选择一部分成员集群，运行用户提交的k8s原生API资源对象（包括CRD资源）。karmada scheduler的工作更为宏观，在多云调度过程中karmada scheduler不会考虑容器端口是否会跟其他容器端口冲突，不会考虑需要的volume是否已经到位，也不会考虑容器镜像在服务器上是否已经存在等细节，这些问题交给kube-scheduler。

使用的karmada版本为v0.8.0后的commit：fd89085。

## 1. 是他不是他，两个不同层面的调度器

在本karmada源码分析系列文章中的第一篇《karmada上手指南》中，我们提到使用`hack/local-up-karmada.sh`将karmada控制面部署到一个k8s集群上，部署完成后，我们就有了两个控制面：

1. 一个k8s控制面，用来运行karmada控制面，称为karmada-host
1. 一个karmada控制面，称为karmada-apiserver。

执行`kubectl get po -A --context=karmada-host`，我们可以得到已经部署的组件列表：

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

其中我们可以看到两个调度器：在`kube-system` namespace中的`kube-scheduler-karmada-host-control-plane`，和在`karmada-system` namespace中的`karmada-scheduler-7c8d678979-bgq4f`。

其中`kube-system` namespace中的是标准的k8s调度器kube-scheduler，它在这里的任务相对简单，只是负责将以deployment形式部署的karmada众多组建调度到karmada-host集群上。

而在`karmada-system` namespace中的karamda-scheduler则是工作在更高一个层面，也就是集群联邦层面上的多云调度器。其目的是将用户提交给karmada控制面的应用调度到karamda管理的成员集群中，所谓的应用包括了deployment等各种k8s原生API资源，以及CRD资源。

## 2. karmada scheduler框架

karmada scheduler是一个单独的二进制文件，基于cobra框架实现。它的初始化过程调用`NewScheduler`创建karmada scheduler对象，并使用shared informer监听resource binding、propagation policy和cluster资源的增删改事件。在leader election中选举成功的karmada scheduler实例会运行自己的`Run`方法，这个方法在resource binding资源首次同步成功之后，在单独的goroutine中运行`worker`方法。

> resource binding和propagation policy都存在cluster版本的资源：cluster resource binding和cluster propagation policy。为了简化karmada scheduler的分析，本文仅讨论非cluster版本

和k8s中的kube-scheduler一样，karmada scheduler也使用扩展点（extension point）的方式管理多种调度算法插件。相对于kube-scheduler众多的扩展点（pre filter、filter、post filter、pre score、score、post score等），karmada scheduler目前只有filter和score两个扩展点，调度过程和kube-scheduler类似，也是先filter（筛选）再score（优选）。filter扩展点上的调度算法插件都应实现`FilterPlugin`接口（包含一个`Filter`方法），score扩展点上的调度算法插件都应实现`ScorePlugin`接口（包含一个`Score`方法）。

karmada scheduler在调度每个k8s原生API资源对象（包含CRD资源）时，会逐个调用各扩展点上的插件：

1. filter扩展点上的调度算法插件将不满足propagation policy的成员集群过滤掉  
karmada scheduler对每个考察中的成员集群调用每个插件的`Filter`方法，该方法都能返回一个`Result`对象表示该插件的调度结果，其中的`code`代表待下发资源是否能调度到某个成员集群上，`reason`用来解释这个结果，`err`包含调度算法插件执行过程中遇到的错误。
1. score扩展点上的调度算法插件为每个经过上一步过滤的集群计算评分  
karmada scheduler对每个经过上一步过滤的成员集群调用每个插件的`Score`方法，该方法都能返回一个`float64`类型的评分结果

最终按照第二步的评分高低选择成员集群作为调度结果。

目前karmada的调度算法插件只有cluster affinity、taint toleration和api installed三个。虽然这三个调度算法插件都同时实现了`FilterPlugin`和`ScorePlugin`接口，但它们计算的score被硬编码为0，所以实际上并不具备score plugin的作用。

相对于kube-scheduler能够方便地使用文件或configmap开启关闭调度算法插件，目前karmada scheduler中算法插件的开关还是硬编码在代码里：`NewScheduler`函数调用`NewGenericScheduler`函数时，在传入的参数中打开了所有的三个调度算法插件。

## 3. karmada scheduler的各种工作场景

从工作顺序上讲，karmada scheduler工作在resource detector后，也就是要等到resource detector绑定propagation policy和k8s原生API资源对象（包括CRD资源）之后才介入。所以karamda scheduler的输入是resource detector的输出：resource binding。对resource detector、resource binding等概念感兴趣的读者可以查看本karmada源码分析系列文章中的上一篇文章《从karmada API角度分析多云环境下的应用资源编排：设计与实现》。

当karmada scheduler的worker逐一处理内部队列`queue`中的resource binding的更新事件时（这些事件由karmada scheduler定义的不同list/watch handler加入`queue`中），这些resource binding对象可能处于以下几种状态，这些不同的状态决定了karmada scheduler下一步处理流程：

1. 首次调度（`FirstSchedule`）：这个阶段的resource binding刚由resource detector的绑定工作创建出来。从未经过karmada scheduler的调度处理。这类resource binding对象的特征是`.spec.clusters`为空
1. 调和调度（`ReconcileSchedule`）：当用户更新了propagation policy的placement，为了使得系统的实际运行状态与用户的期望一致，karmada scheduler不得不将之前已经调度过的k8s原生API资源对象（包括CRD资源）重新调度到新的成员集群中（从这个角度讲，下面的扩缩容调度也是一种调和调度，调和是在k8s中普遍使用的概念，因此“调和调度”这个名字范围太广，含义不明确）。这类resource binding对象的特征是之前已经通过karmada scheduler的调度，即`.spec.clusters`不为空，且之前已经绑定的propagation policy最新的placement不等于之前调度时的placement。
1. 扩缩容调度（`ScaleSchedule`）：当propagation policy包含的replica scheduling strategy与集群联邦中实际运行的replica数量不一致时，需要重新调度之前已经完成调度的k8s原生API资源对象（包括CRD资源）
1. 故障恢复调度（`FailoverSchedule`）：当上次调度结果中的成员集群发生故障，也就是resource binding的`.spec.clusters`包含的成员集群状态不全都是就绪（ready），karmada scheduler需要重新调度应用，以恢复集群故障带来的应用故障
1. 无需调度（`AvoidSchedule`）：当上次调度结果中的成员集群状态均为就绪（ready），也就是resource binding的`.spec.clusters`包含的成员集群状态全都是就绪（ready），则无需做任何调度工作，只是在日至中记录告警信息：Don't need to schedule binding。

3.1-3.3小节就上面几种情况分别描述karmada scheduler的工作流程。

### 3.1. 首次调度和调和调度

karmada scheduler使用自身的`getTypeFromResourceBindings`方法判定从`queue`中得到的resource binding的所处状态。

当resource binding的`.spec.clusters`为空，就表示该resource binding资源从未经过调度，这时karmada scheduler判定它的状态为首次调度。

当resource binding的`.spec.cluster`不为空，但是之前绑定的propagation policy的placement发生了变化，则karmada scheduler判定它的状态为调和调度。karmada scheduler是得知如何上一轮调度中propagation policy的placement？原来karmada scheduler在上一次调度完成后，把当时的propagation policy的placement记在了resource binding的annotations中。比如下面的yaml是一个经过karmada scheduler调度的resource binding对象：

```yaml
apiVersion: work.karmada.io/v1alpha1
kind: ResourceBinding
metadata:
  annotations:
    policy.karmada.io/applied-placement: '{"clusterAffinity":{"clusterNames":["member1","member2"]}}'
  labels:
    propagationpolicy.karmada.io/name: nginx-propagation
    propagationpolicy.karmada.io/namespace: default
    manager: karmada-controller-manager
  name: nginx-deployment
  namespace: default
spec:
  clusters:
  - name: member1
  - name: member2
  replicaRequirements:
    resourceRequest:
      cpu: "0"
      ephemeral-storage: "0"
      memory: "0"
      pods: "0"
  replicas: 1
  resource:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
    namespace: default
    resourceVersion: "9439"
```

可以看到这个resource binding对象中，有`propagationpolicy.karmada.io/name`和`propagationpolicy.karmada.io/namespace`两个label标记了绑定的是哪个propagation policy（在这里是`default` namespace中的`nginx-propagation`）。在`policy.karmada.io/applied-placement`这个annotations中还标记了当时的placment：`'{"clusterAffinity":{"clusterNames":["member1","member2"]}}'`。

所以如果用户修改了`default` namespace中的`nginx-propagation` propagation policy的placement，比如在应用的部署目标集群`member1`和`member2`中增加了`member3`，karmada scheduler就会检测到这个变化，并判定该resource binding处于调和调度状态。

首次调度和调和调度采用同样的处理流程（karmada scheduler的`scheduleOne`方法），所以在本小节中一起说明。

`scheduleOne`方法把主要的调度逻辑交给generic scheduler，`scheduleOne`剩下的逻辑就比较简单：在调用generic scheduler的`Schedule`方法后，得到目标集群`ScheduleResult`，将`ScheduleResult`写入resource binding的`.spec.cluster`中，并将当前的placement写入resource binding的`policy.karmada.io/applied-placement` annotation中。

现在我们看generic scheduler的调度逻辑是怎样的。

首先，调用generic scheduler的`findClustersThatFit`方法，调用所有filter扩展点上的调度算法插件（`RunFilterPlugins`方法），将无法调度待下发资源（如前面yaml中的nginx deployment）成员集群过滤掉。当前支持cluster affinity、taint toleration和api installed三个调度算法插件。其中：

1. cluster affinity插件按照cluster affinity过滤集群，具体可以用label selector、cluster name和field selector三种方式。上面的例子中的`nginx-propagation`使用了cluster name方法，直接指定了目标成员集群的名字`member1`和`member2`
1. taint toleration插件测试placment中指定的toleration是否可以满足某个成员集群上定义的taint
1. api installed插件检查成员集群的API是否支持待下发的k8s原生API资源对象（包括CRD资源）。由于同一种资源在k8s中可以属于不同的group和version，不同版本的k8s集群对API的支持又不尽相同，所以当用户提交了一个需要下发到成员集群中的资源，不见得每个成员集群都能支持。在karmada中，每个成员集群的cluster对象的`.status.apiEnablements`中都记录了该集群支持的API资源，因此只要将待下发资源与成员集群的`apiEnablements`对比即可

然后，调用generic scheduler的`prioritizeClusters`方法，调用所有score扩展点上的调度算法插件（`RunScorePlugins`方法），计算经过上一步过滤的成员集群列表中每个成员集群的得分。当前所有的调度算法插件都返回0，因此不具有实际上的优选作用。

接着，调用generic scheduler的`selectClusters`方法，出于高可用的目标，将资源尽可能分散在多个成员集群小组上。propagation policy支持多个spread constraint。spread constraint可以根据成员集群的某个属性（当前仅支持cluster、后续可能增加对成员集群region、zone、provider等属性支持）将集群联邦中的成员集群分为多个小组，也可以根据label将成员集群分为多个小组，在满足spread constraint的`maxGroups`和`minGroups`的前提下，`selectClusters`方法将资源尽可能分散到多个小组，从而实现应用高可用。

最后，调用generic scheduler的`assignReplicas`方法，根据replica scheduling strategy调整每个目标成员集群上replica的数量。`assignReplicas`方法返回的`TargetCluster`不仅包含了调度结果是哪些成员集群，还为每个成员集群分配了replica数量。

所谓的replica scheduling strategy是指：在propagation policy中定义placement时，同时指定replica scheduling strategy。下面我们举例说明replica scheduling strategy的使用方法。

假定当前karmada集群联邦中有两个成员集群：`member1`和`member2`，我们需要将一个repolica为3的nginx deployment部署到`member1`和`member2`。其中部署在`member1`的replica为总replica数量的2/3，部署在`member2`上的replica为总replica数量的1/3。

首先我们定义如下的nginx deployment，replica数量为3：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: default
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
```

然后我们定义如下的propagation policy：

```yaml
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: nginx-propagation
  namespace: default
spec:
  resourceSelectors:
    - apiVersion: apps/v1
      kind: Deployment
      name: nginx
  placement:
    clusterAffinity:
      clusterNames:
        - member1
        - member2
    replicaScheduling:
      replicaSchedulingType: Divided
      replicaDivisionPreference: Weighted
      weightPreference:
        staticWeightList:
          - targetCluster:
              clusterNames:
                - member1
            weight: 2
          - targetCluster:
              clusterNames:
                - member2
            weight: 1
```

上面的propagation policy包含了一个replica scheduling strategy（`.spec.placement.replicaScheduling`），对它说明如下：

1. 该replica scheduling policy的类型为“切分”，也就是将nginx的deployment的replica数量切分到多个成员集群上。  
karmada scheduler支持两种类型的replica scheduling strategy：“切分”（divided）和“复制”（duplicated），可以通过设置`.spec.placment.replicaScheduling.replicaSchedulingType`为`Divided`或`Duplicated`来选择。在上面例子中的“切分”策略将原本deployment资源的replica数量3切分到个成员集群中。如果选择的是`Duplicated`，则将待下发的deployment对象的replia数量复制到成员集群中。
1. 切分的时候按照权重设定切分比列  
这里设置`.spec.placment.replicaScheduling.replicaDivisionPreference`为`Weighted`，意思是切分replia的时候，以各成员集群的权重切分
1. 各成员集群的权重是2：1（int类型）  
也就是按照2:1的权重将原本的deployment的replica分配到`member1`和`member2`两个成员集群上。

将上述的deployment和propagation policy通过`kubectl apply`提交给karmada apiserver后，执行`kubectl get rb nginx-deployment -n default -o yaml --context=karmada-apiserver`可以查看karamda scheduler更新过的包含调度结果的resource binding。可以看到`.spec.cluster`中，分配到`member1`集群上的replica数量是2，分配到`member2`集群上的replia数量是1，符合replica总数是3，按照2：1的权重分配的设定。

```yaml
apiVersion: work.karmada.io/v1alpha1
kind: ResourceBinding
metadata:
  annotations:
    policy.karmada.io/applied-placement: '{"clusterAffinity":{"clusterNames":["member1","member2"]},"replicaScheduling":{"replicaSchedulingType":"Divided","replicaDivisionPreference":"Weighted","weightPreference":{"staticWeightList":[{"targetCluster":{"clusterNames":["member1"]},"weight":2},{"targetCluster":{"clusterNames":["member2"]},"weight":1}]}}}'
  creationTimestamp: "2021-09-21T05:17:24Z"
  generation: 7
  labels:
    propagationpolicy.karmada.io/name: nginx-propagation
    propagationpolicy.karmada.io/namespace: default
    manager: karmada-controller-manager
  name: nginx-deployment
  namespace: default
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: Deployment
    name: nginx
    uid: 7e2f4ad5-b04b-448b-a249-4340c7a90fcc
spec:
  clusters:
  - name: member1
    replicas: 2
  - name: member2
    replicas: 1
  replicaRequirements:
    resourceRequest:
      cpu: "0"
      ephemeral-storage: "0"
      memory: "0"
      pods: "0"
  replicas: 3
  resource:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
    namespace: default
    resourceVersion: "9439"
```

当前karamda scheduler仅支持deployment的replica scheduling policy，未来可能支持statefulset等其他带replica属性的k8s原生API资源类型（resource detector的`GetReplicaDeclaration`方法会把需要下发到成员集群的deployment资源的replica抽取出来，放在resource binding的`.spec.replicas`里）。

### 3.2. 扩缩容调度

karmada scheduler使用自身的`getTypeFromResourceBindings`方法判定从`queue`中得到的resource binding的所处状态。

当resource binding的`.spec.cluster`不为空，且propagation policy包含的replica scheduling strategy与集群联邦中实际运行的replica数量不一致时，karmada scheduler判定resource binding的状态为扩缩容调度。karmada scheduler调用`IsBindingReplicasChanged`函数做出上述判断，具体有两个判断条件：

1. 如果replica scheduling strategy的类型为`Duplicated`，表示各目标成员集群上应该运行的replica数量应该等于resource binding的`.spec.replica`。如果这两者不等，则karamda scheduler判定为扩缩容调度
1. 如果replica scheduling strategy的类型为`Divided`，表示各成员集群上运行的replica数量之和应该等于resource binding的`.spec.replica`。如果这两者不等，则karmada scheduler判定为扩缩容调度

一旦karmada scheduler判断为扩缩容调度，则调用自身的`scaleScheduleOne`方法，其流程与前面的`assignReplicas`方法类似，这里不再复述。

### 3.3. 故障恢复调度

karmada scheduler使用自身的`getTypeFromResourceBindings`方法判定从`queue`中得到的resource binding的所处状态。

当resource binding的`.spec.cluster`不为空，且上次调度结果中的部分成员集群发生故障（即集群状态不是就绪，对karmada中集群状态管理感兴趣的读者推荐阅读本karmada源码分析系列文章中的《多云环境下的成员集群管理，开源项目karmada是如何做到的》），也就是resource binding的`.spec.clusters`包含的成员集群状态不全都是就绪，需要重新调度应用，以恢复集群故障带来的应用故障。

一旦karmada scheduler判断为故障恢复调度，则调用自身的`rescheduleOne`方法。该方法用当前状态为就绪但不包含在之前调度结果中的集群来替换发生故障的集群：

1. 获取集群联邦中当前所有就绪的集群：ready clusters
1. 从resource binding的`.spec.cluster`中获取之前调度结果中的集群列表：total clusters
1. 计算之前调度结果的集群中当前依然健康的集群：reserved clusters
1. 从ready cluster中去除totol cluster，得到当前状态为就绪，但不包含在之前调度结果中的集群：available clusters，可以考虑用这些集群去补充之前调度结果中发生故障的集群
1. 用filter扩展点上的所有调度算法插件过滤available clusters，把其中不满足filter要求的集群去除，得到candidate clusters
1. 如果发生故障的集群数量大于candidate clusters，且replica scheduling strategy为空或`Duplicated`，则此时候选集群的数量不足以弥补发生故障的集群，无法完成故障恢复，于是终止故障恢复调度
1. 将candidate clusters中的集群补充到reserved clusters中，完成故障恢复调度
