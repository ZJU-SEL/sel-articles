# Cloudhub源码分析

## 引文

本文基于KubeEdge官方文档，加上作者的亲身实践，给出以下KubeEdge1.3.0版本下Cloudcore模块中Cloudhub模组的源码分析。

## 总览

CloudHub是cloudcore的一个模块，是Controller和Edge端之间的中介。它负责下行分发消息(其内封装了k8s资源事件，如pod update等)，也负责接收并发送边缘节点上行消息至controllers。其中下行的消息在应用层增强了传输的可靠性，以应对云边的弱网络环境。

到边缘的连接（通过EdgeHub模块）是通过可选的websocket/quic连接完成的。对于Cloudcore内部通信，Cloudhub直接与Controller通讯。Controller发送到CloudHub的所有请求，与用于存储这个边缘节点的事件对象的通道一起存储在channelq中。

![Cloudhub-1](./images/Cloudhub-1.png)

Cloudhub内部主要有以下几个重要结构：

- MessageDispatcher：下行消息分发中心，也是下行消息队列的生产者，DispatchMessage函数中实现。

- NodeMessageQueue，每个边缘节点有一个专属的消息队列，总体构成一个队列池，以Node UID作为区分，ChannelMessageQueue结构体实现
- WriteLoop，负责将消息写入底层连接，是上述消息队列的消费者

- Connection server，接收边缘节点访问，支持websocket协议和quick协议连接

- Http server：为边缘节点提供证书服务，如证书签发与证书轮转

![Cloudhub-2](./images/Cloudhub-2.png)

云边消息格式的结构如下：

- Header，由beehive框架调用NewMessage函数提供，主要包括ID、ParentID、TimeStamp
- Body，包含消息源，消息Group，消息资源，资源对应的操作

## 源码分析

### 入口

Cloudhub在Cloudcore启动时注册，通过beehive消息通信框架调用Start()函数启动Cloudhub模块。

```go
cloudhub.Register(c.Modules.CloudHub, c.KubeAPIConfig)
```

### 启动流程

Cloudhub首先启动ObjectSyncController，ObjectSyncController是自定义的CRD对象ObjectSync和ClusterObjectSync的控制器，KubeEdge使用K8s CRD存储已成功发送到Edge的资源的最新resourceVersion。当cloudcore重新启动或正常启动时，它将检查resourceVersion以避免发送旧消息。ClusterObjectSync用于保存群集范围的对象，而ObjectSync用于保存命名空间范围的对象。 它们的名称由相关的节点名称和对象UUID组成。当前版本的ClusterObjectSync当前还未使用，属于代码编写阶段，未来会扩充。

接下来，自动生成CA和CloudCore的证书用于后面安全通讯。再生成token，作为是否给EdgeCore签发证书的凭据。StartHTTPServer()启动服务器监听，主要用于EdgeCore申请证书，它将等待edgecore发来请求，获取证书。

然后，启动cloudhub服务，具体的操作是启动一个服务器，等待edgecore发来连接的请求，协议可以是基于tcp的websocket或基于udp的quic。

Cloudhub在服务器启动前会生成一个CloudhubHandler处理边端与云端的消息，结构体如下，主要包括KeepaliveInterval（心跳检测间隔）、WriteTimeout（写入msg的超时时间）、Nodes（存储node的在线情况）、nodeConns（存储connection）、nodeLocks（存储读写connection的互斥锁）、MessageQueue（消息队列）、KeepaliveChannel（心跳通道）、NodeLimit（节点限制，超过节点限制Cloudhub报错）、Handlers（通信的主要处理方法）、MessageAcks（消息确认）。

```go
type MessageHandle struct {
	KeepaliveInterval int
	WriteTimeout      int
	Nodes             sync.Map
	nodeConns         sync.Map
	nodeLocks         sync.Map
	MessageQueue      *channelq.ChannelMessageQueue
	Handlers          []HandleFunc
	NodeLimit         int
	KeepaliveChannel  map[string]chan struct{}
	MessageAcks       sync.Map
}
```

当edgecore第一次连接cloudcore时，服务器会触发messageHandle的OnRegister方法注册节点，否则将复用之前的连接。

OnRegister方法调用ServeConn方法，ServeConn调用RegisterNode方法，完成节点的注册。节点的注册具体包含以下几步：1.分配消息队列和缓存给注册的节点。2.将连接的消息根据路由发送给DeviceController或EdgeController。3.将Node ID存入Nodes、nodeConns、nodeLocks这三个表。

注册成功后循环启动messageHandler的handler协程，分别是KeepaliveCheckLoop检查edge-node是否alive、MessageWriteLoop处理所有写入请求、ListMessageWriteLoop处理所有list类型的请求。

```go
CloudhubHandler.Handlers = []HandleFunc{
   CloudhubHandler.KeepaliveCheckLoop,
   CloudhubHandler.MessageWriteLoop,
   CloudhubHandler.ListMessageWriteLoop,
}
```

如果handler过程中检测到异常，会发送信息到exitServe通道，如果ServeConn检测到ExitCode，会进行相应的UnRegister操作。

如果没有检测到异常，就会执行handleMessage方法进行消息处理，首先检查消息类型是否合法，然后如果是list资源类型的message的请求，则不做消息可靠性处理，根据node ID取出信息组装成msg写入连接，并删除缓存。

## 消息可靠性

如果是message类型，则表示EdgeController和devicecontroller需要同步到边端的消息。这时需要Cloudhub和Edgehub作消息通信。但考虑到云与边缘之间的不稳定网络会导致边缘节点频繁断开连接。如果cloudcore或edgecore重启或离线一段时间，可能会导致发送到边缘节点的消息丢失，而这些消息在没有消息可靠性前是无法到达的。如果没有将新事件成功发送到边缘，这将导致云和边缘之间的不一致。所以云边通信需要一个可靠的消息传输机制。它需要覆盖以下几种情况：1.如果cloudcore重新启动或离线一段时间，则每当cloudcore重新上线时，将最新事件发送到边缘节点（如果有任何更新要发送）。2.如果edgenode重新启动或离线一段时间，无论何时该节点上线时，cloudcore都会发送最新事件以使其保持最新状态。

### 消息可靠机制

有三种类型的消息传递机制：

1.最多一次 At-Most-Once
2.恰好一次 Exactly-Once
3.至少一次 At-Least-Once

第一种方法“At-Most-Once”是不可靠的，第二种方法“Exactly-Once”代价非常高，虽然它提供了可靠的消息传递，没有消息丢失或重复的现象，但是性能比第三种差很多。 由于KubeEdge遵循Kubernetes最终的一致性设计原则，因此，只要消息是最新消息，边缘节点重复接收相同消息不是问题。因此，KubeEdge采用At-Least-Once建议。

### At-Least-Once

下面显示的是消息从云传递到边缘的具体流程。

![reliablemessage-workflow](./images/reliablemessage-workflow.PNG)

1.KubeEdge使用K8s CRD存储已成功发送到Edge的资源的最新resourceVersion。当cloudcore重新启动或正常启动时，它将检查resourceVersion以避免发送旧消息。

2.EdgeController和devicecontroller将消息发送到Cloudhub，MessageDispatcher将根据消息中的节点名称将消息发送到相应的NodeMessageQueue。

3.CloudHub将顺序地将数据从NodeMessageQueue发送到相应的边缘节点[5]，并将消息ID存储在ACK通道中[4]。当收到来自边缘节点的ACK消息时，ACK通道将触发以将消息resourceVersion作为CRD保存到K8s[11]，并发送下一条消息。

4.当Edgecore收到消息时，它将首先将消息保存到本地数据存储中，然后将ACK消息返回到云中。如果cloudhub在此间隔内未收到ACK消息，它将继续重新发送该消息5次。如果所有5次重试均失败，cloudhub将丢弃该事件。

5.SyncController将处理这些失败的事件。即使边缘节点接收到该消息，返回的ACK消息也可能在传输过程中丢失。在这种情况下，cloudhub将再次发送消息，并且边缘可以处理重复的消息。

### SyncController(cloudcore组件之一)

![sync-controller](./images/sync-controller.PNG)

SyncController将定期比较保存的对象resourceVersion与K8s中的对象，然后触发诸如重试和删除之类的事件。当cloudhub将事件添加到nodeMessageQueue时，它将与nodeMessageQueue中的相应对象进行比较。 如果nodeMessageQueue中的对象较新，它将直接丢弃这些事件。

### MessageQueue

当每个边缘节点成功连接到云时，将创建一个消息队列，该消息队列将缓存所有发送到边缘节点的消息。KubeEdge使用来自kubernetes/client-go的workQueue和cacheStore来实现消息队列和对象存储。 使用Kubernetes workQueue，将合并重复事件以提高传输效率。

```go
type ChannelMessageQueue struct {
   queuePool sync.Map
   storePool sync.Map

   listQueuePool sync.Map
   listStorePool sync.Map

   ObjectSyncController *hubconfig.ObjectSyncController
}
```

- Add message to the queue:

  ```go
  key,_:=getMsgKey(&message)
  nodeStore.Add(message)
  nodeQueue.Add(key)
  ```

- Get the message from the queue:

  ```go
  key,_:=nodeQueue.Get()
  msg,_,_:=nodeStore.GetByKey(key.(string))
  ```

- Structure of the message key:

  ```go
  Key = resourceType/resourceNamespace/resourceName
  ```

### ACK message Format

Ack消息将遵循以下格式：

```go
AckMessage.ParentID = receivedMessage.ID
AckMessage.Operation = "response"
```

由此实现了以下不同情况的相应处理：

1.CloudCore重新启动
当cloudcore重新启动或正常启动时，它将检查resourceVersion以避免发送旧消息。在cloudcore重新启动期间，如果删除了某些对象，则此时的删除事件可能会丢失。 SyncController将处理这种情况。这里需要使用对象GC机制来确保删除：比较存储在CRD中的对象是否存在于K8s中。如果不是，则SyncController将生成并发送删除事件到边缘，并在收到ACK时删除CRD中的对象。

2.EdgeCore重新启动
当edgecore重新启动或离线一段时间后，节点消息队列将缓存所有消息，每当节点重新上线时，将发送消息。当边缘节点离线时，cloudhub将停止发送消息，直到边缘节点重新上线后才重试。

3.EdgeNode已删除
从云中删除边缘节点后，cloudcore将删除相应的消息队列和存储。

