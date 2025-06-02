# k8s网络

访问一个pod的ip例如10.232.46.109，那么会先经过代理机器，在代理机器上查看路由表

```bash
$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
10.232.46.64    192.168.99.42   255.255.255.192 UG    0      0        0 bond0
```

发现要转发给网关192.168.99.42，所以包转发到节点机器192.168.99.42

查看192.168.99.42的路由表

```bash
$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
10.232.46.109   0.0.0.0         255.255.255.255 UH    0      0        0 calia7b9a112dad
```

发现可以到本地的10.232.46.109网卡。

## 同节点的pod通信

在pause容器启动之前，会创建一个虚拟以太网接口对（veth pair），该接口对一端连着容器内部的eth0 ，一端连着容器外部的vethxxx，vethxxx会绑定到容器运行时配置使用的网桥bridge0上，从该网络的IP段中分配IP给容器的eth0。

当同节点上的Pod-A发包给Pod-B时，包传送路线如下：

```
pod-a的eth0—>pod-a的vethxxx—>bridge0—>pod-b的vethxxx—>pod-b的eth0
```

因为相同节点的bridge0是相通的，因此可以通过bridge0来完成不同pod直接的通信，但是不同节点的bridge0是不通的，因此不同节点的pod之间的通信需要将不同节点的bridge0给连接起来。

## 不同节点的pod通信

连接不同节点的bridge0的方式有好几种，主要有overlay和underlay，或常规的三层路由。

不同节点的bridge0需要不同的IP段，保证Pod IP分配不会冲突，节点的物理网卡eth0也要和该节点的网桥bridge0连接。因此，节点a上的pod-a发包给节点b上的pod-b，路线如下：

```
节点a上的pod-a的eth0—>pod-a的vethxxx—>节点a的bridge0—>节点a的eth0—>

节点b的eth0—>节点b的bridge0—>pod-b的vethxxx—>pod-b的eth0
```

其中一个关键点是，需要有Pod IP的路由表，这一点可以通过calico解决。

## 源码分析-kubelet

- 初始化各种所需参数，创建并初始化kubelet结构体，运行kubelet.run方法，
- `kubelet`通过各种类型的`manager`异步工作各自执行各自的任务，例如podManager负责pod生命周期的管理。
- 其中使用到了多种的`channel`来控制状态信号变化的传递，例如比较重要的channel有`podUpdates <-chan UpdatePodOptions`，来传递pod的变化情况。
- 其中pod的管理使用了syncPod的函数来处理pod相关的操作，例如创建sandbox，启动init容器，拉取镜像，启动容器，执行post start hook。

## Service

Service定义了一种抽象：一组Pod的访问逻辑抽象，这组Pod可以理解为一组微服务，而访问该组微服务只需要访问Service这个入口，而无需关心这组微服务对应了多少个实例和实例的变化（例如pod被重建后IP变化的问题）。

Service与Pod之间的关联通过`Label Selector`的方式实现。Service解耦了fronend与backend之间的直接联系，而是由Service来作为fronend和backend之间通信的桥梁。

对 Kubernetes 集群中的应用，Kubernetes 提供了简单的 `Endpoints` API，只要 `Service` 中的一组 `Pod` 发生变更，应用程序就会被更新。 对非 Kubernetes 集群中的应用，Kubernetes 提供了基于 VIP 的网桥的方式访问 `Service`，再由 `Service` 重定向到 backend `Pod`。

参考：https://k8smeetup.github.io/docs/concepts/services-networking/service/

## kube-proxy的工作原理

- kube-proxy本质就是实现service对应的一组pod的流量代理功能，管理service和endpoint的对应关系，根据endpoint的变化在每个节点上生成对应的iptables的规则。

- 当访问service cluster ip和端口时，iptables的规则会通过NAT的方式将流量转到service对应的pod上，默认是轮询的方式访问。

- 缺点是当endpoint特别多的时候，iptables rules将会非常庞大，则存在性能问题。

- 所以目前大部分是通过ingress controller来实现流量代理而不是kube-proxy的方式。

## ingress的工作原理

ingress是k8s的资源对象，描述了一个或多个的路由规则。而ingress controller实时实现对应的路由规则，ingress controller常用的有Nginx ingress controller。

基本逻辑是当新增一个pod的时候，有ingress自动维护对应的路由规则，ingress controller实时更新Nginx的配置，并reload Nginx使其生效。

客户端首先对 kubia.example.com 执行 DNS 查 找， DNS 服务器（或本地操作系统）返回了Ingress 控制器的 IP。客户端然后 向 Ingress 控制器发送 HTTP 请求，并在 Host 头中指定 kubia . example.com。控制器从该头部确定客户端尝试访 问哪个服务，通过与该服务关联 的 Endpoint 对象查看 pod IP ，并将客户端的请求转发给其中一个pod 。
如你所见， Ingress 控制器不会将请求转发给该服务，只用它来选择一个 pod 。大多数（即使不是全部）控制器都是这样工作的 。

# 各组件的原理

## apiserver

apiserver提供了集群各类资源管理的API接口，apiserver是无状态的，将全部的数据状态保存在etcd中，只有apiserver与etcd交互，其他组件 controller-manager、scheduler、kubelet通过apiserver来进行交互。

## Controller-manager

controller-manager是各种控制器的集合，它的功能是使集群资源的状态始终保持在用户期望的状态。

## scheduler

### 预选

全局选可：先根据预选的策略从全局中选出符合要求的候选节点。

预选策略有

- PodFitsResources：判断节点资源是否满足
- PodSelectorMatches：是否有指定label或taint，即nodeselector
- PodFitsHost：是否有指定节点调度

### 选优

从符合条件的候选节点中选择最优的节点。

优选策略有

- LeastRequestedPriority：选择资源消耗最小的节点
- BalancedResourceAllocation：选择资源使用率最均衡的节点
- NodeAffinity：节点亲缘性或反亲缘性调度

具体操作如下：

- PrioritizeNodes通过并行运行各个优先级函数来对节点进行优先级排序。
- 每个优先级函数会给节点打分，打分范围为0-10分。
- 0 表示优先级最低的节点，10表示优先级最高的节点。
- 每个优先级函数也有各自的权重。
- 优先级函数返回的节点分数乘以权重以获得加权分数。
- 最后组合（添加）所有分数以获得所有节点的总加权分数。

### 抢占

当pod不适合任何节点的时候，可能pod会调度失败。这时候可能会根据pod的优先级发生抢占。

## kubelet

kubelet主要用来管理分配到该节点上的pod的生命周期，定期向apiserver汇报节点的健康状况。

# list-watch机制

Etcd 存储集群的数据信息，apiserver 作为统一入口，任何对数据的操作都必须经过 apiserver。客户端(kubelet/scheduler/ontroller-manager)通过 list-watch 监听 apiserver 中资源(pod/rs/rc 等等)的 create, update 和 delete 事件，并针对事件类型调用相应的事件处理函数。

那么 list-watch 具体是什么呢，顾名思义，list-watch 有两部分组成，分别是 list 和 watch。list 非常好理解，就是调用资源的 list API 罗列资源，基于 HTTP 短链接实现；watch 则是调用资源的 watch API 监听资源变更事件，基于 HTTP 长链接实现，也是本文重点分析的对象。

首先消息必须是可靠的，list 和 watch 一起保证了消息的可靠性，避免因消息丢失而造成状态不一致场景。具体而言，list API 可以查询当前的资源及其对应的状态(即期望的状态)，客户端通过拿期望的状态和实际的状态进行对比，纠正状态不一致的资源。Watch API 和 apiserver 保持一个长链接，接收资源的状态变更事件并做相应处理。如果仅调用 watch API，若某个时间点连接中断，就有可能导致消息丢失，所以需要通过 list API 解决消息丢失的问题。从另一个角度出发，我们可以认为 list API 获取全量数据，watch API 获取增量数据。虽然仅仅通过轮询 list API，也能达到同步资源状态的效果，但是存在开销大，实时性不足的问题。

## informer的组件

- Controller：用来把Reflector、DeltaFIFO组合起来形成一个相对固定的、标准的处理流程。
- Reflector：通过 Kubernetes Watch API 监听某种 resource 下的所有事件
- DeltaFIFO：记录对象变化的先进先出的队列
- LocalStore：二级缓存，LocalStore 只会被 Lister 的 List/Get 方法访问 。与DeltaFIFO存在resync。
- Lister：调用 List/Get 方法
- Processor：记录了所有的回调函数实例(即 ResourceEventHandler 实例)，并负责触发这些函数。

**流程：**

1. Informer 在初始化时，Reflector 会先 List API 获得所有的 Pod
2. Reflect 拿到全部Pod后，会将全部 Pod 放到 Store 中
3. 如果有人调用 Lister 的 List/Get 方法获取 Pod， 那么 Lister 会直接从 Store 中拿数据
4. Informer 初始化完成之后，Reflector 开始 Watch Pod，监听 Pod 相关 的所有事件;如果此时 pod_1 被删除，那么 Reflector 会监听到这个事件
5. Reflector 将 pod_1 被删除 的这个事件发送到 DeltaFIFO
6. DeltaFIFO 首先会将这个事件存储在自己的数据结构中(实际上是一个 queue)，然后会直接操作 Store 中的数据，删除 Store 中的 pod_1
7. DeltaFIFO 再 Pop 这个事件到 Controller 中
8. Controller 收到这个事件，会触发 Processor 的回调函数，回调函数主要有OnAdd、OnUpdate和 OnDelete三个方法。

流程2：

1. reflector通过list-watch的机制将对象添加到DeltaFIFO的队列中，ListAndWatch第一次会列出所有的对象，并获取资源对象的版本号，然后watch资源对象的版本号来查看是否有被变更。`list()`可能会导致本地的缓存相对于etcd里面的内容存在延迟，`Reflector`会通过`watch`的方法将延迟的部分补充上，使得本地的缓存数据与etcd的数据保持一致。
2. `Reflector`的主要作用是watch指定的k8s资源，并将变化同步到本地是`store`中。`Reflector`以`resyncPeriod`为周期定期执行list的操作，这样就可以使用`Reflector`来定期处理所有的对象，也可以逐步处理变化的对象。
3. controller.Run函数还会调用processLoop函数，processLoop通过调用HandleDeltas，再调用distribute，processorListener.add最终将不同更新类型的对象加入`processorListener`的channel中，供processorListener.Run使用。
4. processor的主要功能就是记录了所有的回调函数实例(即 ResourceEventHandler 实例)，并负责触发这些函数。processor记录了不同类型的事件函数，其中事件函数在NewXxxController构造函数部分注册，具体事件函数的处理，一般是将需要处理的对象加入对应的controller的任务队列中，然后由类似syncDeployment的同步函数来维持期望状态的同步逻辑。

# Pod的流程

1. 通过kubectl部署一个应用，
2. apiserver接收到请求，并将数据存放到etcd中，
3. controller发现有数据变化，调用对应的controller去创建期望的pod，
4. scheduler通过计算选出适合该pod的最优的节点
5. 节点上的kubelet发现需要创建pod，就通过container的接口去部署pod
6. Kube-proxy管理pod的网络转发，包括服务发现和负载均衡。

详细流程：

1. 流程1(RC创建)用户通过WebUI或者kubectl工具调用`kube-apiserver`提供的标准 `REST API`接口创建一个`ReplicationController`。假定在ReplicationController数据结构中定义为2个Pod。用户相当执行了下面的 `curl`命令。
   
   ```
   curl -XPOST -d "v1.ReplicationController" -H "Content-Type: application/json" http://ip:port/api/v1/namespaces/{namespace}/replicationcontrollers
   ```
   
   **而此处的v1.ReplicationController即用户的声明式数据，其中会指定应用的实例数，使用镜像等信息**

2. 流程2~3(Pod创建)`kube-controller-manager`会通过list-watch机制获取到新建中的 `v1.ReplicationController`数据，并驱使`ReplicationController控制器`工作。 `kube-controller-manager`中各个控制器模块的工作就是让集群现状趋于用户期待。假定用户声明实例是2，而当前集群中对应的Pod数量为0，则控制器就会创建2个Pod.
   
   *当然后续controller-manager也会通过list-watch机制获取到新创建的两个Pod，因为和用户期待值一致，ReplicationController的控制逻辑就收敛在用户期待状态了。*

3. 流程4~5(Pod调度)`kube-scheduler`也会通过list-watch机制获取到新创建的 `v1.Pod`，然后根据调度算法为Pod选择最合适的节点。假定调度结果是两个Pod选择的是minion1和minion2.即刷新`v1.Pod`数据的 `spec.nodeName`字段为minion1和minion2的nodeName。

4. 流程6~7(Pod运行)最后minion上的`kubelet`也会通过list-watch机制获取到调度完的 `v1.Pod`数据，然后通过`container.Runtime`接口创建Pod中指定的容器。并刷新ETCD中Pod的运行状态。

5. 运行流程看Pod状态数据变化根据上述的介绍，在运行过程中Pod状态变化

# PV及PVC的流程

**未使用sc的步骤：**

1. 管理员创建某类型的网络存储，如nfs
2. 管理员创建pv使用该网络存储
3. 用户创建一个pvc绑定该pv
4. 用户创建pod引用该pvc

pv不属于任何namespace，是属于集群级别的资源。

**使用storageclass的步骤流程：**

1. 管理员部署持久卷置配程序，即provisioner
2. 管理员创建一个使用了该provisioner的storageclass
3. 用户创建一个引用了该storageclass的pvc
4. k8s查找其引用的storageclass，并根据pvc中指定的访问模式和存储大小在storageclass配置新的pv
5. Provisioner创建对应的pv，pv和pvc形成绑定关系
6. 创建一个pod，引用该pvc。

# pause容器的作用

1.是提供Pod在Linux中共享命名空间的基础，

2.是提供Pid Namespace并使用init进程。

