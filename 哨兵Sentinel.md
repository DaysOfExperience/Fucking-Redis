# 哨兵的概念

Redis主从模式下，主节点一旦挂了，则写操作无法继续执行，这是肯定不能容忍的，所以方案就是选择一个从节点让它成为主节点，也就是主从切换。但是这个操作，又需要人工进行，诸多不利。所以Redis 从 2.8 开始提供了 Redis Sentinel（哨兵）哨兵模式就是来解决这个问题的。 <u>也就是通过自动化的方式，解决主节点挂了的问题, 自动化进行主从切换</u>

几个名词解释

<img src="C:\Users\yangzilong\AppData\Roaming\Typora\typora-user-images\image-20230905202959955.png" alt="image-20230905202959955" style="zoom:67%;" />

Redis Sentinel 概念：**Redis Sentinel 是 Redis 的高可用实现方案**，在实际的⽣产环境中，对**提高整个系统的高可用**是⾮常有帮助的。

# 主从复制的问题-引出哨兵

Redis主从复制（主从模式）的优点

1. 从节点作为主节点数据的备份

2. 提高读操作的可用性：从节点作为主节点数据的备份，主节点及时挂了，读操作依旧可以在从节点上完成。

3. 提高读并发：可以让主节点只承担写请求，读写分离。

但是主从复制模式并不是万能的，它同样遗留了以下⼏个问题：

1. **主节点发生故障时，进行主从切换的过程是复杂的，需要完全的人工参与，无法自动化完成, 导致故障恢复时间无法保障。**
2. **主节点可以将读压力分散出去，但写压力/存储压力是无法被分担的，还是受到单机的限制。**

其中第⼀个问题是⾼可⽤问题，即 Redis 哨兵主要解决的问题。第⼆个问题是属于存储分布式的问题，留给 Redis 集群去解决。

**此处我们集中讨论第⼀个问题。其实也就是在主节点故障之后，如何利用sentinel哨兵机制自动化地进行主从切换。**

# Sentinel介绍 & Sentinel的初始化

启动一个 Sentinel 可以使用命令：

```bash
$ redis-sentinel /path/to/your/sentinel.conf
$ redis-server /path/to/your/sentinel.conf --sentinel
```

## Sentinel进程启动步骤

### 1. 初始化服务器

 **Sentinel 本质上只是一个运行在特殊模式下的 Redis 服务器**， 所以启动 Sentinel 的**第一步， 就是初始化一个普通的 Redis 服务器**

Sentinel 的初始化过程和普通 Redis 服务器的<u>初始化过程并不完全相同</u>: 如: 普通服务器在初始化时会通过载入 RDB 文件或者 AOF 文件来还原数据库状态， 但是因为 Sentinel 并不使用数据库， 所以初始化 Sentinel 时就不会载入 RDB 文件或者 AOF 文件。

Sentinel 模式下的 Redis 服务器主要功能的使用情况

| 功能                                                         | 使用情况                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| 数据库和键值对方面的命令， 比如 SET 、 DEL 、 FLUSHDB 。     | 不使用。                                                     |
| 事务命令， 比如 MULTI 和 WATCH                               | 不使用。                                                     |
| 脚本命令，比如 EVAL 。                                       | 不使用。                                                     |
| RDB 持久化命令， 比如 SAVE 和 BGSAVE                         | 不使用。                                                     |
| AOF 持久化命令， 比如 BGREWRITEAOF                           | 不使用。                                                     |
| 复制命令，比如 SLAVEOF                                       | Sentinel 内部可以使用，但客户端不可以使用。                  |
| 发布与订阅命令， 比如 PUBLISH 和 SUBSCRIBE 。                | SUBSCRIBE 、 PSUBSCRIBE 、 UNSUBSCRIBE PUNSUBSCRIBE 四个命令在 Sentinel 内部和客户端都可以使用， 但 PUBLISH 命令只能在 Sentinel 内部使用。 |
| 文件事件处理器（负责发送命令请求、处理命令回复）。(这里其实就是多路复用技术) | Sentinel 内部使用， 但关联的文件事件处理器(其实就是文件的读/写时间就绪之后的回调函数)和普通 Redis 服务器不同。 |
| 时间事件处理器（负责执行 `serverCron` 函数）。               | Sentinel 内部使用， 时间事件的处理器仍然是 `serverCron` 函数， `serverCron` 函数会调用 `sentinel.c/sentinelTimer` 函数， 后者包含了 Sentinel 要执行的所有操作。 |

有关文件事件/时间事件可以看《Redis设计与实现》第12章

### 2. 使用Sentinel专用代码

启动 Sentinel 的**第二个步骤就是将一部分普通 Redis 服务器使用的代码替换成 Sentinel 专用代码。**

举例, 如:

- 普通 Redis 服务器使用 `redis.h/REDIS_SERVERPORT` 常量的值作为服务器端口, 而 Sentinel 则使用 `sentinel.c/REDIS_SENTINEL_PORT` 常量的值作为服务器端口

  ```C++
  #define REDIS_SERVERPORT 6379
  #define REDIS_SENTINEL_PORT 26379
  ```

- 普通 Redis 服务器使用 `redis.c/redisCommandTable` 作为服务器的命令表
  而 Sentinel 则使用 `sentinel.c/sentinelcmds` 作为服务器的命令表， 并且其中的 INFO 命令会使用 Sentinel 模式下的专用实现 `sentinel.c/sentinelInfoCommand` 函数， 而不是普通 Redis 服务器使用的实现 `redis.c/infoCommand` 函数

  ```c
  struct redisCommand sentinelcmds[] = {
      {"ping",pingCommand,1,"",0,NULL,0,0,0,0,0},
      {"sentinel",sentinelCommand,-2,"",0,NULL,0,0,0,0,0},
      {"subscribe",subscribeCommand,-2,"",0,NULL,0,0,0,0,0},
      {"unsubscribe",unsubscribeCommand,-1,"",0,NULL,0,0,0,0,0},
      {"psubscribe",psubscribeCommand,-2,"",0,NULL,0,0,0,0,0},
      {"punsubscribe",punsubscribeCommand,-1,"",0,NULL,0,0,0,0,0},
      {"info",sentinelInfoCommand,-1,"",0,NULL,0,0,0,0,0}
  };
  ```

  `sentinelcmds` 命令表也解释了为什么在 Sentinel 模式下， Redis 服务器不能执行诸如 SET 、 DBSIZE 、 EVAL 等等这些命令 —— 因为服务器根本没有在命令表中载入这些命令： PING 、 SENTINEL 、 INFO 、 SUBSCRIBE 、 UNSUBSCRIBE 、 PSUBSCRIBE 和 PUNSUBSCRIBE 这七个命令就是客户端可以对 Sentinel 执行的全部命令了。(不管是redis服务器还是sentinel模式的redis服务器, 在收到一个命令之后, 都需要去命令表中查找)

### 3. 初始化sentinel状态

在应用了 Sentinel 的专用代码之后， **接下来， 服务器会初始化一个 `sentinel.c/sentinelState` 结构（后面简称“Sentinel 状态”）**， <u>这个结构保存了服务器中所有和 Sentinel 功能有关的状态</u> （服务器的一般状态仍然由 `redis.h/redisServer` 结构保存）

```C
struct sentinelState {

    // 当前纪元，用于实现故障转移
    uint64_t current_epoch;

    // 保存了所有被这个 sentinel 监视的主服务器
    // 字典的键是主服务器的名字
    // 字典的值则是一个指向 sentinelRedisInstance 结构的指针
    dict *masters;      // 关键

    // 是否进入了 TILT 模式？
    int tilt;

    // 目前正在执行的脚本的数量
    int running_scripts;

    // 进入 TILT 模式的时间
    mstime_t tilt_start_time;

    // 最后一次执行时间处理器的时间
    mstime_t previous_time;

    // 一个 FIFO 队列，包含了所有需要执行的用户脚本
    list *scripts_queue;

} sentinel;

```

> 此处的`dict *masters`是关键, 即每一个Sentinel模式的Redis服务器都会有一个sentinelState结构体的实例化对象, 保存与sentinel功能相关的状态, 里面的dict *masters为一个指向字典的指针, **这个字典就存储了当前sentinel服务器所监视的所有主redis服务器(用sentinelRedisInstance描述)**, 事实上sentinel服务器还会监视所有的从服务器, 还有其他sentinel服务器, 不过这些都是在`dict *masters`内部存储的.

### 4. 初始化Sentinel状态的masters服务器

**Sentinel 状态中的 `masters` 字典记录了所有被 Sentinel 监视的<u>主服务器</u>的相关信息**， 其中：

- 字典的键是被监视主服务器的名字。
- 而字典的值则是被监视主服务器对应的 `sentinel.c/sentinelRedisInstance` 结构。

**每个 `sentinelRedisInstance` 结构体实例化对象（后面简称“实例结构”）代表一个被 Sentinel 监视的 Redis 服务器实例**（instance）， 这个实例可以是主服务器、从服务器、或者另外一个 Sentinel 。

> 但是sentinelState结构中的masters字典的值都是代表着Redis主服务器的实例, 其实就是用这个sentinelRedisInstance去描述这个Redis主服务器实例, 再用dict字典数据结构组织起来, 也就是先描述, 再组织, 这样sentinel服务器就能把所有监视的主服务器管理起来

下面展示了sentinelRedisInstance实例结构在<u>表示主服务器时</u>使用的其中一部分属性， 本章接下来将逐步对实例结构中的各个属性进行介绍：

```C++
typedef struct sentinelRedisInstance {

    // 标识值，记录了实例的类型，以及该实例的当前状态
    int flags;   // 这个sentinelRedisInstance结构体所代表的是什么类型的Redis服务器示例呢? 主/从/sentinel服务器

    // 实例的名字
    // 主服务器的名字由用户在配置文件中设置
    // 从服务器以及 Sentinel 的名字由 Sentinel 自动设置
    // 格式为 ip:port ，例如 "127.0.0.1:26379"
    char *name;

    // 实例的运行 ID
    char *runid;

    // 配置纪元，用于实现故障转移
    uint64_t config_epoch;

    // 实例的地址
    // 指向 sentinel.c/sentinelAddr 结构的指针
    sentinelAddr *addr;   // 结构体: char* ip; int port; 存储这个实例的ip+port

    // SENTINEL down-after-milliseconds 选项设定的值
    // 实例无响应多少毫秒之后才会被判断为主观下线（subjectively down）
    mstime_t down_after_period;

    // SENTINEL monitor <master-name> <IP> <port> <quorum> 选项中的 quorum 参数
    // 判断这个实例为客观下线（objectively down）所需的支持投票数量
    int quorum;

    // SENTINEL parallel-syncs <master-name> <number> 选项的值
    // 在执行故障转移操作时，可以同时对新的主服务器进行同步的从服务器数量
    int parallel_syncs;

    // SENTINEL failover-timeout <master-name> <ms> 选项的值
    // 刷新故障迁移状态的最大时限
    mstime_t failover_timeout;

    // ...
    // 此处并不全, 更多关键信息看下方

} sentinelRedisInstance;
```

**对 Sentinel 状态结构体对象的初始化将引发对 `masters` 字典的初始化， 而 `masters` 字典的初始化是根据被载入的 Sentinel 配置文件来进行的。**

初始化masters字典就是为了初始化主服务器的sentinelRedisInstance结构体对象

> 举例:
>
> ```C
> #####################
> # master1 configure #
> #####################
> 
> sentinel monitor master1 127.0.0.1 6379 2
> 
> sentinel down-after-milliseconds master1 30000
> 
> sentinel parallel-syncs master1 1
> 
> sentinel failover-timeout master1 900000
> 
> #####################
> # master2 configure #
> #####################
> 
> sentinel monitor master2 127.0.0.1 12345 5
> 
> sentinel down-after-milliseconds master2 50000
> 
> sentinel parallel-syncs master2 5
> 
> sentinel failover-timeout master2 450000
>     
> ```
>
> 那么 Sentinel 将为主服务器 `master1` 创建如图 IMAGE_MASTER1 所示的sentinelRedisInstance类型的实例结构， 并为主服务器 `master2` 创建如图 IMAGE_MASTER2 所示的sentinelRedisInstance类型的实例结构， 而这两个实例结构又会被保存到 Sentinel 状态结构体的 `masters` 字典中， 如图 IMAGE_SENTINEL_STATE 所示。
>
> <img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230921134645669.png" alt="image-20230921134645669" style="zoom:67%;" />
>
> <img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230921134658784.png" alt="image-20230921134658784" style="zoom:67%;" />
>
> <img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230921134709438.png" alt="image-20230921134709438" style="zoom:67%;" />

### 5.创建连向主服务器的网络连接&订阅链接

**Sentinel服务器与Sentinel状态中的matsers字典中记录的若干个监视的Redis主服务器创建两个异步网络连接**

- 命令连接, 用于sentinel向主服务器发送命令, 并接收命令回复(此时Sentinel服务器将作为主服务器的客户端)
- 订阅连接, 用于订阅主服务器的 \__sentinel__:hello频道

> 为什么有两个连接？
>
> 在 Redis 目前的发布与订阅功能中， 被发送的信息都不会保存在 Redis 服务器里面， 如果在信息发送时， 想要接收信息的客户端不在线或者断线， 那么这个客户端就会丢失这条信息。
>
> 因此， 为了不丢失 `__sentinel__:hello` 频道的任何信息， Sentinel 必须专门用一个订阅连接来接收该频道的信息。
>
> 而另一方面， 除了订阅频道之外， Sentinel 还又必须向主服务器发送命令， 以此来与主服务器进行通讯， 所以 Sentinel 还必须向主服务器创建命令连接。
>
> 并且因为 Sentinel 需要与多个实例创建多个网络连接， 所以 Sentinel 使用的是异步连接。

图 IMAGE_SENTINEL_CONNECT_SERVER 展示了一个 Sentinel 向被它监视的两个主服务器 `master1` 和 `master2` 创建命令连接和订阅连接的例子。

<img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230921135038821.png" alt="image-20230921135038821" style="zoom:67%;" />

接下来的一节将介绍 Sentinel 是如何通过命令连接和订阅连接来与被监视主服务器进行通讯的。

# Sentinel与主服务器建立的命令连接和订阅连接的作用

## 获取主服务器信息

**Sentinel默认会以<u>每十秒一次</u>的频率，<u>通过命令连接向被监视的主服务器发送INFO命令</u>，并通过分析INFO命令的回复来获取主服务器的当前信息。**

举例:

<img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230921144321439.png" alt="image-20230921144321439" style="zoom:67%;" />

如图所示，主服务器master有三个从服务器slave0、slave1和slave2，并且一个Sentinel正在连接主服务器，那么Sentinel将持续地向主服务器发送INFO命令，并获得类似于以下内容的回复：

```C
# Server
...
run_id:7611c59dc3a29aa6fa0609f841bb6a1019008a9c
...
# Replication
role:master
...
slave0:ip=127.0.0.1,port=11111,state=online,offset=43,lag=0
slave1:ip=127.0.0.1,port=22222,state=online,offset=43,lag=0
slave2:ip=127.0.0.1,port=33333,state=online,offset=43,lag=0
...
# Other sections
...

```

通过分析主服务器返回的INFO命令回复，Sentinel可以获取以下两方面的信息：

- 一方面是关于主服务器本身的信息，包括run_id域记录的服务器运行ID，以及role域记录的服务器角色；
- 另一方面是关于主服务器属下所有从服务器的信息，每个从服务器都由一个"slave"字符串开头的行记录，每行的ip=域记录了从服务器的IP地址，而port=域则记录了从服务器的端口号。**根据这些IP地址和端口号，Sentinel无须用户提供从服务器的地址信息，就可以自动发现从服务器。**
  也就是sentinel配置文件中不需要用户添加从服务器的相关信息, 只需要主服务器的信息, 因为在sentinel通过INFO命令与主服务器交互时, 可以通过INFO的返回值获取到该主服务器下的从服务器的相关信息.

根据run_id域和role域记录的信息，Sentinel将对主服务器的实例结构进行更新，例如，主服务器重启之后，它的运行ID就会和sentinelRedisInstance实例结构之前保存的运行ID不同，Sentinel检测到这一情况之后，就会对实例结构的运行ID进行更新。

至于主服务器返回的从服务器信息，**则会被用于更新主服务器实例结构的slaves字典，这个字典记录了主服务器属下从服务器的名单：**(也就是每个主服务器的sentinelRedisInstance都/也有一个dict字典数据成员, 这个字典存储的是该主服务器属下的从服务器的实例结构~)

- 字典的键是由Sentinel自动设置的从服务器名字，格式为ip:port：如对于IP地址为127.0.0.1，端口号为11111的从服务器来说，Sentinel为它设置的名字就是127.0.0.1:11111。
- 至于**字典的值则是从服务器对应的`struct sentinelRedisInstance`实例结构**：比如说，如果键是127.0.0.1:11111，那么这个键的值就是IP地址为127.0.0.1，端口号为11111的从服务器的实例结构。

Sentinel在分析INFO命令中包含的从服务器信息时，会检查从服务器对应的实例结构是否已经存在于主服务器的实例结构的slaves字典, 若已经存在，则Sentinel对从服务器的实例结构进行更新。若不存在，则说明这个从服务器是新发现的从服务器(这个主服务器新增的从属服务器)，Sentinel会在主实例结构的slaves字典中为这个从服务器新创建一个实例结构。

对于我们之前列举的主服务器master和三个从服务器slave0、slave1和slave2的例子来说，Sentinel将分别为三个从服务器创建它们各自的实例结构，并将这些结构保存到主服务器实例结构的slaves字典里面，如图所示。

![image-20230921145130699](https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230921145130699.png)

注意对比图中主服务器实例结构和从服务器实例结构之间的区别：

- 主服务器实例结构的flags属性的值为SRI_MASTER，而从服务器实例结构的flags属性的值为SRI_SLAVE。
- 主服务器实例结构的name属性的值是用户使用Sentinel配置文件设置的，而从服务器实例结构的name属性的值则是Sentinel根据从服务器的IP地址和端口号自动设置的。

> 总结: Sentinel默认以每十秒一次的频率向监视的所有主服务器发送INFO命令, INFO命令的回复中包括主服务器的信息数据, 主服务器属下的所有从服务器的信息数据.
>
> 对于主服务器的数据, 可以让Sentinel更新, 对于从服务器的数据, 可以让Sentinel用来更新从服务器的实例数据/新增从服务器的实例数据, 这里的更新/新增其实就是更新/新增sentinelRedisIInstance结构体对象
>
> 主服务器属下从服务器的sentinelRedisIInstance实例结构就存储在主服务器的sentinelRedisIInstance实例结构中, 也是用一个dict字典结构组织起来的
>
> 这样有个好处是, sentinel配置文件中不需要用户手动添加从属服务器的相关信息, sentinel可以通过与主服务器建立好的命令连接自动获取, 及时进行新增/更新.

## 获取从服务器信息

当Sentinel发现主服务器有新的从服务器出现时，**Sentinel除了会为这个新的从服务器创建相应的实例结构之外，Sentinel还会创建连接到从服务器的命令连接和订阅连接。**

> 举例:
>
> 对于下图所示的主从服务器关系来说，Sentinel将对slave0、slave1和slave2三个从服务器分别创建命令连接和订阅连接
>
> <img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230921150036144.png" alt="image-20230921150036144" style="zoom:67%;" />

**在创建命令连接之后，Sentinel在默认情况下，会以每十秒一次的频率通过命令连接向从服务器发送INFO命令**，并获得类似于以下内容的回复

```c
# Server
...
run_id:32be0699dd27b410f7c90dada3a6fab17f97899f
...
# Replication
role:slave
master_host:127.0.0.1
master_port:6379
master_link_status:up
slave_repl_offset:11887
slave_priority:100
# Other sections
...

```

根据INFO命令的回复，Sentinel会提取出以下信息：

- 从服务器的运行ID run_id。
- 从服务器的角色role。
- 主服务器的IP地址master_host，以及主服务器的端口号master_port。
- 主从服务器的连接状态master_link_status。
- 从服务器的优先级slave_priority。
- 从服务器的复制偏移量slave_repl_offset。

**根据这些信息，Sentinel会对从服务器的实例结构进行更新**

下图展示了Sentinel根据上面的INFO命令回复对从服务器的实例结构进行更新之后，实例结构的样子。

<img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230921150250946.png" alt="image-20230921150250946" style="zoom: 50%;" />

> 虽然周期性地向主服务器发送INFO可以通过INFO的回复信息获取到主服务器属下的从服务器的相关信息, 但是那里的信息是有限的, 不全的, 所以sentinel还需要周期性向从服务器发送INFO, 以获取更全的信息数据, 从而更新从服务器的sentinelRedisInstance实例结构

> 下面也会用到命令连接&订阅连接

## 向主服务器和从服务器发送信息

在默认情况下，**Sentinel会以<u>每两秒一次的频率，通过命令连接</u>向所有被监视的主服务器和从服务器发送以下格式的PUBLISH命令**：

```
PUBLISH __sentinel__:hello "<s_ip>,<s_port>,<s_runid>,<s_epoch>,<m_name>,<m_ip>,<m_port>,<m_epoch>"Copy to clipboardErrorCopied
```

> 这条命令向服务器的**sentinel**:hello频道发送了一条信息，信息的内容由多个参数组成：
>
> - 其中以s_开头的参数记录的是**Sentinel本身的信息**，各个参数的意义如表16-2所示。
> - 而m_开头的参数记录的则是**主服务器的信息**，各个参数的意义如表16-3所示。如果Sentinel正在监视的是主服务器，那么这些参数记录的就是主服务器的信息；如果Sentinel正在监视的是从服务器，那么这些参数记录的就是从服务器正在复制的主服务器的信息。
>
> 表16-2　信息中和Sentinel有关的参数
>
> ![image-20230921150740030](https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230921150740030.png)
>
> 表16-3　信息中和主服务器有关的参数
>
> ![image-20230921150748354](https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230921150748354.png)
>
> 以下是一条Sentinel通过PUBLISH命令向主服务器发送的信息示例：
>
> ```
> "127.0.0.1,26379,e955b4c85598ef5b5f055bc7ebfd5e828dbed4fa,0,mymaster,127.0.0.1,6379,0"Copy to clipboardErrorCopied
> ```
>
> 这个示例包含了以下信息：
>
> - Sentinel的IP地址为127.0.0.1端口号为26379，运行ID为e955b4c85598ef5b5f055bc7ebfd5e828dbed4fa，当前的配置纪元为0。
> - 主服务器的名字为mymaster，IP地址为127.0.0.1，端口号为6379，当前的配置纪元为0。

## 接收来自主服务器和从服务器的频道信息

当Sentinel与一个主服务器或者从服务器建立起订阅连接之后，**Sentinel就会通过订阅连接，向服务器发送以下命令：**

```
SUBSCRIBE __sentinel__:helloCopy to clipboardErrorCopied
```

> 目前对于publish 和 subscribe命令不了解, 目前看来, publish是向指定频道发送信息, subscribe是订阅频道, 从接收信息.
>
> PUBLISH & SUBSCRIBE介绍:
>
> ...

Sentinel对**sentinel**:hello频道的订阅会一直持续到Sentinel与服务器的连接断开为止。

这也就是说，对于每个与Sentinel连接的服务器，**Sentinel既通过命令连接向服务器的sentinel:hello频道发送信息，又通过订阅连接从服务器的sentinel:hello频道接收信息**

<img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230921151806989.png" alt="image-20230921151806989" style="zoom:67%;" />

---

**对于监视同一个服务器的多个Sentinel来说，一个Sentinel发送的信息会被其他Sentinel接收到，这些信息会被用于更新其他Sentinel对发送信息Sentinel的认知，也会被用于更新其他Sentinel对被监视服务器的认知。**(利用的是频道功能)

> 举个例子，假设现在有sentinel1、sentinel2、sentinel3三个Sentinel在监视同一个服务器，那么当sentinel1向服务器的**sentinel**:hello频道发送一条信息时，所有订阅了**sentinel**:hello频道的Sentinel（包括sentinel1自己在内）都会收到这条信息，如图
>
> <img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230921151822792.png" alt="image-20230921151822792" style="zoom:67%;" />

当一个Sentinel从**sentinel**:hello频道收到一条信息时，Sentinel会对这条信息进行分析，提取出信息中的Sentinel IP地址、Sentinel端口号、Sentinel运行ID等八个参数，并进行以下检查：

- 如果信息中记录的Sentinel运行ID和接收信息的Sentinel的运行ID相同，那么说明这条信息是Sentinel自己发送的，Sentinel将丢弃这条信息，不做进一步处理。
- 相反地，如果信息中记录的Sentinel运行ID和接收信息的Sentinel的运行ID不相同，那么说明这条信息是监视同一个服务器的其他Sentinel发来的，**接收信息的Sentinel将根据信息中的各个参数，对相应主服务器的实例结构进行更新。**
  信息中也包括发送sentinel的信息, 所以应该也会对发送sentinel的实例结构进行更新, 但是这方面其实还没讲到, 看后面.

> 上方两点的总结: 利用sentinel和主服务器建立好的订阅连接, sentinel会通过PUBLISH命令向该sentinel监视的所有主从服务器的频道中发送信息, 这个信息主要是主从结构中的主服务器的信息以及自身sentinel的信息
>
> 此时所有监视这个主从结构中的主/从服务器的sentinel都会收到这个频道中的信息, 说过了, 主要是主服务器的信息&发送sentinel的信息, 包括发送这个信息的sentinel也会收到, 经过简单的判断, 其它的sentinel就可以利用收到的主服务器的信息去更新这个sentinel服务器中存储的主服务器的实例结构
>
> 也就是说, 除了默认每10s一次的INFO命令可以获取主服务器的信息, 来更新主服务器的实例结构, 这里利用频道连接的PUBLISH&SUBSCRIBE也可以获取其他sentinel发来的, 共同监视的同一个主服务器的信息, 从而更新主服务器的实例结构~
>
> PUBLISH是周期性的, 每2s一次的, 而SUBSCRIBE不是周期性的, 是一次命令, 持续订阅频道, 接收频道中的数据的

### 更新sentinels字典

Sentinel为主服务器创建的实例结构中的sentinels字典保存了除Sentinel本身之外，所有同样监视这个主服务器的其他Sentinel的资料(这里的资料其实就是用sentinelRedisInstance去描述其他sentinel服务器, 并记录其信息)：

- sentinels字典的键是其中一个Sentinel的名字，格式为ip:port，比如对于IP地址为127.0.0.1，端口号为26379的Sentinel来说，这个Sentinel在sentinels字典中的键就是"127.0.0.1:26379"。(和slaves字典一样)
- sentinels字典的值则是键所对应Sentinel的实例结构(sentinelRedisInstance结构体实例化对象)，比如对于键"127.0.0.1:26379"来说，这个键在sentinels字典中的值就是IP为127.0.0.1，端口号为26379的Sentinel的实例结构。

当一个Sentinel接收到其他Sentinel发来的信息时（我们称呼发送信息的Sentinel为源Sentinel，接收信息的Sentinel为目标Sentinel），目标Sentinel会从信息中分析并提取出以下两方面参数：

- 与Sentinel有关的参数：源Sentinel的IP地址、端口号、运行ID和配置纪元。
- 与主服务器有关的参数：源Sentinel正在监视的主服务器的名字、IP地址、端口号和配置纪元。(上方已经说了)

根据信息中提取出的主服务器参数，目标Sentinel会在自己的Sentinel状态的masters字典中查找相应的主服务器实例结构，然后根据提取出的Sentinel参数，检查主服务器实例结构的sentinels字典中，源Sentinel的实例结构是否存在：

- 如果源Sentinel的实例结构已经存在，那么对源Sentinel的实例结构进行更新。
- **如果源Sentinel的实例结构不存在，那么说明源Sentinel是刚刚开始监视主服务器的新Sentinel，目标Sentinel会为源Sentinel创建一个新的实例结构，并将这个结构添加到sentinels字典里面。**
  <u>也就是说, 每一个sentinel服务器的sentinel状态中的masters字典中的主服务器的实例结构中, 应当存储着除了当前这个sentinel以外的, 所有的同样监视这个主服务器的sentinel的实例结构, 其实也是用dict存储的.</u>

> 举个例子，假设分别有127.0.0.1:26379、127.0.0.1:26380、127.0.0.1:26381三个Sentinel正在监视主服务器127.0.0.1:6379，那么当127.0.0.1:26379这个Sentinel接收到以下信息时：
>
> ```
> 1) "message"
> 2) "__sentinel__:hello"
> 3) "127.0.0.1,26379,e955b4c85598ef5b5f055bc7ebfd5e828dbed4fa,0,mymaster,127.0.0.1,6379,0"
> 1) "message"
> 2) "__sentinel__:hello"
> 3) "127.0.0.1,26381,6241bf5cf9bfc8ecd15d6eb6cc3185edfbb24903,0,mymaster,127.0.0.1,6379,0"
> 1) "message"
> 2) "__sentinel__:hello"
> 3) "127.0.0.1,26380,a9b22fb79ae8fad28e4ea77d20398f77f6b89377,0,mymaster,127.0.0.1,6379,0"Copy to
> ```
>
> Sentinel将执行以下动作：
>
> - 第一条信息的发送者为127.0.0.1:26379自己，这条信息会被忽略。
> - 第二条信息的发送者为127.0.0.1:26381，Sentinel会根据这条信息中提取出的内容，对sentinels字典中127.0.0.1:26381对应的实例结构进行更新。
> - 第三条信息的发送者为127.0.0.1:26380，Sentinel会根据这条信息中提取出的内容，对sentinels字典中127.0.0.1:26380所对应的实例结构进行更新。
>
> 下图展示了Sentinel 127.0.0.1:26379为主服务器127.0.0.1:6379创建的实例结构，以及结构中的sentinels字典。
>
> <img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230921173700687.png" alt="image-20230921173700687" style="zoom:50%;" />
>
> 和127.0.0.1:26379一样，其他两个Sentinel也会创建类似于上图所示的sentinels字典，区别在于字典中保存的Sentinel信息不同：
>
> - 127.0.0.1:26380创建的sentinels字典会保存127.0.0.1:26379和127.0.0.1:26381两个Sentinel的信息。
> - 而127.0.0.1:26381创建的sentinels字典则会保存127.0.0.1:26379和127.0.0.1:26380两个Sentinel的信息。

**<u>因为一个Sentinel可以通过分析接收到的频道信息来获知其他Sentinel的存在，并通过发送频道信息来让其他Sentinel知道自己的存在，所以用户在使用Sentinel的时候并不需要提供各个Sentinel的地址信息，监视同一个主服务器的多个Sentinel可以自动发现对方。</u>**

> 到此为止, 每个sentinel状态中的masters中会存储若干个此sentinel监视的主服务器的实例结构(注: 实例结构其实就是sentinelRedisInstance结构对象), 而每个主服务器的实例结构中会有slaves字典, 存储该主服务器属下的所有从服务器的实例结构, 还有一个sentinels字典, 存储除了本sentinel以外的所有监视此主服务器的sentinel的实例结构
>
> masters字典需要用sentinel的配置文件初始化, 也就是需要用户配置. 而slaves字典不需要配置文件, 因为sentinel服务器通过与主服务器建立好的命令连接&INFO命令即可获取属下从服务器的信息, 从而维护slaves字典. 同样sentinels字典也不需要配置文件, 不需要用户配置, 因为通过命令连接&订阅连接&PUBLISH&SUBSCRIBE, 即可获取到其他监视此主服务器的sentinel的信息, 从而维护sentinels字典

### 创建连向其他sentinel的命令连接

当Sentinel通过频道信息发现一个新的Sentinel时，它不仅会为新Sentinel在sentinels字典中创建相应的实例结构，**还会创建一个连向新Sentinel的命令连接，而新Sentinel也同样会创建连向这个Sentinel的命令连接，最终监视同一主服务器的多个Sentinel将形成相互连接的网络**：Sentinel A有连向Sentinel B的命令连接，而Sentinel B也有连向Sentinel A的命令连接。

<img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230921174625008.png" alt="image-20230921174625008" style="zoom: 67%;" />

上图展示了三个监视同一主服务器的Sentinel之间是如何互相连接的。

为什么各个监视同一个主服务器的sentinel之间需要建立命令连接?

**使用命令连接相连的各个Sentinel可以通过向其他Sentinel发送命令请求来进行信息交换，本章接下来将对Sentinel实现主观下线检测和客观下线检测的原理进行介绍，这两种检测都会使用Sentinel之间的命令连接来进行通信。**

> Sentinel之间不会创建订阅连接
>
> Sentinel在连接主服务器或者从服务器时，会同时创建命令连接和订阅连接，但是在连接其他Sentinel时，却只会创建命令连接，而不创建订阅连接。
>
> 这是因为Sentinel需要通过接收主服务器或者从服务器发来的频道信息来发现未知的新Sentinel，所以才需要建立订阅连接，而相互已知的Sentinel只要使用命令连接来进行通信就足够了。(也就揭示了sentinel和主从服务器之间的订阅连接到作用和目的)

总结: 哨兵机制的核心任务之一就是监视, 先监视, 若出问题了(比如主节点故障), 再自动进行故障转移, 而每个哨兵节点要监视主服务器, 从服务器, 还会监视其他sentinel服务器, 所以, Sentinel进程启动步骤的最后一步就是创建连向主服务器的网络连接&订阅链接(主服务器的masters字典的初始化是通过配置文件完成的), 而这一部分其实讲的就是如何利用与主服务器建立好的命令连接&订阅连接

1. 通过命令连接, 与主服务器周期性的INFO命令, 获取主服务器信息, 自动获取从服务器信息(不需要用户的配置文件), 并且建立与从服务器的命令连接&订阅连接
2. 通过与从服务器的命令连接和周期性的INFO命令, 获取从服务器信息
3. 通过与主服务器的命令连接&订阅连接, 自动获取同样监视此主服务器的其他sentinel的信息, 并且建立与其他sentinel的命令连接

而具体每个sentinel之间的命令连接的作用, 下面就会介绍

# 哨兵机制自动恢复主节点故障的流程

## 检测主观下线-PING命令

**在默认情况下，Sentinel会以每秒一次的频率向所有与它创建了命令连接的实例（包括主服务器、从服务器、其他Sentinel在内）发送PING命令，并通过实例返回的PING命令回复来判断实例是否在线。**

> <img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230921182658042.png" alt="image-20230921182658042" style="zoom:67%;" />
>
> - Sentinel1将向Sentinel2、主服务器master、从服务器slave1和slave2发送PING命令。
>
> - Sentinel2将向Sentinel1、主服务器master、从服务器slave1和slave2发送PING命令。

实例对PING命令的回复可以分为以下两种情况：

- 有效回复：实例返回+PONG、-LOADING、-MASTERDOWN三种回复的其中一种。

- 无效回复：实例返回除+PONG、-LOADING、-MASTERDOWN三种回复之外的其他回复，或者在指定时限内没有返回任何回复。

Sentinel配置文件中的down-after-milliseconds选项指定了Sentinel判断实例进入主观下线所需的时间长度：如果一个实例在down-after-milliseconds毫秒内，连续向Sentinel返回无效回复，那么Sentinel会修改这个实例所对应的实例结构，在结构的flags属性中打开**SRI_S_DOWN**标识，以此来表示这个实例已经进入主观下线状态。(所以, sentinelRedisInstance结构的flags并非只标识实例的角色, 还标识状态)

> 如果配置文件指定Sentinel1的down-after-milliseconds选项的值为50000毫秒，那么当主服务器master连续50000毫秒都向Sentinel1返回无效回复时，Sentinel1就会将master标记为主观下线，并在master所对应的实例结构的flags属性中打开SRI_S_DOWN标识
>
> <img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230921183133630.png" alt="image-20230921183133630" style="zoom:67%;" />

> **主观下线时长选项的作用范围**
>
> 用户设置的down-after-milliseconds选项的值，不仅会被Sentinel用来判断主服务器的主观下线状态，还会被用于判断主服务器属下的所有从服务器，以及所有同样监视这个主服务器的其他Sentinel的主观下线状态。

> **多个Sentinel设置的主观下线时长可能不同**
>
> down-after-milliseconds选项另一个需要注意的地方是，对于监视同一个主服务器的多个Sentinel来说，这些Sentinel所设置的down-after-milliseconds选项的值也可能不同，因此，当一个Sentinel将主服务器判断为主观下线时，其他Sentinel可能仍然会认为主服务器处于在线状态。

## 检测客观下线-SENTINEL is-master-down-by-addr命令

**当Sentinel将一个主服务器判断为主观下线之后，为了确认这个主服务器是否真的下线了，它会向同样监视这一主服务器的其他Sentinel进行询问，看它们是否也认为主服务器已经进入了下线状态（可以是主观下线或者客观下线）。当Sentinel从其他Sentinel那里接收到足够数量的已下线判断之后，Sentinel就会将从服务器判定为客观下线，并对主服务器执行故障转移操作。**

### 发送SENTINEL is-master-down-by-addr命令

Sentinel使用：

```
SENTINEL is-master-down-by-addr <ip> <port> <current_epoch> <runid>
```

**命令询问**其他Sentinel是否同意主服务器已下线

> ![image-20230921183802406](https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230921183802406.png)
>
> 举个例子，如果被Sentinel判断为主观下线的主服务器的IP为127.0.0.1，端口号为6379，并且Sentinel当前的配置纪元为0，那么Sentinel将向其他Sentinel发送以下命令：
>
> ```
> SENTINEL is-master-down-by-addr 127.0.0.1 6379 0 *  // *就是用于检测主服务器的客观下线状态
> ```

### 接收SENTINEL is-master-down-by-addr命令

当一个Sentinel（目标Sentinel）接收到另一个Sentinel（源Sentinel）发来的SENTINEL is-master-down-by命令时，**目标Sentinel会分析并取出命令请求中包含的各个参数，并根据其中的主服务器IP和端口号，检查主服务器是否已下线**，然后向源Sentinel返回一条包含三个参数的Multi Bulk回复作为SENTINEL is-master-down-by命令的回复：

```
1) <down_state>
2) <leader_runid>
3) <leader_epoch>Copy to clipboardErrorCopied
```

![image-20230921184236034](https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230921184236034.png)

举个例子，如果一个Sentinel返回以下回复作为SENTINEL is-master-down-by-addr命令的回复：

```
1) 1
2) *   // 表示用于检测主服务器的下线状态, 和发来的SENTINEL is-master-down-by-addr的最后一个参数的*是配对的
3) 0   // 若参数2为*, 则这个无效, 其实就是为了选举领头Sentinel设计的
```

那么说明Sentinel也同意主服务器已下线。若第一个参数为0, 则表示这个Sentinel不认为这个主服务器下线.

### 接收SENTINEL is-master-down-by-addr命令的回复

**根据其他Sentinel发回的SENTINEL is-master-down-by-addr命令回复，Sentinel将统计其他Sentinel同意主服务器已下线的数量，当这一数量达到配置指定的判断客观下线所需的数量时，Sentinel会将主服务器实例结构flags属性的SRI_O_DOWN标识打开，表示主服务器已经进入客观下线状态**

![主服务器被标记为客观下线](https://book-redis-design.netlify.app/images/000122.jpg)

> **客观下线状态的判断条件**
>
> 当认为主服务器已经进入下线状态的Sentinel的数量，超过Sentinel配置中设置的quorum参数的值，那么该Sentinel就会认为主服务器已经进入客观下线状态。比如说，如果Sentinel在启动时载入了以下配置：
>
> ```
> sentinel monitor master 127.0.0.1 6379 2
> ```
>
> 那么包括当前Sentinel在内，只要总共有两个Sentinel认为主服务器已经进入下线状态，那么当前Sentinel就将主服务器判断为客观下线。

> **不同Sentinel判断客观下线的条件可能不同**
>
> 对于监视同一个主服务器的多个Sentinel来说，它们将主服务器标判断为客观下线的条件可能也不同：当一个Sentinel将主服务器判断为客观下线时，其他Sentinel可能并不是那么认为的。比如两个sentinel的quorum参数的值不同即可

## 选举领头Sentinel

**当一个主服务器被判断为客观下线时，监视这个下线主服务器的各个Sentinel会进行协商，选举出一个领头Sentinel，并由领头Sentinel对下线主服务器执行故障转移操作。**

以下是Redis选举领头Sentinel的规则和方法：

- 所有在线的Sentinel都有被选为领头Sentinel的资格，换句话说，监视同一个主服务器的多个在线Sentinel中的任意一个都有可能成为领头Sentinel。
- 每次进行领头Sentinel选举之后，不论选举是否成功，所有Sentinel的配置纪元（configuration epoch）的值都会自增一次。配置纪元实际上就是一个计数器，并没有什么特别的。
- 在一个配置纪元里面，所有Sentinel都有一次将某个Sentinel设置为局部领头Sentinel的机会，并且局部领头一旦设置，在这个配置纪元里面就不能再更改。
- 每个发现主服务器进入客观下线的Sentinel都会要求其他Sentinel将自己设置为局部领头Sentinel。
- <u>当一个Sentinel（源Sentinel）向另一个Sentinel（目标Sentinel）发送SENTINEL is-master-down-by-addr命令，并且命令中的runid参数不是*符号而是源Sentinel的运行ID时，这表示源Sentinel要求目标Sentinel将前者设置为后者的局部领头Sentinel。</u>
- **Sentinel设置局部领头Sentinel的规则是先到先得：最先向目标Sentinel发送设置要求的源Sentinel将成为目标Sentinel的局部领头Sentinel，而之后接收到的所有设置要求都会被目标Sentinel拒绝。**
- <u>目标Sentinel在接收到SENTINEL is-master-down-by-addr命令之后，将向源Sentinel返回一条命令回复，回复中的leader_runid参数和leader_epoch参数分别记录了目标Sentinel的局部领头Sentinel的运行ID和配置纪元。源Sentinel在接收到目标Sentinel返回的命令回复之后，会检查回复中leader_epoch参数的值和自己的配置纪元是否相同以及回复中的leader_runid参数与源Sentinel的运行ID是否一致，若都一致那么表示目标Sentinel将源Sentinel设置成了局部领头Sentinel。</u>
- **如果有某个Sentinel被半数以上的Sentinel设置成了局部领头Sentinel，那么这个Sentinel成为领头Sentinel。**
  举个例子，在一个由10个Sentinel组成的Sentinel系统里面，只要有大于等于10/2+1=6个Sentinel将某个Sentinel设置为局部领头Sentinel，那么被设置的那个Sentinel就会成为领头Sentinel。
- 因为领头Sentinel的产生需要半数以上Sentinel的支持，并且每个Sentinel在每个配置纪元里面只能设置一次局部领头Sentinel，所以在一个配置纪元里面，只会出现一个领头Sentinel。
- 如果在给定时限内，没有一个Sentinel被选举为领头Sentinel，那么各个Sentinel将在一段时间之后再次进行选举，直到选出领头Sentinel为止。

> 所以, 选举领头Sentinel使用的依旧是SENTINEL is-master-down-by-addr命令, 只是里面的参数变了, 这依靠的依旧是Sentinel与Sentinel之间的命令连接

> 为了熟悉以上规则，让我们来看一个选举领头Sentinel的过程。
>
> 假设现在有三个Sentinel正在监视同一个主服务器，并且这三个Sentinel之前已经通过SENTINEL is-master-down-by-addr命令确认主服务器进入了客观下线状态，如图
>
> <img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230921185234497.png" alt="image-20230921185234497" style="zoom:67%;" />
>
> 那么为了选出领头Sentinel，三个Sentinel将再次向其他Sentinel发送SENTINEL is-master-down-by-addr命令，如图
>
> <img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230921185255120.png" alt="image-20230921185255120" style="zoom:67%;" />
>
> 和检测客观下线状态时发送的SENTINEL is-master-down-by-addr命令不同，Sentinel这次发送的命令会带有Sentinel自己的运行ID，例如：
>
> ```
> SENTINEL is-master-down-by-addr 127.0.0.1 6379 0 e955b4c85598ef5b5f055bc7ebfd5e828dbed4fa
> ```
>
> 如果接收到这个命令的Sentinel还没有设置局部领头Sentinel的话，它就会将运行ID为e955b4c85598ef5b5f055bc7ebfd5e828dbed4fa的Sentinel设置为自己的局部领头Sentinel，并返回类似以下的命令回复：
>
> ```
> 1) 1
> 2) e955b4c85598ef5b5f055bc7ebfd5e828dbed4fa
> 3) 0
> ```
>
> 然后接收到命令回复的Sentinel就可以根据这一回复，统计出有多少个Sentinel将自己设置成了局部领头Sentinel。
>
> 根据命令请求发送的先后顺序不同，可能会有某个Sentinel的SENTINEL is-master-down-by-addr命令比起其他Sentinel发送的相同命令都更快到达，并最终胜出领头Sentinel的选举，然后这个领头Sentinel就可以开始对主服务器执行故障转移操作了。

> 简⽽⾔之, Raft 算法的核⼼就是 "先下⼿为强". 谁率先发出了拉票请求, 谁就有更⼤的概率成为 leader。这⾥的决定因素成了 "⽹络延时". ⽹络延时本⾝就带有⼀定随机性. 具体选出的哪个节点是 leader, 这个不重要, 重要的是能选出⼀个节点即可.
>

## 故障转移

### 1) 选出新的主服务器

故障转移操作第一步要做的就是**在已下线主服务器属下的所有从服务器中，挑选出一个状态良好、数据完整的从服务器，然后向这个从服务器发送SLAVEOF no one命令，将这个从服务器转换为主服务器。**

> **新的主服务器是怎样挑选出来的**
>
> 领头Sentinel会将已下线主服务器的所有从服务器保存到一个列表里面，然后按照以下规则，一项一项地对列表进行过滤：
>
> 遵循先筛后选: 
>
> 1）删除列表中所有处于下线或者断线状态的从服务器，这可以保证列表中剩余的从服务器都是正常在线的。
>
> 2）删除列表中所有最近五秒内没有回复过领头Sentinel的INFO命令的从服务器，这可以保证列表中剩余的从服务器都是最近成功进行过通信的。
>
> 3）删除所有与已下线主服务器连接断开超过down-after-milliseconds\*10毫秒的从服务器：down-after-milliseconds选项指定了判断主服务器下线所需的时间，而删除断开时长超过down-after-milliseconds*10毫秒的从服务器，则可以保证列表中剩余的从服务器都没有过早地与主服务器断开连接，换句话说，列表中剩余的从服务器保存的数据都是比较新的。
>
> 之后，领头Sentinel将根据从服务器的优先级, 复制偏移量offset, 运行ID选出一个从服务器, 若前一个标准相等, 则按照后一个标准选择: 优先级越高(优先级是配置⽂件中的配置项 slave-priority 或者 replica-priority), 复制偏移量越大(复制偏移量最大的从服务器就是保存着最新/多数据的从服务器), 运行ID越小

下图展示了在一次故障转移操作中，领头Sentinel向被选中的从服务器server2发送SLAVEOF no one命令的情形。

<img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230921205726765.png" alt="image-20230921205726765" style="zoom:67%;" />

> 将server2通过SLAVEOF no one命令升级为主服务器

在发送SLAVEOF no one命令之后，领头Sentinel会以每秒一次的频率（平时是每十秒一次），向被升级的从服务器发送INFO命令，并观察命令回复中的角色（role）信息，当被升级服务器的role从原来的slave变为master时，领头Sentinel就知道被选中的从服务器已经顺利升级为主服务器了。

例如，在上图展示的例子中，领头Sentinel会一直向server2发送INFO命令，当server2返回的命令回复从：

```
# Replication
role:slave
...
# Other sections
...
```

变为：

```
# Replication
role:master
...
# Other sections
...
```

的时候，领头Sentinel就知道server2已经成功升级为主服务器了。

<img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230921205955971.png" alt="image-20230921205955971" style="zoom:67%;" />

> server2成功升级为主服务器

上图展示了server2升级成功之后，各个服务器和领头Sentinel的样子。

### 2) 修改从服务器的复制目标

**当新的主服务器出现之后，领头Sentinel下一步要做的就是，让已下线主服务器属下的所有从服务器去复制新的主服务器，这一动作可以通过向从服务器发送SLAVEOF命令来实现。**

> 下图展示了在故障转移操作中，领头Sentinel向已下线主服务器server1的两个从服务器server3和server4发送SLAVEOF命令，让它们复制新的主服务器server2的例子。
>
> <img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230921210119818.png" alt="image-20230921210119818" style="zoom:67%;" />
>
> 让从服务器复制新的主服务器
>
> server3和server4成为server2的从服务器之后，各个服务器以及领头Sentinel的样子。
>
> <img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230921210225389.png" alt="image-20230921210225389" style="zoom:67%;" />
>
> server3和server4成为server2的从服务器

### 3) 将旧的主服务器变为从服务器

**故障转移操作最后要做的是，将已下线的主服务器设置为新的主服务器的从服务器。**

**因为旧的主服务器已经下线，所以这种设置是保存在server1对应的实例结构里面的，当server1重新上线时，Sentinel就会向它发送SLAVEOF命令，让它成为server2的从服务器。**

<img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230921210425594.png" alt="image-20230921210425594" style="zoom:67%;" />

> server1被设置为新主服务器的从服务器

<img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230921210418026.png" alt="image-20230921210418026" style="zoom:67%;" />

> server1重新上线并成为server2的从服务器

# 重点回顾

- **Sentinel 只是一个运行在特殊模式下的 Redis 服务器**， 它使用了和普通模式不同的命令表， 所以 Sentinel 模式能够使用的命令和普通 Redis 服务器能够使用的命令不同。
- **Sentinel 会读入用户指定的配置文件， 为每个要被监视的主服务器创建相应的实例结构， 并创建连向主服务器的命令连接和订阅连接， 其中命令连接用于向主服务器发送命令请求， 而订阅连接则用于接收指定频道的消息。**
- **Sentinel 通过向主服务器发送 INFO 命令来获得主服务器属下所有从服务器的地址信息， 并为这些从服务器创建相应的实例结构， 以及连向这些从服务器的命令连接和订阅连接。**
- 在一般情况下， Sentinel 以每十秒一次的频率向被监视的主服务器和从服务器发送 INFO 命令， 当主服务器处于下线状态， 或者 Sentinel 正在对主服务器进行故障转移操作时， Sentinel 向从服务器发送 INFO 命令的频率会改为每秒一次。
- 对于监视同一个主服务器和从服务器的多个 Sentinel 来说， 它们会以每两秒一次的频率， 通过向被监视服务器的 `__sentinel__:hello` 频道发送消息来向其他 Sentinel 宣告自己的存在。
- **每个 Sentinel 也会从 `__sentinel__:hello` 频道中接收其他 Sentinel 发来的信息， 并根据这些信息为其他 Sentinel 创建相应的实例结构， 以及命令连接。**
- <u>Sentinel 只会与主服务器和从服务器创建命令连接和订阅连接， Sentinel 与 Sentinel 之间则只创建命令连接。</u>
  (事实上, 在检测, 故障转移的过程中, 只是用了sentinel之间的命令连接发送命令, 以及sentinel与主从服务器之间的命令连接来发送命令, 并不使用订阅连接, 订阅连接只用于sentinel与sentinel之间发现/持续获知对方的存在(目前看来))



- Sentinel 以每秒一次的频率向实例（包括主服务器、从服务器、其他 Sentinel）发送 PING 命令， 并根据实例对 PING 命令的回复来判断实例是否在线： 当一个实例在指定的时长中连续向 Sentinel 发送无效回复时， Sentinel 会将这个实例判断为主观下线。
- 当 Sentinel 将一个主服务器判断为主观下线时， 它会向同样监视这个主服务器的其他 Sentinel 进行询问， 看它们是否同意这个主服务器已经进入主观下线状态。(同样通过命令)
- 当 Sentinel 收集到足够多的主观下线投票之后， 它会将主服务器判断为客观下线， 并在选举出leader sentinel之后, 由leader sentinel发起一次针对主服务器的故障转移操作。(其中选举, 故障转移都是通过命令的方式~)

# 补充

## Redis Sentinel 架构

**当主节点出现故障时，Redis Sentinel 能自动完成故障发现和故障转移，并通知应用方，从而实现真正的高可用。**

![image-20230912123853297](C:\Users\yangzilong\AppData\Roaming\Typora\typora-user-images\image-20230912123853297.png)

## Redis Sentinel 具有以下几个功能

- **监控**: Sentinel 节点会定期检测 Redis 数据节点、其余哨兵节点是否可达。

- **自动的故障转移**: 如果master故障, sentinel实现从节点晋升（promotion）为主节点并维护后续正确的主从关系。

- **通知**: Sentinel 节点会将故障转移的结果通知给应⽤⽅。

## 部署redis哨兵 (基于 docker)

todo...

## 小结

**上述过程, 都是 "无人值守" , Sentinel 自动完成的. 这样做就解决了主节点宕机之后需要人工干预的问题, 提高了系统的可用性.**

⼀些注意事项:

- 哨兵节点不能只有⼀个. 否则哨兵节点挂了也会影响系统可⽤性.(这就是为什么是哨兵节点集群，部署多个redis-sentinel)
- 哨兵节点最好是奇数个. ⽅便选举 leader sentinel, 得票更容易超过半数.
- 哨兵节点不负责存储数据. 仍然是 redis 主从节点负责存储.
- 哨兵 + 主从复制解决的问题是 "提⾼可⽤性"，自动化解决主节点故障，没有提高写操作的并发量.
- 哨兵 + 主从复制不能提⾼数据的存储容量. 当我们需要存的数据接近或者超过机器的物理内存, 这样的结构就难以胜任了.

为了能存储更多的数据, 就引⼊了集群.  其实集群一方面使得redis存储的数据增多了，另一方面也就提高了写并发，因为此时一个分片只需承担 1 / 分片数量 的写并发。
