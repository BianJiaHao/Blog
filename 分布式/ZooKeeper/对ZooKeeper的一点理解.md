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


### 安装 

### 核心灵魂

### 代码实现 - 配置中心

### 代码实现 - 分布式锁

### 代码地址 
https://github.com/BianJiaHao/Zookeeper.git

