# 关于Pomelo的一些说明


#### 1. 关于日志收集和监控

目前, 如果在[分布式部署的环境](https://github.com/NetEase/pomelo/wiki/Pomelo%E7%9A%84%E5%88%86%E5%B8%83%E5%BC%8F%E9%83%A8%E7%BD%B2%E6%96%B9%E6%B3%95)下, `game-server服务器集群`中的各个服务器进程的日志是分别输出在其所在的机器硬盘上的. 可以考虑对日志的收集和集中监控, 具体可参考[Scribe](https://github.com/facebook/scribe)(Facebook开源的日志收集系统). 并且, 对于各种具体业务逻辑的日志也可以规定其特有的格式, 然后使用BI(Business intelligence)系统对日志进行智能分析并给出符合特定需求的报表.

#### 2. 关于负载均衡

##### 集群负载均衡的几种调度算法

1. (Round-Robin Scheduling : RR)
rr - 纯轮询方式. 把每项请求按顺序在真正服务器中分派。

2. (Least-Connection : LC)
lc - 根据最小连接数分派.

3. 目标地址散列调度（Destination Hashing Scheduling : DH）
dh - 针对目标IP地址的负载均衡, 它是一种静态映射算法, 通过一个散列(Hash)函数将一个目标IP地址映射到一台服务器.

... 等等 ...

我们在[Treasures](https://github.com/NetEase/pomelo/wiki/Treasures-%E4%BB%8B%E7%BB%8D)中的负载均衡算法为: 使用crc算法将玩家输入的用户名字符串转换为整数, 然后对`connector服务器`个数取余hash到某个`connector服务器`上, 将要连接的`connector服务器 host和port`返回给客户端.

##### Sticky Sessions

这样我们在Treasures中就实现了所谓的`Sticky Sessions`. 因为客户端通过`gate服务器`拿到了`connector服务器 host和port`, 接着就与这个`connector服务器`建立长连接, 而`connector服务器`会保存与客户端的`session`并根据实际情况对该`session`中的内容进行修改.

在我们的另一个使用Pomelo框架开发的`网易消息推送平台`服务器应用中, LVS依据`Round-Robin Scheduling(纯轮询方式)`来对客户端的连接进行负载均衡, 将客户端的连接分配到某个业务服务器上, 该服务器会将自身的一个`标识`作为cookie的一部分ack给客户端. 这样, 该客户端之后的所有请求都会带有这个`标识`, 这个`标识`会被LVS所识别并路由到对应的服务器上. 如此便实现了`Sticky Sessions`.

##### 为什么要实现Sticky Sessions

当我们使用反向代理做负载均衡时, 用户的多次请求可能被转发到不同的后端服务器. 若集群中有两台功能完全相同的服务器, 用户发出的第一个请求被分配至服务器A, A中保存了一些用户信息在`session`中, 该用户再次发送的请求却被分配到了服务器B. 如果B要用之前A保存的用户信息, 就不容易做到了.

基于以上简单论述, 在某些使用Pomelo框架开发的应用服务器中应该考虑实现`Sticky Sessions`.


#### 3. 关于服务器具体业务逻辑状态监控

* 在我们使用Pomelo框架开发的具体应用服务器的运行过程中, 我们常常关心某些数据的值及其变化趋势, 例如: 在[LordOfPomelo](https://github.com/NetEase/pomelo/wiki/LordOfPomelo-%E4%BB%8B%E7%BB%8D)中, 我们非常关心某个时间段内某个`area服务器`上进行战斗的次数. 这时, 我们可以让各个`area服务器`每格一段时间就将当前时刻进行战斗的次数和时间戳上报给`master服务器`, 然后在`master服务器`将这些数据汇总并记日志, 以供后面对日志进行相关的BI分析.

#### 4. 关于Pomelo服务器框架所支持的connector类型

* Pomelo服务器框架可以支持[Socket.IO](http://socket.io/)和[WebSocket](http://www.websocket.org/)两种连接的客户端, 只要客户端实现了相关接口就可以和服务器进行通讯了. 具体可以参考[pomelo-robot-demo](https://github.com/NetEase/pomelo-robot-demo)中`app/script/lord.js`的实现, 注: 这里实现的WebSocket的客户端, 大家可以对照相关代码来实现Socket.IO的客户端并对LordOfPomelo进行自动化测试.
