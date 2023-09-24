# 集群

## 0. 集群思想

> 集群这个词
>
> 广义的集群：只要是多台机器构成分布式系统，就可以称之为一个集群。前面的主从结构，哨兵模式，都可以称之为广义的集群。
>
> 狭义的集群：redis为了解决存储大量数据时内存空间不足的问题，提供的集群cluster模式。

Redis的哨兵模式, 目的/作用是提⾼了redis分布式系统的可⽤性. 但是真正⽤来存储数据的还是 master 和 slave 节点. 所有的数据都需要存储在单个 master 和 slave 节点中.

**如果数据量很大, 接近/超出了 master / slave 所在机器的物理内存, 就可能出现严重问题了.**

如何获取更⼤的空间? 加机器即可! 所谓 "⼤数据" 的核⼼, 其实就是⼀台机器搞不定了, ⽤多台机器来搞.

**Redis 的集群就是在上述的思路之下, 引入多组 Master / Slave , 每⼀组 Master / Slave 存储数据全集的一部分, 从而构成一个更大的整体, 称为 Redis 集群 (Cluster).**

Redis集群是Redis提供的分布式数据库方案，集群通过分片（sharding）来进行数据共享，并提供复制和故障转移功能。

> 假定整个数据全集是 1 TB, 引⼊三组 Master / Slave 来存储. 那么每⼀组机器只需要存储整个数据全集的 1/3 即可.

![image-20230912165850148](https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230912165850148.png)

在上述图中

- Master1 和 Slave11 和 Slave12 保存的是同样的数据. 占总数据的 1/3
- Master2 和 Slave21 和 Slave22 保存的是同样的数据. 占总数据的 1/3
- Master3 和 Slave31 和 Slave32 保存的是同样的数据. 占总数据的 1/3

每个红框部分都可以称为是⼀个 **分片 (Sharding).**

如果全量数据进⼀步增加, 只要再增加更多的分⽚, 即可解决存储空间不足的问题.

### 数据分片算法（重点）

**Redis cluster 的核心思路是用多组master - cluster来存全量数据的每个部分. 那么接下来的核心问题就是, 给定⼀个键值对 (⼀个具体的 key), 那么这个键值对数据应该存储在哪个分片上? 读取的时候又应该去哪个分片读取?**

围绕这个问题, 业界有三种⽐较主流的实现⽅式.

#### 1. 哈希求余

设有 N 个分⽚, 使⽤ [0, N-1] 这样序号对每个分片进⾏编号. 针对某个给定的 key, 先通过哈希函数（如md5）计算 hash 值, 再把得到的结果 % N, 得到的结果即为分⽚编号.

> 例如, N 为 3. 给定 key 为 hello, 对 hello 计算 hash 值(⽐如使⽤ md5 算法), 得到的结果为 bc4b2a76b9719d91 , 再把这个结果 % 3, 结果为 0, 那么就把 hello 这个 key 放到 0 号分⽚上. 当然, 实际⼯作中涉及到的系统, 计算 hash 的⽅式不⼀定是 md5, 但是思想是⼀致的.

后续如果要读取某个 key , 也是针对 key 计算 hash , 再对 N 求余, 就可以找到对应的分⽚编号了. 存储与读取数据时计算key所在的分片步骤是一样的。

**优点: 简单高效, 数据分配均匀.**

**缺点: 集群扩容时，开销较大**
⼀旦需要对redis cluster - redis集群进行扩容, N 改变了, 原有的映射规则被破坏, 就需要让节点之间的数据相互传输, 重新排列, 以满足新的映射规则. <u>此时需要搬运的数据量是比较多的, 开销较大.</u> 也就是：若某个key在扩容之后，不应该待在原有的分片中，则需要进行重新分配， 也就是搬运数据。（类似于数组形式的哈希表进行扩容时，原有的数据都需要重新插入到新的哈希表，当然前提是哈希函数与数组大小有关（比如求余哈希函数））

![image-20230924201601560](https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230924201601560.png)

如上图，假定有100-120 20个hash值（key通过哈希函数的计算结果），当分片从3扩容到4时，有17个哈希值%3和%4的结果不同，需要搬运。也就是说17/20 = 85%的数据都需要搬运。**这样的扩容代价是很大的。**

#### 2. 一致性哈希算法

**为了降低上述的搬运开销, 能够更高效扩容，降低扩容时需要搬运的数据量**, 业界提出了 "⼀致性哈希算法".

key 映射到分⽚序号的过程不再是简单求余了, ⽽是改成以下过程:

> 第⼀步, 把 0 -> 2^32-1 这个数据空间, 映射到⼀个圆环上. 数据按照顺时针⽅向增⻓.
>
> 第⼆步, 假设当前存在三个分⽚, 就把分⽚放到圆环的某个位置上.（平均分布）
>
> 第三步, 假定有⼀个 key, 计算得到 hash 值 H, 那么这个 key 映射到哪个分⽚呢? 规则很简单, 就是从 H 所在位置, 顺时针往下找, 找到的第⼀个分片，即为该 key 所从属的分⽚.
>
> 这就相当于, N 个分⽚, 把整个圆环分成了 N 个管辖区间. Key 的 hash 值落在某个区间内, 就归区间对应的分片管理.

<img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230924201621748.png" alt="image-20230924201621748" style="zoom:67%;" />

<img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230924201632999.png" alt="image-20230924201632999" style="zoom:67%;" />

在这个情况下, 如果扩容⼀个分⽚, 如何处理呢? 原有分⽚在环上的位置不动, 只要在环上新安排⼀个分⽚位置即可.

此时, 只需要把 0 号分⽚上的部分数据（50%，也可以不是50%）, 搬运给 3 号分⽚即可. 1 号分⽚和 2 号分⽚管理的区间都是不变的.

也就是说，分片从3到4的时候，只需要搬运全量数据的1/6 = 16.7%，相比于之前的哈希求余的85%就少很多了。

**优点: 大大降低了扩容时数据搬运的规模, 提高了扩容操作的效率.** 

**缺点: 数据分配不均匀，有的多有的少（数据倾斜）**

#### 3. 哈希槽分区算法 (Redis 使用)

为了解决上述问题 (搬运成本⾼ 和 数据分配不均匀), Redis cluster 引⼊了哈希槽 (hash slots) 算法.

`hash_slot = crc16(key) % 16384`

其中 crc16 也是⼀种 hash 算法

16384 其实是 16 * 1024, 也就是 2^14.（redis的哈希槽分区算法中总的槽位的数量固定，就是16384）

相当于是把全部的哈希值, 映射到 16384 个槽位上, 也就是 [0, 16383].

然后再把这些槽位⽐较均匀的分配给每个分⽚. 每个分⽚的节点都需要记录⾃⼰持有哪些槽位.

假设当前有三个分⽚, ⼀种可能的槽位分配⽅式:

0 号分⽚: [0, 5461], 共 5462 个槽位

1 号分⽚: [5462, 10923], 共 5462 个槽位

2 号分⽚: [10924, 16383], 共 5460 个槽位

> 这⾥的槽位分配规则是很灵活的. 每个分⽚持有的槽位也不⼀定连续. 每个分⽚的节点使⽤ 位图 来表⽰⾃⼰持有哪些槽位. 对于 16384 个槽位来说, 需要 2048 个字节（16384 / 8）(2KB) 大小的内存空间表示.

如果需要进⾏扩容, ⽐如新增⼀个 3 号分⽚, 就可以针对原有的槽位进⾏重新分配.

⽐如可以把之前每个分⽚持有的槽位, 各拿出⼀点, 分给新分⽚. 

⼀种可能的分配⽅式: 

- 0 号分⽚: [0, 4095], 共 4096 个槽位
- 1 号分⽚: [5462, 9557], 共 4096 个槽位
- 2 号分⽚: [10924, 15019], 共 4096 个槽位
- 3 号分⽚: [4096, 5461] + [9558, 10923] + [15019, 16383], 共 4096 个槽位

> 我们在实际使⽤ Redis 集群分⽚的时候, 不需要⼿动指定哪些槽位分配给哪个分⽚, 只需要告诉某个分⽚应该持有多少个槽位即可, Redis 会⾃动完成后续的槽位分配, 以及对应的 key 搬运的⼯作.

此时，你会发现，从3个分片扩容到4个分片时，只需要搬运1/4 = 25%的数据，4个分片扩容到5个分片时，只需要搬运1/5 = 25%的数据，并且，这已经是最小限度的需要搬运的数据量了，因为你新增一个分片，怎么着它也得有数据吧，且在解决数据倾斜的问题之后，这就是最小的需要搬运的数据量。

此处还有两个问题: 

> 问题⼀: Redis 集群是最多有 16384 个分⽚吗? 
>
> 并⾮如此. 如果⼀个分⽚只有⼀个槽位, 这对于集群的数据均匀分布其实是难以保证的. （这时候数据的分布情况就完全由哈希函数决定了）实际上 Redis 的作者建议集群分⽚数不应该超过 1000. 
>
> ⽽且, 16000 这么⼤规模的集群, 本⾝的可用性也是⼀个⼤问题. ⼀个系统越复杂, 出现故障的概率是越⾼的.

> 问题⼆: 为什么是 16384 个槽位?
>
> 翻译过来⼤概意思是: 
>
> - **节点之间通过心跳包通信. 心跳包中包含了该节点持有哪些 slots. 这个是使用位图这样的数据结构表示的**. 表⽰ 16384 (16k) 个 slots, 需要的位图⼤⼩是 2KB. 如果给定的 slots 数更多了, ⽐如 65536 个了, 此时就需要消耗更多的空间，即 8 KB 的位图表⽰了. 8 KB, 对于内存来说不算什么, 但是在频繁的⽹络⼼跳包中, 还是⼀个不⼩的开销的.
>   （心跳包：频繁，周期性的，且网络带宽资源实际上是比内存资源更宝贵的！）
> - 另⼀⽅⾯, Redis 集群⼀般不建议超过 1000 个分⽚. 所以 16384 对于最⼤ 1000 个分⽚来说是**⾜够⽤的,** 同时也会使对应的槽位配置位图体积不⾄于很⼤.

## 1. 节点

一个 Redis 集群通常由多个节点（node）组成， 在刚开始的时候， 每个节点都是相互独立的，要组建一个真正可工作的集群， 我们必须将各个独立的节点连接起来， 构成一个包含多个节点的集群。

**连接各个节点的工作可以使用 CLUSTER MEET 命令**来完成， 该命令的格式如下：

```
CLUSTER MEET <ip> <port>
```

向一个节点 `node` 发送 CLUSTER MEET 命令， 可以让 `node` 节点与 `ip` 和 `port` 所指定的节点进行握手（handshake）， 当握手成功时， `node` 节点就会将 `ip` 和 `port` 所指定的节点添加到 `node` 节点当前所在的集群中。

> 举个例子， 假设现在有三个独立的节点 `127.0.0.1:7000` 、 `127.0.0.1:7001` 、 `127.0.0.1:7002` （下文省略 IP 地址，直接使用端口号来区分各个节点）， 我们首先使用客户端连上节点 7000 ， 通过发送 CLUSTER NODE 命令可以看到， 集群目前只包含 7000 自己一个节点：
>
> ```
> $ redis-cli -c -p 7000
> 127.0.0.1:7000> CLUSTER NODES
> 51549e625cfda318ad27423a31e7476fe3cd2939 :0 myself,master - 0 0 0 connected
> ```
>
> 通过向节点 7000 发送以下命令， 我们可以将节点 7001 添加到节点 7000 所在的集群里面：
>
> ```
> 127.0.0.1:7000> CLUSTER MEET 127.0.0.1 7001
> OK
> 
> 127.0.0.1:7000> CLUSTER NODES
> 68eef66df23420a5862208ef5b1a7005b806f2ff 127.0.0.1:7001 master - 0 1388204746210 0 connected
> 51549e625cfda318ad27423a31e7476fe3cd2939 :0 myself,master - 0 0 0 connected
> ```
>
> 继续向节点 7000 发送以下命令， 我们可以将节点 7002 也添加到节点 7000 和节点 7001 所在的集群里面：
>
> ```
> 127.0.0.1:7000> CLUSTER MEET 127.0.0.1 7002
> OK
> 
> 127.0.0.1:7000> CLUSTER NODES
> 68eef66df23420a5862208ef5b1a7005b806f2ff 127.0.0.1:7001 master - 0 1388204848376 0 connected
> 9dfb4c4e016e627d9769e4c9bb0d4fa208e65c26 127.0.0.1:7002 master - 0 1388204847977 0 connected
> 51549e625cfda318ad27423a31e7476fe3cd2939 :0 myself,master - 0 0 0 connected
> ```
>
> 现在， 这个集群里面包含了 7000 、 7001 和 7002 三个节点，下图展示了这三个节点进行握手的整个过程。
>
> <img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230923144344468.png" alt="image-20230923144344468" style="zoom: 67%;" /><img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230923144352125.png" alt="image-20230923144352125" style="zoom:67%;" /><img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230923144410449.png" alt="image-20230923144410449" style="zoom:67%;" />
>
> <img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230923144423126.png" alt="image-20230923144423126" style="zoom:67%;" /><img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230923144432520.png" alt="image-20230923144432520" style="zoom:67%;" />

本节接下来的内容将介绍启动节点的方法， 和集群有关的数据结构， 以及 `CLUSTER MEET` 命令的实现原理。

### 启动节点-集群模式下的Redis服务器

**<u>一个节点就是一个运行在集群模式下的 Redis 服务器</u>**

**Redis 服务器在启动时会根据 `cluster-enabled` 配置选项的是否为 `yes` 来决定是否开启服务器的集群模式**

> 这个定义很关键, 因为下文中会大量使用节点这个词, 而我们需要清晰的认识到 **一个节点就是一个运行在集群模式下的Redis服务器**

<img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230923144040292.png" alt="image-20230923144040292" style="zoom:67%;" />

> 节点（运行在集群模式下的 Redis 服务器）会继续使用所有在单机模式中使用的服务器组件， 比如说：
>
> - 节点会继续使用文件事件处理器来处理命令请求和返回命令回复。
> - 节点会继续使用时间事件处理器来执行 `serverCron` 函数， <u>而 `serverCron` 函数又会调用集群模式特有的 `clusterCron` 函数： `clusterCron` 函数负责执行在集群模式下需要执行的常规操作</u>， 比如向集群中的其他节点发送 Gossip 消息， 检查节点是否断线； 又或者检查是否需要对下线节点进行自动故障转移， 等等。
> - 节点会继续使用数据库来保存键值对数据，键值对依然会是各种不同类型的对象。
> - 节点会继续使用 RDB 持久化模块和 AOF 持久化模块来执行持久化工作。
> - 节点会继续使用发布与订阅模块来执行 PUBLISH 、 SUBSCRIBE 等命令。
> - 节点会继续使用复制模块来进行节点的复制工作。
> - 节点会继续使用 Lua 脚本环境来执行客户端输入的 Lua 脚本。
>
> 诸如此类。

<u>除此之外， 节点会继续使用 `redisServer` 结构来保存服务器的状态， 使用 `redisClient` 结构来保存客户端的状态</u>

<u>至于那些只有在集群模式下才会用到的数据， 节点将它们保存到了 `cluster.h/clusterNode` 结构， `cluster.h/clusterLink` 结构， 以及 `cluster.h/clusterState` 结构里面， 接下来的一节将对这三种数据结构进行介绍。</u>

> 所以, 集群中的节点, 实际上就是一个处于集群模式下的Redis服务器, 类似于 **Sentinel 本质上只是一个运行在特殊模式下的 Redis 服务器**， 所以启动 Sentinel 的**第一步， 就是初始化一个普通的 Redis 服务器**
>
> 而使一个Redis服务器运行在集群模式下, 可以在配置文件中将cluster-enabled设置为yes, Redis服务器启动时会根据这个配置项决定是否开启集群模式
>
> 和哨兵模式不太一样的是, 集群模式下的Redis服务器会继续使用所有在单机模式中使用的服务器组件, 除此之外, 有关集群的功能, 主要使用`cluster.h/clusterNode` 结构， `cluster.h/clusterLink` 结构， 以及 `cluster.h/clusterState` 结构, 共三个集群数据结构来实现.

### 集群数据结构

#### cluster.h/clusterNode

**`clusterNode` 结构保存了一个节点的当前状态**， 比如节点的创建时间， 节点的名字， 节点当前的配置纪元， 节点的 IP 和地址， 等等。

**每个节点(集群模式的Redis服务器)都会使用一个 `clusterNode` 结构来记录自己的状态， 并为集群中的所有其他节点（包括主节点和从节点）都创建一个相应的 `clusterNode` 结构， 以此来记录其他节点的状态：**

```
struct clusterNode {

    // 创建节点的时间
    mstime_t ctime;

    // 节点的名字，由 40 个十六进制字符组成
    // 例如 68eef66df23420a5862208ef5b1a7005b806f2ff
    char name[REDIS_CLUSTER_NAMELEN];

    // 节点标识
    // 使用各种不同的标识值记录节点的角色（比如主节点或者从节点），
    // 以及节点目前所处的状态（比如在线或者下线）。
    int flags;

    // 节点当前的配置纪元，用于实现故障转移
    uint64_t configEpoch;

    // 节点的 IP 地址
    char ip[REDIS_IP_STR_LEN];

    // 节点的端口号
    int port;

    // 保存连接节点所需的有关信息
    clusterLink *link;

    // ...

};
```

#### cluster.h/clusterLink

**`clusterNode` 结构的 `link` 属性是一个 `clusterLink` 结构， 该结构保存了连接节点所需的有关信息**， 比如套接字描述符， 输入缓冲区和输出缓冲区：

```
typedef struct clusterLink {

    // 连接的创建时间
    mstime_t ctime;

    // TCP 套接字描述符
    int fd;

    // 输出缓冲区，保存着等待发送给其他节点的消息（message）。
    sds sndbuf;

    // 输入缓冲区，保存着从其他节点接收到的消息。
    sds rcvbuf;

    // 与这个连接相关联的节点，如果没有的话就为 NULL
    struct clusterNode *node;

} clusterLink;
```

`redisClient` 结构和 `clusterLink` 结构的相同和不同之处

`redisClient` 结构和 `clusterLink` 结构都有自己的套接字描述符和输入、输出缓冲区， 这两个结构的区别在于， `redisClient` 结构中的套接字和缓冲区是用于连接客户端的， 而 `clusterLink` 结构中的套接字和缓冲区则是用于连接节点的。

#### cluster.h/clusterState

**最后， <u>每个节点都保存</u>着一个 `clusterState` 结构， 这个结构记录了在当前节点的视角下， 集群目前所处的状态 —— 比如集群是在线还是下线， 集群包含多少个/哪些节点， 集群当前的配置纪元**， 诸如此类：

```
typedef struct clusterState {

    // 指向当前节点的指针
    clusterNode *myself;

    // 集群当前的配置纪元，用于实现故障转移
    uint64_t currentEpoch;

    // 集群当前的状态：是在线还是下线
    int state;

    // 集群中至少处理着一个槽的节点的数量
    int size;

    // 集群节点名单（包括 myself 节点）
    // 字典的键为节点的名字，字典的值为节点对应的 clusterNode 结构
    dict *nodes;

    // ...

} clusterState;
```

> 以前面介绍的 7000 、 7001 、 7002 三个节点为例， 下图展示了节点 7000 创建的 `clusterState` 结构， 这个结构从节点 7000 的角度记录了集群、以及集群包含的三个节点的当前状态 （为了空间考虑，图中省略了 `clusterNode` 结构的一部分属性）：
>
> - 结构的 `currentEpoch` 属性的值为 `0` ， 表示集群当前的配置纪元为 `0` 。
> - 结构的 `size` 属性的值为 `0` ， 表示集群目前没有任何节点在处理槽： 因此结构的 `state` 属性的值为 `REDIS_CLUSTER_FAIL` —— 这表示集群目前处于下线状态。
> - 结构的 `nodes` 字典记录了集群目前包含的三个节点， 这三个节点分别由三个 `clusterNode` 结构表示： 其中 `myself` 指针指向代表节点 7000 的 `clusterNode` 结构， 而字典中的另外两个指针则分别指向代表节点 7001 和代表节点 7002 的 `clusterNode` 结构， 这两个节点是节点 7000 已知的在集群中的其他节点。
> - 三个节点的 `clusterNode` 结构的 `flags` 属性都是 `REDIS_NODE_MASTER` ，说明三个节点都是主节点。
>
> <img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230923201944905.png" alt="image-20230923201944905" style="zoom:80%;" />
>
> 节点 7001 和节点 7002 也会创建类似的 `clusterState` 结构：
>
> - 不过在节点 7001 创建的 `clusterState` 结构中， `myself` 指针将指向代表节点 7001 的 `clusterNode` 结构， 而节点 7000 和节点 7002 则是集群中的其他节点。
> - 而在节点 7002 创建的 `clusterState` 结构中， `myself` 指针将指向代表节点 7002 的 `clusterNode` 结构， 而节点 7000 和节点 7001 则是集群中的其他节点。

也就是说, 集群中的每个节点 - 处于集群模式的Redis服务器, 都会用一个clusterNode结构去描述每一个集群中的节点, 若集群有10个节点, 则10个节点中就共有10 * 10个clusterNode结构体的实例化对象

每个节点内部的clusterState中的dict *nodes中的代表自己的那个键值对的值(clusterNode\*类型)指向的对象和myself指针指向同一个clusterNode, 都是记录自己状态的clusterNode

### CLUSTER MEET 命令的实现

**通过向节点 A 发送 CLUSTER MEET 命令， 客户端可以让接收命令的节点 A 将另一个节点 B 添加到节点 A 当前所在的集群里面**(实际上, 也会让节点B将节点A添加到当前自己所在的集群里面, 双向的)：

```
CLUSTER MEET <ip> <port>
```

收到命令的节点 A 将与节点 B 进行握手（handshake）， 以此来确认彼此的存在， 并为将来的进一步通信打好基础：

1. **节点 A 会为节点 B 创建一个 `clusterNode` 结构，并将该结构添加到自己的 `clusterState.nodes` 字典里面。**
2. 节点 A 根据 CLUSTER MEET 命令给定的 IP 地址和端口号， <u>向节点 B 发送一条 `MEET` 消息</u>（message）。
3. **节点 B 会为节点 A 创建一个 `clusterNode` 结构， 并将该结构添加到自己的 `clusterState.nodes` 字典里面。**
4. <u>节点 B 将向节点 A 返回一条 `PONG` 消息。</u>
5. <u>节点 A 将向节点 B 返回一条 `PING` 消息。</u>
6. <u>握手完成。</u>

> 这里的MEET, PING, PONG消息见7. 消息
>
> 命令是直接给Redis客户端 - Redis使用者提供使用的, 而消息是Redis的内部实现.

![image-20230923150144050](https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230923150144050.png)

之后，节点 A 会将节点 B 的信息通过 Gossip 协议传播给集群中的其他节点，让其他节点也与节点 B 进行握手，最终，经过一段时间之后，节点 B 会被集群中的所有节点认识。

也就是说, 若集群中当前有N个节点, 需要新增一个X节点, 则只需向N个节点中的一个节点发送 CLUSTER MEET x_ip x_port即可将X节点加入集群中

## 2. 槽指派(指派是一个动词)

**Redis集群通过分片的方式来保存数据库中的键值对：集群的整个数据库被分为16384个槽（slot），数据库中的每个键都属于这16384个槽的其中一个，集群中的每个节点可以处理0个或最多16384个槽。**

当数据库中的16384个槽都有节点在处理时，**集群处于上线状态（ok）**；相反地，如果数据库中有任何一个槽没有得到处理，那么**集群处于下线状态（fail）。**

> 在上一节，我们使用CLUSTER MEET命令将7000、7001、7002三个节点连接到了同一个集群里面，不过这个集群目前仍然处于下线状态，因为集群中的三个节点都没有在处理任何槽：
>
> ```
> 127.0.0.1:7000> CLUSTER INFO
> cluster_state:fail            // 集群状态
> cluster_slots_assigned:0
> cluster_slots_ok:0
> cluster_slots_pfail:0
> cluster_slots_fail:0
> cluster_known_nodes:3
> cluster_size:0
> cluster_current_epoch:0
> cluster_stats_messages_sent:110
> cluster_stats_messages_received:28
> ```
>
> **通过向节点发送CLUSTER ADDSLOTS命令，我们可以将一个或多个槽指派（assign）给节点负责**：
>
> ```
> CLUSTER ADDSLOTS <slot> [slot ...]
> ```
>
> 举个例子，执行以下命令可以将槽0至槽5000指派给节点7000负责：
>
> ```
> 127.0.0.1:7000> CLUSTER ADDSLOTS 0 1 2 3 4 ... 5000
> OK
> 127.0.0.1:7000> CLUSTER NODES
> 9dfb4c4e016e627d9769e4c9bb0d4fa208e65c26 127.0.0.1:7002 master - 0 1388316664849 0 connected
> 68eef66df23420a5862208ef5b1a7005b806f2ff 127.0.0.1:7001 master - 0 1388316665850 0 connected
> 51549e625cfda318ad27423a31e7476fe3cd2939 :0 myself,master - 0 0 0 connected 0-5000
> ```
>
> 为了让7000、7001、7002三个节点所在的集群进入上线状态，我们继续执行以下命令，将槽5001至槽10000指派给节点7001负责：
>
> ```
> 127.0.0.1:7001> CLUSTER ADDSLOTS 5001 5002 5003 5004 ... 10000
> OK
> ```
>
> 然后将槽10001至槽16383指派给7002负责：
>
> ```
> 127.0.0.1:7002> CLUSTER ADDSLOTS 10001 10002 10003 10004 ... 16383
> OK
> ```
>
> 当以上三个CLUSTER ADDSLOTS命令都执行完毕之后，<u>数据库中的16384个槽都已经被指派给了相应的节点，集群进入上线状态</u>：
>
> ```
> 127.0.0.1:7000> CLUSTER INFO
> cluster_state:ok				// 集群所处状态
> cluster_slots_assigned:16384
> cluster_slots_ok:16384
> cluster_slots_pfail:0
> cluster_slots_fail:0
> cluster_known_nodes:3
> cluster_size:3
> cluster_current_epoch:0
> cluster_stats_messages_sent:2699
> cluster_stats_messages_received:2617
> 127.0.0.1:7000> CLUSTER NODES
> 9dfb4c4e016e627d9769e4c9bb0d4fa208e65c26 127.0.0.1:7002 master - 0 1388317426165 0 connected 10001-16383
> 68eef66df23420a5862208ef5b1a7005b806f2ff 127.0.0.1:7001 master - 0 1388317427167 0 connected 5001-10000
> 51549e625cfda318ad27423a31e7476fe3cd2939 :0 myself,master - 0 0 0 connected 0-5000
> ```

本节接下来的内容将首先介绍节点保存槽指派信息的方法，以及节点之间传播槽指派信息的方法，之后再介绍CLUSTER ADDSLOTS命令的实现。

### 记录节点的槽指派信息

**clusterNode结构的slots属性和numslot属性记录了节点负责处理哪些槽**：

```
struct clusterNode {
  // ...
  unsigned char slots[16384/8];
  int numslots;
  // ...
};
```

slots属性是一个二进制位数组（bit array）- 位图，这个数组的长度为16384/8=2048个字节，共包含16384个二进制位。每个二进制位的0/1标识着当前clusterNode是否负责处理该槽(是否存储该槽位对应的键值对), Redis以0为起始索引，16383为终止索引，对slots数组中的16384个二进制位进行编号

> 一个slots数组示例：这个数组索引1、3、5、8、9、10上的二进制位的值都为1，而其余所有二进制位的值都为0，这表示节点负责处理槽1、3、5、8、9、10。
>
> ![image-20230923150825015](https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230923150825015.png)

因为取出和设置slots数组中的任意一个二进制位的值的复杂度仅为O（1），所以对于一个给定节点的slots数组来说，程序检查节点是否负责处理某个槽，又或者将某个槽指派给节点负责，这两个动作的复杂度都是O（1）。

至于numslots属性则记录节点负责处理的槽的数量，也即是slots数组中值为1的二进制位的数量。

比如对于上图所示的slots数组来说，节点处理的槽数量为6。

### 传播节点的槽指派信息 - 通过消息

一个节点除了会将自己负责处理的槽记录在clusterNode结构的slots属性和numslots属性之外，**它还会将自己的slots数组通过消息发送给集群中的其他节点，以此来告知其他节点自己目前负责处理哪些槽。**

> 举个例子，对于前面展示的包含7000、7001、7002三个节点的集群来说：
>
> - 节点7000会通过消息向节点7001和节点7002发送自己的slots数组，以此来告知这两个节点，自己负责处理槽0至槽5000
> - 节点7001会通过消息告知7000, 7002这两个节点，自己负责处理槽5001至槽10000
> - 节点7002会通过消息告知7000, 7001这两个节点，自己负责处理槽10001至槽16383
>
> ![image-20230923151111514](https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230923151111514.png)
>
> 节点7001和节点7002类似

**当节点A通过消息从节点B那里接收到节点B的slots数组时，节点A会在自己的clusterState.nodes字典中查找节点B对应的clusterNode结构，并对结构中的slots数组进行保存或者更新。**

因为集群中的每个节点都会将自己的slots数组通过消息发送给集群中的其他节点，并且每个接收到slots数组的节点都会将数组保存到相应节点的clusterNode结构里面，**因此，集群中的每个节点都会知道数据库中的16384个槽分别被指派给了集群中的哪些节点。**

常规/正常情况下, 每个节点中记录同一个节点状态的clusterNode中的slots是相同的, 都是该节点当前负责的槽位

### 记录集群所有槽的指派信息

**clusterState结构中的slots数组记录了集群中所有16384个槽的指派信息：**

```
typedef struct clusterState {
  // ...
  clusterNode *slots[16384];
  // ...
} clusterState;
```

**slots数组包含16384个项，每个数组项都是一个指向clusterNode结构的指针**：若槽位号为i的槽位指派给了某节点, 则数组的i下标出存储该节点对应的clusterNode结构, 若尚未指派给任何节点, 则存储NULL

> 举个例子，对于7000、7001、7002三个节点来说，它们的clusterState结构的slots数组将会是下图所示的样子：
>
> - 数组项slots[0]至slots[5000]的指针都指向代表节点7000的clusterNode结构，表示槽0至5000都指派给了节点7000。
> - 数组项slots[5001]至slots[10000]的指针都指向代表节点7001的clusterNode结构，表示槽5001至10000都指派给了节点7001。
> - 数组项slots[10001]至slots[16383]的指针都指向代表节点7002的clusterNode结构，表示槽10001至16383都指派给了节点7002。
>
> <img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230923151428606.png" alt="image-20230923151428606" style="zoom: 50%;" />

**为什么同时需要clusterNode.slots 和 clusterState.slots ?**

>clusterState.slots的作用
>
>- 如果节点只使用clusterNode.slots数组来记录槽的指派信息，那么为了知道槽i是否已经被指派，或者槽i被指派给了哪个节点，程序需要遍历clusterState.nodes字典中的所有clusterNode结构，检查这些结构的slots数组，直到找到负责处理槽i的节点为止，这个过程的复杂度为O（N），其中N为clusterState.nodes字典保存的clusterNode结构的数量。(一般来说, N不会很多 = =)
>  <u>而通过将所有槽的指派信息保存在clusterState.slots数组里面，程序要检查槽i是否已经被指派，又或者取得负责处理槽i的节点，只需要访问clusterState.slots[i]的值即可，这个操作的复杂度仅为O（1）</u>
>
>  举个例子，对于下图所示的slots数组来说，如果程序需要知道槽10002被指派给了哪个节点，那么只要访问数组项slots[10002]，就可以马上知道槽10002被指派给了节点7002
>
>  <img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230923151640962.png" alt="image-20230923151640962" style="zoom:67%;" />
>
>clusterNode.slots的作用
>
>- 当程序需要将某个节点的槽指派信息通过消息发送给其他节点时，程序只需要将相应节点的clusterNode.slots数组整个发送出去就可以了。如果Redis不使用clusterNode.slots数组，而单独使用clusterState.slots数组的话，那么每次要将节点A的槽指派信息传播给其他节点时，程序必须先遍历整个clusterState.slots数组，记录节点A负责处理哪些槽，然后才能发送节点A的槽指派信息，这比直接发送clusterNode.slots数组要麻烦和低效得多。时间复杂度为O(16384)

clusterState.slots数组记录了集群中所有槽的指派信息，而clusterNode.slots数组只记录了clusterNode结构所代表的节点的槽指派信息，这是两个slots数组的关键区别所在。

### CLUSTER ADDSLOTS命令的实现

**CLUSTER ADDSLOTS命令接受一个或多个槽作为参数，并将所有输入的槽指派给接收该命令的节点负责**：

```
CLUSTER ADDSLOTS <slot> [slot ...]
```

CLUSTER ADDSLOTS命令的实现:

0. 遍历所有输入槽，检查它们是否都是未指派槽 - clusterState.slots[i] == NULL i为所有参数, 若有任何一个槽已经被指派给了某个节点, 则向客户端返回错误，并终止命令执行
1. 设置clusterState结构的slots数组, 将slots[i]的指针指向代表当前节点的clusterNode结构
2. 访问代表当前节点的clusterNode结构的slots数组, 将数组在索引i上的二进制位设置为1

伪代码:

```
def CLUSTER_ADDSLOTS(*all_input_slots):
    # 遍历所有输入槽，检查它们是否都是未指派槽
    for i in all_input_slots:
        # 如果有哪怕一个槽已经被指派给了某个节点
        # 那么向客户端返回错误，并终止命令执行
        if clusterState.slots[i] != NULL:
            reply_error()
            return
    # 如果所有输入槽都是未指派槽
    # 那么再次遍历所有输入槽，将这些槽指派给当前节点
    for i in all_input_slots:
        # 设置clusterState结构的slots数组                    // !!!!
        # 将slots[i]的指针指向代表当前节点的clusterNode结构
        clusterState.slots[i] = clusterState.myself
        # 访问代表当前节点的clusterNode结构的slots数组         // !!!!
        # 将数组在索引i上的二进制位设置为1
        setSlotBit(clusterState.myself.slots, i)
```

> 举个例子，下图展示了一个节点的clusterState结构，clusterState.slots数组中的所有指针都指向NULL，并且clusterNode.slots数组中的所有二进制位的值都是0，这说明当前节点没有被指派任何槽，并且集群中的所有槽都是未指派的。
>
> <img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230923213343316.png" alt="image-20230923213343316" style="zoom: 50%;" />
>
> 当客户端对所示的节点执行命令：
>
> ```
> CLUSTER ADDSLOTS 1 2
> ```
>
> 将槽1和槽2指派给节点之后，节点的clusterState结构将被更新成下图所示的样子：
>
> - clusterState.slots数组在索引1和索引2上的指针指向了代表当前节点的clusterNode结构。
> - 并且clusterNode.slots数组在索引1和索引2上的位被设置成了1。
>
> <img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230923213359884.png" alt="image-20230923213359884" style="zoom: 67%;" />
>
> 最后，在CLUSTER ADDSLOTS命令执行完毕之后，节点会通过<u>发送消息</u>告知集群中的其他节点，自己目前正在负责处理哪些槽。

## 3. 在集群中执行命令

在对数据库中的16384个槽都进行了指派之后，集群就会进入上线状态，这时客户端就可以向集群中的节点发送数据命令了。

当客户端向节点发送与数据库键有关的命令时，接收命令的节点会计算出命令要处理的数据库键属于哪个槽，并检查这个槽是否指派给了自己：

- 如果键所在的槽正好就指派给了当前节点，那么节点直接执行这个命令。
- <u>如果键所在的槽并没有指派给当前节点，那么节点会向客户端返回一个MOVED错误，指引客户端转向（redirect）至正确的节点，并再次发送之前想要执行的命令。</u>

<img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230923153144476.png" alt="image-20230923153144476" style="zoom:67%;" />

> 举个例子，如果我们在之前提到的，由7000、7001、7002三个节点组成的集群中，用客户端连上节点7000，并发送以下命令，那么命令会直接被节点7000执行：
>
> ```
> 127.0.0.1:7000> SET date "2013-12-31"
> OK
> ```
>
> 因为键date所在的槽2022正是由节点7000负责处理的。
>
> 但是，如果我们执行以下命令，那么客户端会先被转向至节点7001，然后再执行命令：
>
> ```
> 127.0.0.1:7000> SET msg "happy new year!"
> -> Redirected to slot [6257] located at 127.0.0.1:7001
> OK
> 127.0.0.1:7001> GET msg
> "happy new year!"
> ```
>
> 这是因为键msg所在的槽6257是由节点7001负责处理的，而不是由最初接收命令的节点7000负责处理：
>
> - 当客户端第一次向节点7000发送SET命令的时候，节点7000会向客户端返回MOVED错误，指引客户端转向至节点7001。
> - 当客户端转向到节点7001之后，客户端重新向节点7001发送SET命令，这个命令会被节点7001成功执行。

本节接下来的内容将介绍计算键所属槽的方法，节点判断某个槽是否由自己负责的方法，以及MOVED错误的实现方法，最后，本节还会介绍节点和单机Redis服务器保存键值对数据的相同和不同之处。

### 计算键属于哪个槽

节点使用以下算法来计算给定键key属于哪个槽：

```c
def slot_number(key):
    return CRC16(key) & 16383
```

其中CRC16（key）语句用于计算键key的CRC-16校验和，而&16383语句则用于计算出一个介于0至16383之间的整数作为键key的槽号。

使用CLUSTER KEYSLOT命令可以查看一个给定键属于哪个槽：

```
127.0.0.1:7000> CLUSTER KEYSLOT "date"
(integer) 2022
127.0.0.1:7000> CLUSTER KEYSLOT "msg"
(integer) 6257
127.0.0.1:7000> CLUSTER KEYSLOT "name"
(integer) 5798
127.0.0.1:7000> CLUSTER KEYSLOT "fruits"
(integer) 14943
```

CLUSTER KEYSLOT命令就是通过调用上面给出的槽分配算法来实现的，以下是该命令的伪代码实现：

```
def CLUSTER_KEYSLOT(key):
    # 计算槽号
    slot = slot_number(key)
    # 将槽号返回给客户端
    reply_client(slot)
```

### 判断槽是否由当前节点负责处理

当节点计算出键所属的槽i之后，<u>节点就会检查clusterState.slots数组中的项i，根据clusterState.slots[i]是否等于clusterState.myself来判断键所在的槽是否由自己负责：</u>

如果clusterState.slots[i]等于clusterState.myself，那么说明槽i由当前节点负责，<u>节点可以执行客户端发送的命令。</u>

### MOVED错误

<u>当节点发现键所在的槽并非由自己负责处理的时候，节点就会向客户端返回一个MOVED错误，指引客户端转向至正在负责槽的节点。</u>

MOVED错误的格式为：

```
MOVED <slot> <ip>:<port>
```

其中slot为键所在的槽，而ip和port则是负责处理槽slot的节点的IP地址和端口号。例如错误：

```
MOVED 10086 127.0.0.1:7002
```

表示槽10086正由IP地址为127.0.0.1，端口号为7002的节点负责。

<u>当客户端接收到节点返回的MOVED错误时，客户端会根据MOVED错误中提供的IP地址和端口号，转向至负责处理槽slot的节点，并向该节点重新发送之前想要执行的命令。</u>

> 以前面的客户端从节点7000转向至7001的情况作为例子：
>
> ```
> 127.0.0.1:7000> SET msg "happy new year!"
> -> Redirected to slot [6257] located at 127.0.0.1:7001
> OK
> 127.0.0.1:7001>
> ```
>
> 客户端向节点7000发送SET命令，并获得MOVED错误的过程。
>
> ![image-20230923153745160](https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230923153745160.png)
>
> 客户端根据MOVED错误，转向至节点7001，并重新发送SET命令的过程。
>
> ![image-20230923153847333](https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230923153847333.png)

**<u>一个集群客户端通常会与集群中的多个节点创建套接字连接，而所谓的节点转向实际上就是换一个套接字来发送命令。</u>**

**如果客户端尚未与想要转向的节点创建套接字连接，那么客户端会先根据MOVED错误提供的IP地址和端口号来连接节点，然后再进行转向。**

> 被隐藏的MOVED错误
>
> 集群模式的redis-cli客户端在接收到MOVED错误时，并不会打印出MOVED错误，而是根据MOVED错误自动进行节点转向，并打印出转向信息，所以我们是看不见节点返回的MOVED错误的：
>
> ```c
> $ redis-cli -c -p 7000 # 集群模式
> 127.0.0.1:7000> SET msg "happy new year!"
> -> Redirected to slot [6257] located at 127.0.0.1:7001
> OK
> 127.0.0.1:7001>
> ```
>
> 但是，如果我们使用单机（stand alone）模式的redis-cli客户端，再次向节点7000发送相同的命令，那么MOVED错误就会被客户端打印出来：
>
> ```c
> $ redis-cli -p 7000 # 
> 单机模式
> 127.0.0.1:7000> SET msg "happy new year!"
> (error) MOVED 6257 127.0.0.1:7001
> 127.0.0.1:7000>
> ```
>
> 这是因为单机模式的redis-cli客户端不清楚MOVED错误的作用，所以它只会直接将MOVED错误直接打印出来，而不会进行自动转向。

### 节点数据库的实现

集群节点保存键值对以及键值对过期时间的方式，与第9章里面介绍的单机Redis服务器保存键值对以及键值对过期时间的方式完全相同。

<u>节点和单机服务器在数据库方面的一个区别是，节点只能使用0号数据库，而单机Redis服务器则没有这一限制。</u>

举个例子，节点7000的数据库状态，数据库中包含列表键"lst"，哈希键"book"，以及字符串键"date"，其中键"lst"和键"book"带有过期时间。

![image-20230923154141082](https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230923154141082.png)

<u>另外，除了将键值对保存在数据库里面之外，节点(集群模式的Redis服务器)还会用clusterState结构中的slots_to_keys跳跃表来保存槽和键之间的关系：</u>

```
typedef struct clusterState {
  // ...
  zskiplist *slots_to_keys;
  // ...
} clusterState;
```

<u>slots_to_keys跳跃表每个节点的分值（score）都是一个槽号，而每个节点的成员（member）都是一个数据库键：(按照槽位号对数据库键进行排序)</u>

- 每当节点往数据库中添加一个新的键值对时，节点就会将这个键以及键的槽号关联到slots_to_keys跳跃表。
- 当节点删除数据库中的某个键值对时，节点就会在slots_to_keys跳跃表解除被删除键与槽号的关联。

举例: 节点7000将创建类似下图所示的slots_to_keys跳跃表：

- 键"book"所在跳跃表节点的分值为1337.0，这表示键"book"所在的槽为1337。
- 键"date"所在跳跃表节点的分值为2022.0，这表示键"date"所在的槽为2022。
- 键"lst"所在跳跃表节点的分值为3347.0，这表示键"lst"所在的槽为3347。

**通过在slots_to_keys跳跃表中记录各个数据库键所属的槽，节点可以很方便地对属于某个或某些槽的所有数据库键进行批量操作，例如命令CLUSTER GETKEYSINSLOT命令可以返回最多count个属于槽slot的数据库键，而这个命令就是通过遍历slots_to_keys跳跃表来实现的。**

![image-20230923154417537](https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230923154417537.png)

> 总结:
>
> **clusterNode结构的slots属性(位图)和numslot属性记录了节点负责处理哪些槽**
>
> **clusterState结构中的slots数组(clusterNode*指针数据)记录了集群中16384个槽的指派信息, 即每个槽指派给了哪个clusterNode, 就存储指向对应clusterNode的指针**
>
> clusterNode.slots可以很方便的记录/传输节点当前负责的槽位(如果用cluster.State就需要遍历)
>
> clusterState.slots记录每个槽位的指派情况, 可以很方便的查询某个槽位当前指派给了哪个节点(如果用clusterNode.slots就需要遍历所有的clusterNode)
>
> 上面针对的都是 槽位 和 节点之间的关系: 节点负责了哪些槽, 每个槽指派给了哪些节点.
>
> 也就导致了仅有上方的数据无法支持如下操作: 快速取出某槽位/某些槽位中的键值, 此时**使用clusterState的zskiplist *slots_to_keys数据成员即可.  它记录了槽位和键值之间的关系, 并按照槽位号对键值进行了排序**

## 4. 重新分片

**Redis集群的重新分片操作可以将任意数量已经指派给某个节点（源节点）的槽改为指派给另一个节点（目标节点），并且相关槽所属的键值对也会从源节点被移动到目标节点。**

<u>重新分片操作可以在线（online）进行，在重新分片的过程中，集群不需要下线，并且源节点和目标节点都可以继续处理命令请求。</u>

> 举个例子，对于之前提到的，包含7000、7001、7002三个节点的集群来说，我们可以向这个集群添加一个IP为127.0.0.1，端口号为7003的节点（后面简称节点7003）：
>
> ```
> $ redis-cli -c -p 7000
> 127.0.0.1:7000> CLUSTER MEET 127.0.0.1 7003
> OK
> 127.0.0.1:7000> cluster nodes
> 51549e625cfda318ad27423a31e7476fe3cd2939 :0 myself,master - 0 0 0 connected 0-5000
> 68eef66df23420a5862208ef5b1a7005b806f2ff 127.0.0.1:7001 master - 0 1388635782831 0 connected 5001-10000
> 9dfb4c4e016e627d9769e4c9bb0d4fa208e65c26 127.0.0.1:7002 master - 0 1388635782831 0 connected 10001-16383
> 04579925484ce537d3410d7ce97bd2e260c459a2 127.0.0.1:7003 master - 0 1388635782330 0 connected
> ```
>
> 然后通过重新分片操作，将原本指派给节点7002的槽15001至16383改为指派给节点7003。
>
> 以下是重新分片操作执行之后，节点的槽分配状态：
>
> ```
> 127.0.0.1:7000> cluster nodes
> 51549e625cfda318ad27423a31e7476fe3cd2939 :0 myself,master -0 0 0 connected 0-5000
> 68eef66df23420a5862208ef5b1a7005b806f2ff 127.0.0.1:7001 master -0 1388635782831 0 connected 5001-10000
> 9dfb4c4e016e627d9769e4c9bb0d4fa208e65c26 127.0.0.1:7002 master -0 1388635782831 0 connected 10001-15000
> 04579925484ce537d3410d7ce97bd2e260c459a2 127.0.0.1:7003 master -0 1388635782330 0 connected 15001-16383
> ```
>

### 重新分片的实现原理

**Redis集群的重新分片操作是由Redis的集群管理软件redis-trib负责执行的，Redis提供了进行重新分片所需的所有命令，而redis-trib则通过向源节点和目标节点发送命令来进行重新分片操作。**

<u>redis-trib对集群的单个槽slot进行重新分片的步骤如下</u>：

1）redis-trib对目标节点发送CLUSTER SETSLOT \<slot> IMPORTING <source_id>命令，让目标节点准备好从源节点导入（import）属于槽slot的键值对。

2）redis-trib对源节点发送CLUSTER SETSLOT <slot\> MIGRATING <target_id>命令，让源节点准备好将属于槽slot的键值对迁移（migrate）至目标节点。

3）redis-trib向源节点发送CLUSTER GETKEYSINSLOT \<slot> \<count>命令，获得最多count个属于槽slot的键值对的键名（key name）。

4）对于步骤3获得的每个键名，redis-trib都向源节点发送一个MIGRATE ...命令，**将被选中的键原子地从源节点迁移至目标节点。**

5）重复执行步骤3和步骤4，直到源节点保存的所有属于槽slot的键值对都被迁移至目标节点为止。

6）redis-trib向集群中的任意一个节点发送CLUSTER SETSLOT \<slot> NODE <target_id>命令，将槽slot指派给目标节点，这一指派信息会<u>通过消息</u>发送至整个集群，最终集群中的所有节点都会知道槽slot已经指派给了目标节点。

如果重新分片涉及多个槽，那么redis-trib将对每个给定的槽分别执行上面给出的步骤。

<img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230923160047531.png" alt="image-20230923160047531" style="zoom:67%;" />

<img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230923160059437.png" alt="image-20230923160059437" style="zoom:67%;" />

## 5. ASK错误

<u>在进行重新分片期间，源节点向目标节点迁移一个槽的过程中，可能会出现这样一种情况：属于被迁移槽的一部分键值对保存在源节点里面，而另一部分键值对则保存在目标节点里面。</u>

当客户端向源节点发送一个与数据库键有关的命令，并且命令要处理的数据库键恰好就属于正在被迁移的槽时：

- 源节点会先在自己的数据库里面查找指定的键，如果找到的话，就直接执行客户端发送的命令。

- 相反地，如果源节点没能在自己的数据库里面找到指定的键，<u>那么这个键有可能已经被迁移到了目标节点，源节点将向客户端返回一个ASK错误，指引客户端转向正在导入槽的目标节点，并再次发送之前想要执行的命令。</u>(和MOVED错误很像)

源节点判断是否需要向客户端发送ASK错误的整个过程。

<img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230923160238609.png" alt="image-20230923160238609" style="zoom:67%;" />

> 举个例子，假设节点7002正在向节点7003迁移槽16198，这个槽包含"is"和"love"两个键，其中键"is"还留在节点7002，而键"love"已经被迁移到了节点7003。
>
> 如果我们向节点7002发送关于键"is"的命令，那么这个命令会直接被节点7002执行：
>
> ```
> 127.0.0.1:7002> GET "is"
> "you get the key 'is'"Copy to clipboardErrorCopied
> ```
>
> 而如果我们向节点7002发送关于键"love"的命令，那么客户端会先被转向至节点7003，然后再次执行命令：
>
> ```
> 127.0.0.1:7002> GET "love"
> -> Redirected to slot [16198] located at 127.0.0.1:7003
> "you get the key 'love'"
> 127.0.0.1:7003>
> ```
>

> 被隐藏的ASK错误
>
> 和接到MOVED错误时的情况类似，集群模式的redis-cli在接到ASK错误时也不会打印错误，而是自动根据错误提供的IP地址和端口进行转向动作。如果想看到节点发送的ASK错误的话，可以使用单机模式的redis-cli客户端：
>
> ```
> $ redis-cli -p 7002
> 127.0.0.1:7002> GET "love"
> (error) ASK 16198 127.0.0.1:7003
> ```

> **注意**
>
> 在写这篇文章的时候，集群模式的redis-cli并未支持ASK自动转向，上面展示的ASK自动转向行为实际上是根据MOVED自动转向行为虚构出来的。因此，当集群模式的redis-cli真正支持ASK自动转向时，它的行为和上面展示的行为可能会有所不同。
>
> 本节将对ASK错误的实现原理进行说明，并对比ASK错误和MOVED错误的区别。

### CLUSTER SETSLOT IMPORTING命令的实现

**clusterState结构的importing_slots_from数组记录了当前节点正在从其他节点导入的槽：**

```
typedef struct clusterState {
  // ...
  clusterNode *importing_slots_from[16384];
  // ...
} clusterState;Copy to clipboardErrorCopied
```

<u>如果importing_slots_from[i]的值不为NULL，而是指向一个clusterNode结构，那么表示当前节点正在从clusterNode所代表的节点导入槽i。</u>

在对集群进行重新分片的时候，向目标节点发送命令：

```
CLUSTER SETSLOT <i> IMPORTING <source_id>
```

可以将目标节点clusterState.importing_slots_from[i]的值设置为source_id所代表节点的clusterNode结构。

> 举个例子，如果客户端向节点7003发送以下命令：
>
> ```
> # 9dfb... 是节点7002 的ID 
> 127.0.0.1:7003> CLUSTER SETSLOT 16198 IMPORTING 9dfb4c4e016e627d9769e4c9bb0d4fa208e65c26
> OK
> ```
>
> 那么节点7003的clusterState.importing_slots_from数组将变成下图所示的样子。
>
> <img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230923160527699.png" alt="image-20230923160527699" style="zoom:67%;" />

### CLUSTER SETSLOT MIGRATING命令的实现

**clusterState结构的migrating_slots_to数组记录了当前节点正在迁移至其他节点的槽：**

```
typedef struct clusterState {
   // ...
   clusterNode *migrating_slots_to[16384];
   // ...
} clusterState;
```

<u>如果migrating_slots_to[i]的值不为NULL，而是指向一个clusterNode结构，那么表示当前节点正在将槽i迁移至clusterNode所代表的节点。</u>

在对集群进行重新分片的时候，向源节点发送命令：

```
CLUSTER SETSLOT <i> MIGRATING <target_id>
```

可以将源节点clusterState.migrating_slots_to[i]的值设置为target_id所代表节点的clusterNode结构。

> 举个例子，如果客户端向节点7002发送以下命令：
>
> ```
> # 0457... 是节点7003 的ID 
> 127.0.0.1:7002> CLUSTER SETSLOT 16198 MIGRATING 04579925484ce537d3410d7ce97bd2e260c459a2
> OK
> ```
>
> 那么节点7002的clusterState.migrating_slots_to数组将变成图17-28所示的样子。
>
> <img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230923160747583.png" alt="image-20230923160747583" style="zoom:67%;" />

### ASK错误

如果节点收到一个关于键key的命令请求，并且键key所属的槽i正好就指派给了这个节点，那么节点会尝试在自己的数据库里查找键key，如果找到了的话，节点就直接执行客户端发送的命令。

**与此相反，如果节点没有在自己的数据库里找到键key，那么节点会检查自己的clusterState.migrating_slots_to[i]，看键key所属的槽i是否正在进行迁移，如果槽i的确在进行迁移的话，那么节点会向客户端发送一个ASK错误，引导客户端到正在导入槽i的节点去查找键key。**

举个例子，假设在节点7002向节点7003迁移槽16198期间，有一个客户端向节点7002发送命令：

```
GET “love”
```

因为键"love"正好属于槽16198，所以节点7002会首先在自己的数据库中查找键"love"，但并没有找到，通过检查自己的clusterState.migrating_slots_to[16198]，节点7002发现自己正在将槽16198迁移至节点7003，于是它向客户端返回错误：

```
ASK 16198 127.0.0.1:7003
```

这个错误表示客户端可以尝试到IP为127.0.0.1，端口号为7003的节点去执行和槽16198有关的操作

![image-20230923160933163](https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230923160933163.png)

**接到ASK错误的客户端会根据错误提供的IP地址和端口号，转向至正在导入槽的目标节点，<u>然后首先向目标节点发送一个ASKING命令，之后再重新发送原本想要执行的命令。</u>**

以前面的例子来说，当客户端接收到节点7002返回的以下错误时：

```
ASK 16198 127.0.0.1:7003
```

客户端会转向至节点7003，首先发送命令：

```
ASKING
```

然后再次发送命令：

```
GET "love"
```

并获得回复：

```
"you get the key 'love'"
```

![image-20230923161035169](https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230923161035169.png)

### ASKING命令

**ASKING命令唯一要做的就是打开发送该命令的客户端的REDIS_ASKING标识**，以下是该命令的伪代码实现：

```
def ASKING():
    # 打开标识
    client.flags |= REDIS_ASKING
    # 向客户端返回OK 回复
    reply("OK")
```

<u>在一般情况下，如果客户端向节点发送一个关于槽i的命令，而槽i又没有指派给这个节点的话，那么节点将向客户端返回一个MOVED错误；但是，如果节点的clusterState.importing_slots_from[i]\(该clusterNode *指针数组记录了正在从其他节点导入到当前节点的槽位)显示当前节点正在从其他节点(其实就是那个数组里存储的指针指向的clusterNode所代表的节点)导入槽i，并且发送命令的客户端带有REDIS_ASKING标识，那么节点将破例执行这个关于槽i的命令一次</u>

**当客户端接收到ASK错误并转向至正在导入槽的节点时，客户端会先向节点发送一个ASKING命令，然后才重新发送想要执行的命令，这是因为如果客户端不发送ASKING命令，而直接发送想要执行的命令的话，那么客户端发送的命令将被节点拒绝执行，并返回MOVED错误。**(因为根本上此刻这个键值对对应的槽位还没有指派给这个目标节点, 而是正在导入这个节点)

举个例子，我们可以使用普通模式的redis-cli客户端，向正在导入槽16198的节点7003发送以下命令：

```
$ ./redis-cli -p 7003
127.0.0.1:7003> GET "love"
(error) MOVED 16198 127.0.0.1:7002
```

<u>虽然节点7003正在导入槽16198，但槽16198目前仍然是指派给了节点7002</u>，所以节点7003会向客户端返回MOVED错误，指引客户端转向至节点7002。

但是，如果我们在发送GET命令之前，先向节点发送一个ASKING命令，那么这个GET命令就会被节点7003执行：

```
127.0.0.1:7003> ASKING
OK
127.0.0.1:7003> GET "love"
"you get the key 'love'"
```

**另外要注意的是，客户端的REDIS_ASKING标识是一个一次性标识，当节点执行了一个带有REDIS_ASKING标识的客户端发送的命令之后，客户端的REDIS_ASKING标识就会被移除。**

<u>举个例子，如果我们在成功执行GET命令之后，再次向节点7003发送GET命令，那么第二次发送的GET命令将执行失败，因为这时客户端的REDIS_ASKING标识已经被移除：</u>

```
127.0.0.1:7003> ASKING       #打开REDIS_ASKING标识
OK
127.0.0.1:7003> GET "love"   #移除REDIS_ASKING标识
"you get the key 'love'"
127.0.0.1:7003> GET "love"   # REDIS_ASKING标识未打开，执行失败
(error) MOVED 16198 127.0.0.1:7002
```

### ASK错误和MOVED错误的区别

ASK错误和MOVED错误都会导致客户端转向，它们的区别在于：

- <u>MOVED错误代表客户端目前访问的节点根本不持有对应槽位的负责权/或者说槽的负责权已经从一个节点转移到了另一个节点(对应的键值对也已经全部转移完毕)</u>：在客户端收到关于槽i的MOVED错误之后，客户端每次遇到关于槽i的命令请求时，都可以直接将命令请求发送至MOVED错误所指向的节点，因为该节点就是目前负责槽i的节点。

  例如: 键"redis"的槽位为111, 而目前槽111的负责权在节点B上, 当客户端向节点A发送get redis时, 就会遇到MOVED错误, 并自动转向节点B, 再次发送get redis命令

- <u>与此相反，ASK错误只是两个节点在迁移槽的过程中使用的一种临时措施</u>：在客户端收到关于槽i的ASK错误之后，客户端只会在接下来的一次命令请求中将关于槽i的命令请求发送至ASK错误所指示的节点，但这种转向不会对客户端今后发送关于槽i的命令请求产生任何影响，客户端仍然会将关于槽i的命令请求发送至目前负责处理槽i的节点，除非ASK错误再次出现。

  例如: 键"redis"的槽位为111, 且槽位111的负责权正在从节点A转向节点B, 且此时键redis已经被原子的迁移到了节点B(其余节点不确定, 可能尚未迁移完毕), 则此时客户端向节点A发送get redis, 就会收到ASK错误.

## 6. 复制与故障转移

**Redis集群中的节点分为主节点（master）和从节点（slave），其中主节点用于处理槽，而从节点则用于复制某个主节点，并在被复制的主节点下线时，代替下线主节点继续处理命令请求。** 一个主节点和若干个复制此主节点的从节点称为Redis集群中的一个分片

> 可看不可不看= =
>
> 举个例子，对于包含7000、7001、7002、7003四个主节点的集群来说，我们可以将7004、7005两个节点添加到集群里面，并将这两个节点设定为节点7000的从节点（图中以双圆形表示主节点，单圆形表示从节点）。
>
> <img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230924173449227.png" alt="image-20230924173449227" style="zoom: 67%;" />
>
> 下表记录了集群各个节点的当前状态，以及它们正在做的工作。
>
> <img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230923161558484.png" alt="image-20230923161558484" style="zoom: 67%;" />
>
> 如果这时，节点7000进入下线状态，那么集群中仍在正常运作的几个主节点将在节点7000的两个从节点——节点7004和节点7005中选出一个节点作为新的主节点，这个新的主节点将接管原来节点7000负责处理的槽，并继续处理客户端发送的命令请求。(这里的, 剩余的几个主节点就类似于哨兵模式里若干个哨兵节点, 它们负责挑选出一个新的主节点, 而这里是剩余的几个主节点负责挑选)
>
> 例如，如果节点7004被选中为新的主节点，那么节点7004将接管原来由节点7000负责处理的槽0至槽5000，节点7005也会从原来的复制节点7000，改为复制节点7004（图中用虚线包围的节点为已下线节点）。
>
> <img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230923161614280.png" alt="image-20230923161614280" style="zoom:67%;" />
>
> <img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230924173523806.png" alt="image-20230924173523806" style="zoom:67%;" />
>
> 如果在故障转移完成之后，下线的节点7000重新上线，那么它将成为节点7004的从节点
>
> <img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230924173534309.png" alt="image-20230924173534309" style="zoom:67%;" />
>
> <img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230924173707623.png" alt="image-20230924173707623" style="zoom:67%;" />

本节接下来的内容将介绍节点的复制方法，检测节点是否下线的方法，以及对下线主节点进行故障转移的方法。

### 设置从节点&主从节点相关数据结构

向一个节点发送命令：

```
CLUSTER REPLICATE <node_id>
```

**可以让接收命令的节点成为node_id所指定节点的从节点，并开始对主节点进行复制**：

- 接收到该命令的节点首先会在自己的clusterState.nodes字典中找到node_id所对应节点的clusterNode结构，**并将自己的clusterState.myself.slaveof指针指向这个结构**，以此来记录这个节点正在复制的主节点：

  ```
  struct clusterNode {
       // ...
       // 如果这是一个从节点，那么指向主节点
       struct clusterNode *slaveof;
       // ...
  };
  ```

- 然后节点会修改自己在clusterState.myself.flags中的属性，关闭原本的REDIS_NODE_MASTER标识，打开REDIS_NODE_SLAVE标识，表示这个节点已经由原来的主节点变成了从节点。

- 最后，节点会调用复制代码，并根据clusterState.myself.slaveof指向的clusterNode结构所保存的IP地址和端口号，**对主节点进行复制。**因为节点的复制功能和单机Redis服务器的复制功能使用了相同的代码，所以让从节点复制主节点相当于向从节点发送命令SLAVEOF \<master_ip> \<master_port>。

节点7004在复制节点7000时的clusterState结构：

<img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230924173909036.png" alt="image-20230924173909036" style="zoom: 50%;" />

- clusterState.myself.flags属性的值为REDIS_NODE_SLAVE，表示节点7004是一个从节点。
- clusterState.myself.slaveof指针指向代表节点7000的结构，表示节点7004正在复制的主节点为节点7000。

<u>一个节点成为从节点，并开始复制某个主节点这一信息会**通过消息**发送给集群中的其他节点，最终集群中的所有节点都会知道某个从节点正在复制某个主节点。</u>

集群中的所有节点都会在代表主节点的clusterNode结构的slaves属性和numslaves属性中记录正在复制这个主节点的从节点名单：

```
struct clusterNode {
    // ...
    // 正在复制这个主节点的从节点数量
    int numslaves;
    // 一个数组
    // 每个数组项指向一个正在复制这个主节点的从节点的clusterNode结构
    struct clusterNode **slaves;
    // ...
};
```

举个例子，下图为节点7004和节点7005成为节点7000的从节点之后，集群中的各个节点为节点7000创建的clusterNode结构的样子：

- 代表节点7000的clusterNode结构的numslaves属性的值为2，这说明有两个从节点正在复制节点7000。
- 代表节点7000的clusterNode结构的slaves数组的两个项分别指向代表节点7004和代表节点7005的clusterNode结构，这说明节点7000的两个从节点分别是节点7004和节点7005。

<img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230924174015036.png" alt="image-20230924174015036" style="zoom:50%;" />

### 故障检测

集群中的每个节点都会定期地向集群中的其他节点发送PING消息，以此来检测对方是否在线，如果接收PING消息的节点没有在规定的时间内，向发送PING消息的节点返回PONG消息，<u>那么发送PING消息的节点就会将接收PING消息的节点标记为疑似下线（probable fail，PFAIL）。</u>(标记疑似下线的方式: PING发送方节点(处于集群状态的Redis服务器)会在自己的clusterState.nodes字典中找到接收PING消息的节点对应的clusterNode结构, 将其中的flags打开REDIS_NODE_PFAIL标记)

<img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230924183423616.png" alt="image-20230924183423616" style="zoom: 67%;" />

集群中的各个节点会通过互相发送消息的方式来交换集群中各个节点的状态信息，例如某个节点认为其他某个节点是处于在线状态、疑似下线状态（PFAIL），还是已下线状态（FAIL）。

当一个主节点A通过消息得知主节点B认为主节点C进入了疑似下线状态时，主节点A会在自己的clusterState.nodes字典中找到主节点C所对应的clusterNode结构，并将主节点B的下线报告（failure report）添加到A结点中存储的C对应的clusterNode结构的fail_reports链表里面：(也就是每个节点会记录其他的每个节点, 当前有多少个节点认为你有问题)

```
struct clusterNode {
  // ...
  // 一个链表，记录了所有其他节点对该节点的下线报告
  list *fail_reports;
  // ...
};
```

每个下线报告由一个clusterNodeFailReport结构表示：(fail_reports是一个存储clusterNodeFailReport结构的链表)

```
struct clusterNodeFailReport {
  // 报告目标节点已经下线的节点
  struct clusterNode *node;
  // 最后一次从node节点收到下线报告的时间
  // 程序使用这个时间戳来检查下线报告是否过期
  // （与当前时间相差太久的下线报告会被删除）
  mstime_t time;
} typedef clusterNodeFailReport;
```

举个例子，如果主节点7001在收到主节点7002、主节点7003发送的消息后得知，主节点7002和主节点7003都认为主节点7000进入了疑似下线状态，那么主节点7001将为主节点7000创建下图所示的下线报告。(标识7002和7003认为7000为疑似下线状态)

<img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230924174839497.png" alt="image-20230924174839497" style="zoom:67%;" />

**如果在一个集群里面，半数以上负责处理槽的主节点都将某个主节点x报告为疑似下线，那么这个主节点x将被标记为已下线（FAIL），将主节点x标记为已下线的节点会向集群广播一条关于主节点x的FAIL消息，所有收到这条FAIL消息的节点都会立即将主节点x标记为已下线。**

(也就是说, 某个节点可能最先在X节点的下线报告链表中收集到大于半数的其他主节点的对于X节点的疑似下线报告, 此时该节点就会将X标记为已下线(FAIL), 相当于主观下线, 这时它会通过广播消息的方式, 告知其他节点, X节点确认下线了~)

举个例子，对于上图所示的下线报告来说，主节点7002和主节点7003都认为主节点7000进入了下线状态，并且主节点7001也认为主节点7000进入了疑似下线状态（代表主节点7000的结构打开了REDIS_NODE_PFAIL标识），综合起来，在集群四个负责处理槽的主节点里面，有三个都将主节点7000标记为下线，数量已经超过了半数，所以主节点7001会将主节点7000标记为已下线，并向集群广播一条关于主节点7000的**FAIL消息**

<img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230924183450697.png" alt="image-20230924183450697" style="zoom: 50%;" />

### 选举新的主节点

新的主节点是通过选举产生的。

以下是集群选举新的主节点的方法：

1. 集群的配置纪元是一个自增计数器，它的初始值为0。当集群里的某个节点开始一次故障转移操作时，集群配置纪元的值会被增一。
2. 对于每个配置纪元，**集群里每个负责处理槽的主节点都有一次投票的机会，而第一个向主节点要求投票的从节点将获得主节点的投票。**
3. <u>当从节点发现自己正在复制的主节点进入已下线状态时</u>，从节点会向集群广播一条CLUSTERMSG_TYPE_FAILOVER_AUTH_REQUEST消息，要求所有收到这条消息、并且具有投票权的主节点向这个从节点投票。
4. 如果一个主节点具有投票权（它正在负责处理槽），并且这个主节点尚未投票给其他从节点，那么主节点将向要求投票的从节点返回一条CLUSTERMSG_TYPE_FAILOVER_AUTH_ACK消息，表示这个主节点支持从节点成为新的主节点。(也就是该从节点收集到了一票)
5. 每个参与选举的从节点都会根据自己收到了多少条这种消息来统计自己获得了多少主节点的支持。如果集群里有N个具有投票权的主节点，那么当一个从节点收集到大于半数张支持票时(所有主节点的一半)，这个从节点就会当选为新的主节点。

> 因为在每一个配置纪元里面，每个具有投票权的主节点只能投一次票，所以如果有N个主节点进行投票，那么具有大于等于N/2+1张支持票的从节点只会有一个，这确保了新的主节点只会有一个。如果在一个配置纪元里面没有从节点能收集到足够多的支持票，那么集群进入一个新的配置纪元，并再次进行选举，直到选出新的主节点为止。
>
> 这个选举新主节点的方法和第16章介绍的选举领头Sentinel的方法非常相似，因为两者都是基于Raft算法的领头选举（leader election）方法来实现的。

简单来说: 从节点发现自己复制的主节点已下线了(其实这个发现一般是通过接收其他主节点发送的该主节点已下线的消息), 就会向所有节点进行拉票(通过广播消息), 而只有负责处理槽的在线主节点有投票权, 谁最先拉票, 主节点就会投给谁, 第一个收集到N/2+1张票的从节点就会成为新的主节点

### 被选举的从节点进行故障转移

当一个从节点获取到成为新的主节点的资格之后, 就会进行故障转移, 以下是故障转移的执行步骤：

1. 被选中的从节点会执行SLAVEOF no one命令，成为新的主节点。
2. 新的主节点会撤销所有对已下线主节点的槽指派，并将这些槽全部指派给自己。(槽位的负责权的转移)
3. 新的主节点向集群广播一条PONG消息，这条PONG消息可以让集群中的其他节点立即知道这个节点已经由从节点变成了主节点，并且已经接管了原本由已下线节点负责处理的槽。

故障转移完成, 新的主节点开始接收和自己负责处理的槽有关的命令请求

> 感觉不是很细, 其他从节点怎么复制新的主节点呢? 旧主节点恢复如何自动复制新的主节点呢?

## 7. 消息

**集群中的各个节点通过发送和接收消息（message）来进行通信**，我们称发送消息的节点为发送者（sender），接收消息的节点为接收者（receiver）

> Sentinel哨兵机制中, 哨兵节点与主从节点通信主要是通过命令的方式进行的, 也会通过频道~

<img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230924183524762.png" alt="image-20230924183524762" style="zoom:67%;" />

节点发送的消息主要有以下五种：

- **MEET消息**：当发送者(集群模式的Redis服务器)接到客户端发送的CLUSTER MEET命令时，发送者会向接收者发送MEET消息，请求接收者加入到发送者当前所处的集群里面。
- **PING消息**：集群里的每个节点默认每隔一秒钟就会从已知节点列表中随机选出五个节点，然后对这五个节点中最长时间没有发送过PING消息的节点发送PING消息，以此来检测被选中的节点是否在线。除此之外，如果节点A最后一次收到节点B发送的PONG消息的时间，距离当前时间已经超过了节点A的cluster-node-timeout选项设置时长的一半，那么节点A也会向节点B发送PING消息，这可以防止节点A因为长时间没有随机选中节点B作为PING消息的发送对象而导致对节点B的信息更新滞后。
- **PONG消息**：当接收者收到发送者发来的MEET消息或者PING消息时，为了向发送者确认这条MEET消息或者PING消息已到达，接收者会向发送者返回一条PONG消息。另外，一个节点也可以通过向集群广播自己的PONG消息来让集群中的其他节点立即刷新关于这个节点的认识，例如当一次故障转移操作成功执行之后，新的主节点会向集群广播一条PONG消息，以此来让集群中的其他节点立即知道这个节点已经变成了主节点，并且接管了已下线节点负责的槽。
- **FAIL消息**：当一个主节点A判断另一个主节点B已经进入FAIL状态时，节点A会向集群广播一条关于节点B的FAIL消息，所有收到这条消息的节点都会立即将节点B标记为已下线。
- **PUBLISH消息**：当节点接收到一个PUBLISH命令时，节点会执行这个命令，并向集群广播一条PUBLISH消息，所有接收到这条PUBLISH消息的节点都会执行相同的PUBLISH命令。

**一条消息由消息头（header）和消息正文（data）组成**，接下来的内容将首先介绍消息头，然后再分别介绍上面提到的五种不同类型的消息正文。

### 消息头

**节点发送的所有消息都由一个消息头包裹**，消息头除了包含消息正文之外，还记录了消息发送者自身的一些信息，因为这些信息也会被消息接收者用到，所以严格来讲，我们可以认为消息头本身也是消息的一部分。

每个消息头都由一个cluster.h/clusterMsg结构表示：

```c
typedef struct {
  // 消息的长度（包括这个消息头的长度和消息正文的长度）
  uint32_t totlen;
  // 消息的类型
  uint16_t type;
  // 消息正文包含的节点信息数量
  // 只在发送MEET、PING、PONG这三种Gossip协议消息时使用
  uint16_t count;
  // 发送者所处的配置纪元
  uint64_t currentEpoch;
  // 如果发送者是一个主节点，那么这里记录的是发送者的配置纪元
  // 如果发送者是一个从节点，那么这里记录的是发送者正在复制的主节点的配置纪元
  uint64_t configEpoch;
  // 发送者的名字（ID） 
  char sender[REDIS_CLUSTER_NAMELEN];
  // 发送者目前的槽指派信息
  unsigned char myslots[REDIS_CLUSTER_SLOTS/8];
  // 如果发送者是一个从节点，那么这里记录的是发送者正在复制的主节点的名字
  // 如果发送者是一个主节点，那么这里记录的是REDIS_NODE_NULL_NAME
  // （一个40字节长，值全为0的字节数组）
  char slaveof[REDIS_CLUSTER_NAMELEN];
  // 发送者的端口号
  uint16_t port;
  // 发送者的标识值
  uint16_t flags;
  // 发送者所处集群的状态
  unsigned char state;
  // 消息的正文（或者说，内容）
  union clusterMsgData data;
} clusterMsg;
```

clusterMsg.data属性指向联合cluster.h/clusterMsgData，这个联合就是消息的正文：

```
union clusterMsgData {
  // MEET、PING、PONG消息的正文
  struct {
    // 每条MEET、PING、PONG消息都包含两个
    // clusterMsgDataGossip结构
    clusterMsgDataGossip gossip[1];
  } ping;
  // FAIL消息的正文
  struct {
    clusterMsgDataFail about;
  } fail;
  // PUBLISH消息的正文
  struct {
    clusterMsgDataPublish msg;
  } publish;
  // 其他消息的正文... 
};
```

<u>clusterMsg结构的currentEpoch、sender、myslots等属性记录了发送者自身的节点信息，接收者会根据这些信息，在自己的clusterState.nodes字典里找到发送者对应的clusterNode结构，并对结构进行更新。</u>

举个例子，通过对比接收者为发送者记录的槽指派信息，以及发送者在消息头的myslots属性记录的槽指派信息，接收者可以知道发送者的槽指派信息是否发生了变化。

又或者说，通过对比接收者为发送者记录的标识值，以及发送者在消息头的flags属性记录的标识值，接收者可以知道发送者的状态和角色是否发生了变化，例如节点状态由原来的在线变成了下线，或者由主节点变成了从节点等等。

### MEET、PING、PONG消息的实现

Redis集群中的各个节点通过`Gossip`协议来交换各自关于不同节点的状态信息，其中Gossip协议由`MEET`、`PING`、`PONG`三种消息实现，这三种消息的正文都由两个cluster.h/clusterMsgDataGossip结构组成：

```
union clusterMsgData {
  // ...
  // MEET、PING和PONG消息的正文
  struct {
    // 每条MEET、PING、PONG消息都包含两个
    // clusterMsgDataGossip结构
    clusterMsgDataGossip gossip[1];
  } ping;
  // 其他消息的正文... 
};
```

因为MEET、PING、PONG三种消息都使用相同的消息正文，所以节点通过消息头的type属性来判断一条消息是MEET消息、PING消息还是PONG消息。

每次发送MEET、PING、PONG消息时，发送者都从自己的已知节点列表中随机选出两个节点（可以是主节点或者从节点），并将这两个被选中节点的信息分别保存到两个clusterMsgDataGossip结构里面。

clusterMsgDataGossip结构记录了被选中节点的名字，发送者与被选中节点最后一次发送和接收PING消息和PONG消息的时间戳，被选中节点的IP地址和端口号，以及被选中节点的标识值：

```
typedef struct {
  // 节点的名字
  char nodename[REDIS_CLUSTER_NAMELEN];
  // 最后一次向该节点发送PING消息的时间戳
  uint32_t ping_sent;
  // 最后一次从该节点接收到PONG消息的时间戳
  uint32_t pong_received;
  // 节点的IP地址
  char ip[16];
  // 节点的端口号
  uint16_t port;
  // 节点的标识值
  uint16_t flags;
} clusterMsgDataGossip;
```

当接收者收到MEET、PING、PONG消息时，接收者会访问消息正文中的两个clusterMsgDataGossip结构，并根据自己是否认识clusterMsgDataGossip结构中记录的被选中节点来选择进行哪种操作：

- 如果被选中节点不存在于接收者的已知节点列表，那么说明接收者是第一次接触到被选中节点，接收者将根据结构中记录的IP地址和端口号等信息，与被选中节点进行握手。
- 如果被选中节点已经存在于接收者的已知节点列表，那么说明接收者之前已经与被选中节点进行过接触，接收者将根据clusterMsgDataGossip结构记录的信息，对被选中节点所对应的clusterNode结构进行更新。

举个发送PING消息和返回PONG消息的例子，假设在一个包含A、B、C、D、E、F六个节点的集群里：

- 节点A向节点D发送PING消息，并且消息里面包含了节点B和节点C的信息，当节点D收到这条PING消息时，它将更新自己对节点B和节点C的认识。
- 之后，节点D将向节点A返回一条PONG消息，并且消息里面包含了节点E和节点F的消息，当节点A收到这条PONG消息时，它将更新自己对节点E和节点F的认识。

整个通信过程如图17-41所示。

![image-20230924203259637](https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230924203259637.png)

### FAIL消息的实现

当集群里的主节点A将主节点B标记为已下线（FAIL）时，主节点A将向集群广播一条关于主节点B的FAIL消息，所有接收到这条FAIL消息的节点都会将主节点B标记为已下线。

在集群的节点数量比较大的情况下，单纯使用Gossip协议来传播节点的已下线信息会给节点的信息更新带来一定延迟，因为Gossip协议消息通常需要一段时间才能传播至整个集群，而发送FAIL消息可以让集群里的所有节点立即知道某个主节点已下线，从而尽快判断是否需要将集群标记为下线，又或者对下线主节点进行故障转移。

FAIL消息的正文由cluster.h/clusterMsgDataFail结构表示，这个结构只包含一个nodename属性，该属性记录了已下线节点的名字：

```
typedef struct {
    char nodename[REDIS_CLUSTER_NAMELEN];
} clusterMsgDataFail;
```

因为集群里的所有节点都有一个独一无二的名字，所以FAIL消息里面只需要保存下线节点的名字，接收到消息的节点就可以根据这个名字来判断是哪个节点下线了。

举个例子，对于包含7000、7001、7002、7003四个主节点的集群来说：

- 如果主节点7001发现主节点7000已下线，那么主节点7001将向主节点7002和主节点7003发送FAIL消息，其中FAIL消息中包含的节点名字为主节点7000的名字，以此来表示主节点7000已下线。
- 当主节点7002和主节点7003都接收到主节点7001发送的FAIL消息时，它们也会将主节点7000标记为已下线。
- 因为这时集群已经有超过一半的主节点认为主节点7000已下线，所以集群剩下的几个主节点可以判断是否需要将集群标记为下线，又或者开始对主节点7000进行故障转移。

节点发送和接收FAIL消息的整个过程。

<img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230924203314932.png" alt="image-20230924203314932" style="zoom:67%;" />

节点7001将节点7000标记为已下线

<img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230924203322722.png" alt="image-20230924203322722" style="zoom:67%;" />

节点7001向集群广播FAIL消息

<img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230924203334122.png" alt="image-20230924203334122" style="zoom:67%;" />

节点7002和节点7003也将节点7000标记为已下线

### PUBLISH消息的实现

当客户端向集群中的某个节点发送命令：

```
PUBLISH <channel> <message>
```

的时候，接收到PUBLISH命令的节点不仅会向channel频道发送消息message，它还会向集群广播一条PUBLISH消息，所有接收到这条PUBLISH消息的节点都会向channel频道发送message消息。

换句话说，向集群中的某个节点发送命令：

```
PUBLISH <channel> <message>
```

将导致集群中的所有节点都向channel频道发送message消息。

举个例子，对于包含7000、7001、7002、7003四个节点的集群来说，如果节点7000收到了客户端发送的PUBLISH命令，那么节点7000将向7001、7002、7003三个节点发送PUBLISH消息，如图17-45所示。

<img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230924203352061.png" alt="image-20230924203352061" style="zoom:67%;" />

接收到PUBLISH命令的节点7000向集群广播PUBLISH消息

PUBLISH消息的正文由cluster.h/clusterMsgDataPublish结构表示：

```
typedef struct {
  uint32_t channel_len;
  uint32_t message_len;
  // 
定义为8 
字节只是为了对齐其他消息结构
  // 
实际的长度由保存的内容决定
  unsigned char bulk_data[8];
} clusterMsgDataPublish;
```

clusterMsgDataPublish结构的bulk_data属性是一个字节数组，这个字节数组保存了客户端通过PUBLISH命令发送给节点的channel参数和message参数，而结构的channel_len和message_len则分别保存了channel参数的长度和message参数的长度：

- 其中bulk_data的0字节至channel_len-1字节保存的是channel参数。
- 而bulk_data的channel_len字节至channel_len+message_len-1字节保存的则是message参数。

举个例子，如果节点收到的PUBLISH命令为：

```
PUBLISH "news.it" "hello"
```

那么节点发送的PUBLISH消息的clusterMsgDataPublish结构将如图17-46所示：其中bulk_data数组的前七个字节保存了channel参数的值"news.it"，而bulk_data数组的后五个字节则保存了message参数的值"hello"。

![image-20230924203410607](https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230924203410607.png)

clusterMsgDataPublish结构示例

> **为什么不直接向节点广播PUBLISH命令**
>
> 实际上，要让集群的所有节点都执行相同的PUBLISH命令，最简单的方法就是向所有节点广播相同的PUBLISH命令，这也是Redis在复制PUBLISH命令时所使用的方法，不过因为这种做法并不符合Redis集群的“各个节点通过发送和接收消息来进行通信”这一规则，所以节点没有采取广播PUBLISH命令的做法。

## 8. 重点回顾

- 节点通过握手来将其他节点添加到自己所处的集群当中。
- 集群中的 `16384` 个槽可以分别指派给集群中的各个节点， 每个节点都会记录哪些槽指派给了自己， 而哪些槽又被指派给了其他节点。
- 节点在接到一个命令请求时， 会先检查这个命令请求要处理的键所在的槽是否由自己负责， 如果不是的话， 节点将向客户端返回一个 `MOVED` 错误， `MOVED` 错误携带的信息可以指引客户端转向至正在负责相关槽的节点。
- 对 Redis 集群的重新分片工作是由客户端执行的， 重新分片的关键是将属于某个槽的所有键值对从一个节点转移至另一个节点。
- 如果节点 A 正在迁移槽 `i` 至节点 B ， 那么当节点 A 没能在自己的数据库中找到命令指定的数据库键时， 节点 A 会向客户端返回一个 `ASK` 错误， 指引客户端到节点 B 继续查找指定的数据库键。
- `MOVED` 错误表示槽的负责权已经从一个节点转移到了另一个节点， 而 `ASK` 错误只是两个节点在迁移槽的过程中使用的一种临时措施。
- 集群里的从节点用于复制主节点， 并在主节点下线时， 代替主节点继续处理命令请求。
- 集群中的节点通过发送和接收消息来进行通讯， 常见的消息包括 `MEET` 、 `PING` 、 `PONG` 、 `PUBLISH` 、 `FAIL` 五种。

## 9. 集群搭建 (基于 docker)

todo...

## 10. 主节点宕机(较新版本Redis的主节点故障处理)

> redis集群中，没有redis-sentinel，但是在集群中的某个主节点出现故障之后，依旧可以自动进行故障处理。
>
> ⼿动停⽌⼀个 master 节点, 观察效果. 
>
> ⽐如上述拓扑结构中, 可以看到 redis1 redis2 redis3 是主节点, 随便挑⼀个停掉. `docker stop redis1`
>
> 连上 redis2 , 观察结果.
>
> ```C++
> ```
>
> 可以看到, 101 已经提⽰ fail, 然后 原本是 slave 的 105 成了新的 master.
>
> 如果重新启动 redis1 `docker start redis1`
>
> 再次观察结果. 可以看到 101 启动了, 仍然是 slave.
>
> 可以使⽤ cluster failover 进⾏集群恢复. 也就是把 101 重新设定成 master. (登录到 101 上执⾏)

**集群中主节点故障的处理流程**

1) **故障判定** 

集群中的所有节点, 都会周期性的使⽤⼼跳包进⾏通信. 

1. 节点 A 给 节点 B 发送 ping 包, B 就会给 A 返回⼀个 pong 包. ping 和 pong 除了 message type 属性之外, 其他部分都是⼀样的. 这⾥包含了集群的配置信息(该节点的id, 该节点从属于哪个分⽚, 是主节点还是从节点, 从属于谁, 持有哪些 slots 的位图...). 
2. 每个节点, 每秒钟, 都会给⼀些随机的节点发起 ping 包, ⽽不是全发⼀遍. 这样设定是为了避免在节点很多的时候, ⼼跳包也⾮常多(⽐如有 9 个节点, 如果全发, 就是 9 * 8 有 72 组⼼跳了, ⽽且这是按 照 N^2 这样的级别增⻓的).
3. 当节点 A 给节点 B 发起 ping 包, B 不能如期回应的时候, 此时 A 就会尝试重置和 B 的 tcp 连接, 看能否连接成功. 如果仍然连接失败, **A 就会把 B 设为 PFAIL 状态(相当于主观下线).**
4. A 判定 B 为 PFAIL 之后, 会通过 redis 内置的 Gossip 协议, **和其他节点进行沟通, 向其他节点确认 B 的状态.** (每个节点都会维护⼀个⾃⼰的 "下线列表", 由于视⻆不同, 每个节点的下线列表也不⼀定相同).
5. **此时 A 发现其他很多节点, 也认为 B 为 PFAIL, 并且数目超过总集群个数的⼀半, 那么 A 就会把 B 标记成 FAIL (相当于客观下线), 并且把这个消息同步给其他节点(其他节点收到之后, 也会把 B 标记成 FAIL).** 
6. <u>⾄此, B节点 就彻底被判定为故障节点了.</u>

> 某个或者某些节点宕机, 有的时候会引起整个集群都宕机 (称为 fail 状态). 
>
> 以下三种情况会出现集群宕机: 
>
> - 某个分⽚, 所有的主节点和从节点都挂了.（整个集群的部分数据的读写都不能执行了）
> - 某个分⽚, 主节点挂了, 但是没有从节点.（和第一个情况本质一样）
> - 超过半数的 master 节点都挂了.（短期内，因为集群有主节点故障处理功能，所以如果真的有板书master都挂了，说明确实出问题了）

2. **故障迁移**

上述例⼦中, B 故障, 并且 A 把 B FAIL 的消息告知集群中的其他节点.

- 如果 B 是从节点, 那么不需要进⾏故障迁移. 
- 如果 B 是主节点, 那么就会由 B 的从节点 (⽐如 C 和 D) 触发故障迁移了

所谓故障迁移, 就是指把从节点提拔成主节点, 继续给整个 redis 集群提供⽀持. 

具体流程如下: 

1. 从节点判定⾃⼰是否具有参选资格：如果从节点和主节点已经太久没通信(此时认为从节点的数据和主节点差异太⼤了), 时间超过阈值, 就失去竞选资格.
1. 具有资格的节点, ⽐如 C 和 D, 就会先休眠⼀定时间. 休眠时间 = 500ms 基础时间 + [0, 500ms] 随机时间 + 排名 * 1000ms. （offset 的值越⼤, 则排名越靠前(越⼩)，则休眠时间较大概率更短）
1. ⽐如 C 的休眠时间到了, C 就会给集群中其他所有节点, 进⾏拉票操作. 但是只有主节点才有投票资格.（从节点也会收到拉票请求，但是没有投票资格，只能看着）
1. 主节点就会把⾃⼰的票投给 C (每个主节点只有 1 票). 当 C 收到的票数超过主节点数⽬的⼀半, C 就会晋升成主节点. (C ⾃⼰负责执⾏ slaveof no one, 并且让 D 执⾏ slaveof C).
1. 同时, C 还会把⾃⼰成为主节点的消息, 同步给集群中其他节点. ⼤家也都会更新⾃⼰保存的集群结构信息.

> 上述选举的过程, 称为 Raft 算法, 是⼀种在分布式系统中⼴泛使⽤的算法. 在随机休眠时间的加持下, 基本上就是谁先唤醒, 谁就能竞选成功.
>
> 这里其实和哨兵那里也一样，重点和目的是为了选出一个，选谁并不是很重要。

故障判定：通过心跳包机制，先由某个节点 - A发现自己与节点B之间的心跳包异常，则A先将B判定为PFAIL - 主观下线，然后A与其他节点沟通，确认其他节点与B节点的状态。A发现超过集群节点总数一半的节点都认为B为PFAIL，则A将B判定为FAIL - 客观下线。并且将此消息同步给其他节点，其他节点收到此消息，也会将B设定为FAIL。

故障迁移：因为这里并没有哨兵，所以不需要先选举出leader哨兵。并且也不是由第一个发现B为FAIL状态的A节点来进行故障迁移。
而是：先看B是它所属分片的主还是从，若从，没事；若主，则进行故障迁移。B的若干从节点看自己是否由参选资格（参选为新的主节点的资格，评判标准为从节点与主节点距离上次通信的时长，超过某阈值（太长）则没有资格），若有资格，则休眠若干时间（具有随机性，但是offset越高休眠时间大概率越短），最先结束休眠的从节点给集群中其他的所有结点发送一个拉票请求，但是只有主节点有投票资格，且只有一票，所以其实就是哪个从节点先结束休眠，则它就有很大概率获取其他主节点的票。若这个最先结束休眠的从节点获取的票超过主节点数目的一半，则它将晋升为新主节点。自己执行slaveof no one，且其他从节点执行slaveof 新主节点。最后一步：这个新主节点还会把自己晋升为新的主节点的消息同步给集群中其他结点。其他结点会修改自己保存的集群的结构信息。

哨兵模式中：是先从sentinel中选出一个leader，leader进行故障迁移。而这里是若干从节点中直接自己拉票选出一个新的主节点，自

己进行故障迁移。

## 11. 集群扩容

扩容是⼀个在开发中⽐较常遇到的场景.

随着业务的发展, 现有集群很可能⽆法容纳⽇益增⻓的数据. 此时给集群中加⼊更多新的机器(分片), 就可以使存储的空间更⼤了.

>  **所谓分布式的本质, 就是使用更多的机器, 引入更多的硬件资源.**

1. 第⼀步: 把新的主节点加⼊到集群
2. 第⼆步: 重新分配 slots
3. 第三步: 给新的主节点添加从节点

（这里都是些操作，且基于docker，所以暂时不弄了）
