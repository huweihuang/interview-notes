# TCP

## 三次握手

第一次握手：A发一个syn包，带seq num给B

第二次握手：B返回一个ack包 seq num+1 给A，同时给A发了个自己的syn包，带seq num

第三次握手：A返回一个ack包 seq num+1给B。

为什么需要三次握手，假如没有第三次握手，此时A断连了，而B不知道就开始向A传输数据，则会造成数据丢失，且B不知道而频繁发送。

## 四次挥手

第一次挥手：A发一个FIN包，带seq num给B（A主动发起关闭）

第二次挥手：B返回一个ack seq num+1 给A（B同意A的关闭）

第三次挥手：B发一个FIN包，带seq num给A （B发起关闭）

第四次挥手：A返回一个ack,seq num+1给B （A同意B的关闭）

为什么要有四次握手，主要是因为TCP是半关闭的，即关闭的单向的，当A向B的通道关闭后，B仍然可以向A发送消息，当B也关闭了A的通道，双方之间才算正式关闭，不能互相发送消息。

# 连接

## 长连接

client向server发起连接，server接受client连接，双方建立连接。Client与server完成一次读写之后，它们之间的连接并不会主动关闭，后续的读写操作会继续使用这个连接。

## 短连接

client向server发起连接请求，server接到请求，然后双方建立连接。client向server发送消息，server回应client，然后一次读写就完成了，这时候双方任何一个都可以发起close操作，不过一般都是client先发起close操作。

# http/https

Https的原理就是：在传输层和应用层之间加了一层SSL/TLS。Https指的是Http协议+ SSL/TLS协议,也就是说，Http协议本来定义了数据的包装方式（要包含请求头，请求方式等），现在在这个基础上，还要求行对数据进行加密，对通信双方身份进行验证（如何加密，如何验证由SSL决定）。

# keepalived的原理

首先，每个节点有一个初始优先级，由配置文件中的priority配置项指定，MASTER节点的priority应比BAKCUP高。运行过程中keepalived根据vrrp_script的weight设定，增加或减小节点优先级。规则如下：

1. “weight”值为正时,脚本检测成功时”weight”值会加到”priority”上,检测失败是不加
   - 主失败: 主priority<备priority+weight之和时会切换
   - 主成功: 主priority+weight之和>备priority+weight之和时,主依然为主,即不发生切换
2. “weight”为负数时,脚本检测成功时”weight”不影响”priority”,检测失败时,Master节点的权值将是“priority“值与“weight”值之差
   - 主失败: 主priotity-abs(weight) < 备priority时会发生切换
   - 主成功: 主priority > 备priority 不切换

# 进程、线程、协程

## 进程

进程是资源分配的单位，即资源隔离。同进程内的所有线程可以共同访问该进程的资源，每个进程都有自己的独立内存空间，不同进程通过进程间通信来通信。由于进程比较重量，占据独立的内存，所以上下文进程间的切换开销（栈、寄存器、虚拟内存、文件句柄等）比较大，但相对比较稳定安全。

## 线程

线程是独立调度和运行的基本单位，线程间通信主要通过共享内存，上下文切换很快，资源开销较少，但相比进程不够稳定容易丢失数据。

## 协程

协程是一种用户态的轻量级线程，协程的调度完全由用户控制。协程没有线程的上下文切换消耗，

# 乐观锁和悲观锁

**乐观锁（Optimistic Lock）** 和 **悲观锁（Pessimistic Lock）** 是两种并发控制策略，用于解决并发访问共享资源时的数据一致性问题。

| 特性   | 乐观锁                             | 悲观锁                                        |
| ---- | ------------------------------- | ------------------------------------------ |
| 定义   | 假设大多数情况下不会发生冲突，在更新时才检测冲突        | 假设会发生冲突，访问资源前先加锁                           |
| 实现方式 | 通常基于 **版本号（Version）** 或 **时间戳** | 基于 **加锁机制**（如互斥锁、数据库锁）                     |
| 性能   | 无锁机制，适合读多写少的场景                  | 有锁机制，适合写多冲突多的场景                            |
| 代价   | 需要重试机制（例如 CAS 重试）               | 上锁/阻塞带来额外性能消耗                              |
| 举例   | 数据库中通过版本号字段控制更新                 | Synchronized、mutex、数据库 select for update 等 |

## Kubernetes 使用哪种锁？

**Kubernetes 大量使用的是“乐观锁”机制。**

为什么？

因为：

- Kubernetes 中的资源对象（如 Pod、Deployment）是以声明式方式进行修改，用户/控制器常常会并发更新同一个资源；

- 它依赖 **etcd** 存储，etcd 自身支持 **基于版本号的乐观并发控制（Optimistic Concurrency Control）**；

- 提高性能，避免频繁阻塞。



## K8s 中乐观锁的实现方式

在 Kubernetes 中，资源对象的每次更新都会携带一个字段：

```yaml
metadata:   resourceVersion: "123456"
```

工作流程：

1. 客户端通过 API Server 获取资源对象，拿到当前的 `resourceVersion`。

2. 客户端修改对象并发出更新请求，带上这个 `resourceVersion`。

3. API Server 将此 `resourceVersion` 与 etcd 中的最新版本进行比较：
   
   - 如果一致，则允许更新。
   
   - 如果不一致，说明资源已被其他人修改，返回 `409 Conflict`。

4. 客户端收到冲突错误，可进行重试逻辑（常用于 controller 中）。

这就是典型的 **乐观锁机制**。