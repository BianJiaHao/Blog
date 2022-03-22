# 对ZooKeeper的一点理解

### 简单介绍

官方网站：https://zookeeper.apache.org/

Zookeeper作为分布式应用之间的开源的协调服务，他提供了一些简单的原语比如

- *create*：在树中的某个位置创建一个节点
- *delete* : 删除一个节点
- *exists*：测试节点是否存在于某个位置
- *get data*：从节点读取数据
- *set data* : 将数据写入节点
- *get children* : 检索节点的子节点列表
- *sync* : 等待数据传播

而通过这些简单的原语，我们可以在客户端编写代码来实现复制的功能，比如后续会提到的配置中心以及分布式锁的实现。

**Zookeeper的数据结构**

> Zookeeper的数据结构是类似于文件树的样子，官方称之为节点 Node，但是和常规的文件树不同的是，Zookeeper的Node节点每个节点都可以存储数据，不像普通的文件夹不能存储数据，只有文件才可以存储数据。
> **重点：虽然ookeeper的节点可以存储数据，但是每个节点的存储数据大小是有限的，大约1M，所有是不推荐使用Zookeeper来作为数据库使用**

![image20220321213508543.png](/upload/2022/03/image-20220321213508543-56812679012a40e599bfab971e304251.png)

**Zookeeper节点的分类**

> 持久节点：不会随着客户端连接的断开而消失的节点，会持久化到磁盘中。
>
> 带序列的持久节点：在生成节点的时候会生成有序的序列，例如：test000001 test000002 。
>
> 临时节点：借助与客户端的Session，会随着客户端连接的断开而消失(**是实现分布式锁的关键，后续会详细进行解释**)。
>
> 带序列的临时节点：和带序列的持久节点一样只不过会随着客户端的断开而消失。

**Zookeeper的保证**

> 顺序一致性:在Zookeeper集群中，有多个slave节点，但是只有master节点负责数据的写入，客户端可以连接任意的slave节点，当客户端发出写的请求时，slave节点会将写请求提交给master节点，统一由master节点来处理，由于master是单节点，这点和Redis相似，单进程在处理写请求的时候可以保证顺序的一致性。
>
> 原子性：原子性的意思就是要么成功，要么失败没有中间的状态，在Zookeeper中就是所有设计到修改的操作要么成功要么失败，比如一个写请求转发到了master，master要保证所有的slave节点都能更新新的数据，但是Zookeeper并没有采用强一致性的策略，因为强一致性就会破坏可用性，Zookeeper采用ZAB协议来实现最终一致性，在后续核心灵魂中会进行详细的阐述。
>
> 单一的系统映像：无论你访问哪一个Zookeeper节点都能访问到相同的数据(**注意：并不是实时的一致性，你可能访问到旧的数据，和第四点及时性有关，但是可以使用sync命令来进行同步**)
>
> 可靠性：Zookeeper集群在master节点宕机后能够快速选举出新的master并继续提供服务，如官方给出的图在图中3号点的时刻，master宕机，但是Zookeeper集群花了大概200ms的时间就选举出了新的master节点并且恢复了高吞吐量，关于为何Zookeeper这么快就可以恢复在后续的Zookeeper的可靠性中会进行详细的说明。
>
> <img src="C:\Users\admin\Desktop\Blog\Blog\分布式\ZooKeeper\images\Snipaste_2022-03-22_16-13-46.png" style="zoom:75%;" />
>
> 
>
> 及时性：与上文中的第三点(单一系统映像有关)，值得是在一定的时间范围内你访问任何一个节点得到的数据都是一致的，这里说的是一定的时间范围内而不是实时，因为Zookeeper是最终一致性。当然你可以在访问数据的时候加上sync就可以保证访问到最新的数据。

**Zookeeper的集群模型**

<img src="C:\Users\admin\Desktop\Blog\Blog\分布式\ZooKeeper\images\Snipaste_2022-03-22_16-28-33.png" style="zoom:75%;" />

> 上图即为Zookeeper的集群模型，有两种状态，一种为可用状态另一种为不可用状态，为什么会出现这两种状态，原因是Master节点是单点的，那么必然有单点故障的问题，master一定会挂，那么整个集群就会变成不可用状态，那岂不是说明Zookeeper不是高可用的，其实不然，Zookeeper及其高可用，原因是他能够快速将不可用状态变为可用状态(**大约200ms的样子**),但是前提是集群存活的机器数量要大于集群数量的一半，如果宕机的节点数量过多则集群也将变得不可用。
>
> **需要注意的是在不可用状态下Zookeeper集群是不对外提供服务的**

**Zookeeper的性能**

> Zookeeper适合于读多写少的环境，在官方给出的性能图中可以看出，即使在只有3台slave节点的集群中，在900台客户端的并发访问下依然能够保证很高的并发量，在读占100%的时候甚至达到了8w的QPS，可见Zookeeper的性能是及其高的。

<img src="C:\Users\admin\Desktop\Blog\Blog\分布式\ZooKeeper\images\Snipaste_2022-03-22_16-37-12.png" style="zoom:75%;" />

**Zookeeper的可靠性**

> 那么Zookeeper是如何实现高可用的呢，换言之就是如何快速从不可用状态恢复回可用状态的呢，首先我们要了解Zookeeper集群的通讯模式，如下图，3888端口是节点直接进行通讯的端口用于选主投票，而2888端口则是master节点开放与slave节点通讯的端口，我们假设现在有4个节点，分别是Node-1，Node-2，Node-3，Node-4，而每个节点都有自己的MyId(**是选举Master的重要凭证**),和Zid(**可以先理解为自己节点数据的版本号，后续会详细解释**)，Master选举会发生在两个阶段：分别是集群刚启动的时候以及Master节点宕机的时候，下图演示的是集群刚启动的状态，Master节点宕机的状态在下文展示。
>
> 我们假设启动节点的顺序为Node-1，Node-2，Node-4，Node-3
>
> Node-1启动：此时集群节点的数量为1并没有超过2(**过半**),集群也是不可用的状态，开启3888端口。
>
> Node-2启动：连接Node-1的3888端口，开启自己的3888端口
>
> Node-4启动：连接Node-1和Node-2的3888端口，开启自己的3888端口，此时集群节点的数量已经过半，开始投票，Node-4将自己的信息发送给Node-1，Node-2，因为集群刚启动所以每个人的Zid都是0，Node-4的Myid为4，Node-1接受到Node-4的投票信息发现他的Zid和自己一样但是Myid比自己大，果断投了一票给Node-4，同样Node-2也是如此，所以Node-4很快收到了包含自己在内的3票已经过半则将直接变为Master节点，并且开启自己的2888端口，Node-1和Node-2连接Node-4的2888端口。
>
> Node-3启动：连接Node-1，Node-2，Node-4的3888端口以及Node-4的2888端口
>
> 每两个节点直接都能够通过3888端口来进行交流并且不重复
>
> <img src="C:\Users\admin\Desktop\Blog\Blog\分布式\ZooKeeper\images\Snipaste_2022-03-22_17-42-13.png" style="zoom:75%;" />
>
> 如果集群运行了一段时间Master节点突然宕机了，则会发生Master选举的第二种情况，Node-3在向Node-4发送请求的时候发现Node-4已经宕机了，则会立即通过3888向Node-1，Node-2发起投票将直接的信息发送给Node-1和Node-2，Node-1收到Node-3的投票信息发现Node-3的Zid比自己小立刻驳回他的投票并且触发对自己的投票，将直接的信息发送给Node-2和Node-3,Node-2也是一样，这样Node-3就会收到Node-1和Node-2的投票信息，分别给他们两个投一票，Node-1又会收到Node-2的信息，发现Node-2的MyId比自己大，给Node-2投一票，此时Node-2已经收到了过半的投票数则将直接变成Master并且开放自己的2888端口让Node-1和Node-3连接。
>
> ![](C:\Users\admin\Desktop\Blog\Blog\分布式\ZooKeeper\images\Snipaste_2022-03-22_17-43-50.png)
>
> 集群恢复为下面的状态：
>
> <img src="C:\Users\admin\Desktop\Blog\Blog\分布式\ZooKeeper\images\Snipaste_2022-03-22_17-49-59.png" style="zoom:75%;" />
>
> 所有这里可以得出Zookeeper高可用的原因，在Master选举的过程中，并不是争取制度，而是谦让制度，谁的Zid和Myid高谁就来当Master





### 安装 

### 核心灵魂

### 代码实现 - 配置中心

### 代码实现 - 分布式锁

### 代码地址 

https://github.com/BianJiaHao/Zookeeper.git
