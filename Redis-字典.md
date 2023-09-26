字典, 是一种用于保存键值对（key-value pair）的抽象数据结构。

在字典中， 一个键（key）可以和一个值（value）进行关联（或者说将键映射为值）， 这些关联的键和值就被称为**键值对**。

字典中的每个键都是独一无二的， 程序可以在字典中根据键查找与之关联的值， 或者通过键来更新值， 又或者根据键来删除整个键值对， 等等。

字典经常作为一种数据结构内置在很多高级编程语言里面， 但 Redis 所使用的 C 语言并没有内置这种数据结构， 因此 Redis 构建了自己的字典实现。

字典在 Redis 中的应用相当广泛， 比如 **Redis 的数据库就是使用字典来作为底层实现的**， 对数据库的增、删、查、改操作也是构建在对字典的操作之上的。

除了用来表示数据库之外， 字典还是哈希键的底层实现之一： 当一个哈希键包含的键值对比较多， 又或者键值对中的元素都是比较长的字符串时， Redis 就会使用字典作为哈希键的底层实现。

# 字典实现

Redis 的字典使用哈希表作为底层实现(注意, 字典并非只能用哈希表实现, 用有序数组, 红黑, AVL, 跳表等数据结构都可以实现)， 一个哈希表里面可以有多个哈希表节点， 而每个哈希表节点就保存了字典中的一个键值对。

接下来将分别介绍 Redis 的哈希表、哈希表节点、以及字典的实现。

## 哈希表

Redis 字典所使用的哈希表由 `dict.h/dictht` 结构定义：

```C
typedef struct dictht {

    // 哈希表数组
    dictEntry **table;   // dictEntry *table[]

    // 哈希表大小
    unsigned long size;   // table数组的大小

    // 哈希表大小掩码，用于计算索引值
    // 总是等于 size - 1
    unsigned long sizemask;

    // 该哈希表已有节点的数量
    unsigned long used;   // 并非一定等于size, 记录的是已有节点的数量, 也就是说table数组可能有空余空间

} dictht;
```

`table` 属性是一个数组， 数组中的每个元素都是一个指向 `dict.h/dictEntry` 结构的指针， 每个 `dictEntry` 结构保存着一个键值对。

`size` 属性记录了哈希表的大小， 也即是 `table` 数组的大小， 而 `used` 属性则记录了哈希表目前已有节点（键值对）的数量。

`sizemask` 属性的值总是等于 `size - 1` (这个属性和哈希值一起决定一个键应该被放到 `table` 数组的哪个索引上面。)

示例: 一个空的哈希表

![image-20230926205111593](https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230926205111593.png)

## 哈希表节点

哈希表节点使用 `dictEntry` 结构表示， 每个 `dictEntry` 结构都保存着一个键值对：

```C
typedef struct dictEntry {

    // 键
    void *key;

    // 值
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
    } v;

    // 指向下个哈希表节点，形成链表
    struct dictEntry *next;    // 使用开散列-链地址法解决哈希冲突

} dictEntry;
```

`key` 属性保存着键值对中的键， 而 `v` 属性则保存着键值对中的值， 其中键值对的值可以是一个指针， 或者是一个 `uint64_t` 整数， 又或者是一个 `int64_t` 整数。

`struct dictEntry *next; `数据成员是因为哈希表使用开散列-链地址法解决哈希冲突: `next` 属性是指向另一个哈希表节点的指针， 这个指针可以将多个哈希值相同的键值对连接在一次， 以此来解决键冲突（collision）的问题。

示例: k1 k0的哈希值相同, 发生了哈希冲突~

![image-20230926205430624](https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230926205430624.png)

## 字典

Redis 中的字典由 `dict.h/dict` 结构表示：

```C
typedef struct dict {

    // 类型特定函数
    dictType *type;

    // 私有数据
    void *privdata;

    // 哈希表
    dictht ht[2];

    // rehash 索引, 当 rehash 不在进行时，值为 -1
    int rehashidx; /* rehashing not in progress if rehashidx == -1 */

} dict;
```

`ht` 属性是一个包含两个哈希表的数组， 数组中的每个项都是一个 `dictht` 哈希表， 一般情况下， 字典只使用 `ht[0]` 哈希表， `ht[1]` 哈希表只会在对 `ht[0]` 哈希表进行 rehash 时使用。

除了 `ht[1]` 之外， 另一个和 rehash 有关的属性就是 `rehashidx` ： 它记录了 rehash 目前的进度， 如果目前没有在进行 rehash ， 那么它的值为 `-1` 。

`type` 属性和 `privdata` 属性是针对不同类型的键值对， 为创建**多态字典**而设置的：

- `type` 属性是一个指向 `dictType` 结构的指针， 每个 `dictType` 结构保存了一簇用于操作特定类型键值对的函数， Redis 会为用途不同的字典设置不同的类型特定函数。
- 而 `privdata` 属性则保存了需要传给那些类型特定函数的可选参数。

```C
typedef struct dictType {

    // 计算哈希值的函数
    unsigned int (*hashFunction)(const void *key);

    // 复制键的函数
    void *(*keyDup)(void *privdata, const void *key);

    // 复制值的函数
    void *(*valDup)(void *privdata, const void *obj);

    // 对比键的函数
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);

    // 销毁键的函数
    void (*keyDestructor)(void *privdata, void *key);

    // 销毁值的函数
    void (*valDestructor)(void *privdata, void *obj);

} dictType;
```

示例: 一个没有进行rehash的字典

![image-20230926205850629](https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230926205850629.png)

# 哈希算法

当要将一个新的键值对添加到字典里面时， 程序需要先根据键值对的键通过哈希函数计算出哈希值和索引值(哈希值 & 哈希表大小-1 == 索引值)， 然后再根据索引值， 将包含新键值对的哈希表节点dictEntry类型对象放到哈希表数组的指定索引上面。

Redis 计算哈希值和索引值的方法如下：

```c
# 使用字典设置的哈希函数，计算键 key 的哈希值
hash = dict->type->hashFunction(key);

# 使用哈希值和哈希表的 sizemask 属性，计算出索引值
# 根据情况不同， ht[x] 可以是 ht[0] 或者 ht[1]
index = hash & dict->ht[x].sizemask;
```

<img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230926210159471.png" alt="image-20230926210159471" style="zoom: 67%;" />

当字典被用作数据库的底层实现， 或者哈希键的底层实现时， Redis 使用 MurmurHash2 算法来计算键的哈希值。

# 解决哈希冲突

当有两个或以上数量的键通过哈希算法计算得到的哈希值 以及 索引值相同时, 即发生了哈希冲突

Redis 的哈希表使用链地址法（separate chaining）来解决键冲突： 每个哈希表节点都有一个 `next` 指针， 多个哈希表节点可以用 `next` 指针构成一个**单向链表**， 被分配到同一个索引上的多个节点可以用这个单向链表连接起来， 这就解决了键冲突的问题。

![image-20230926210621274](https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230926210621274.png)

举例: 当k2和k1两个键发生哈希冲突时, 就会以如图所示的简单单链表的方式组织起来, 以此解决哈希冲突

因为 `dictEntry` 节点组成的链表只有next指针, 没有指向链表表尾的指针， 所以为了速度考虑， 程序总是将新节点添加到链表的表头位置（复杂度为 O(1)）， 排在其他已有节点的前面。(头插)

# rehash

> rehash简单来说就是哈希表的扩容或缩容, 哈希的关键概念之一就是负载因子, 负载因子等于哈希表中已有键值对数量 / 哈希表大小
>
> 当负载因子过大时需要扩容, 负载因子过小时需要缩容, 也就是存储的键值对过多或过少时, 过多会使得哈希冲突概率增加, 过少会浪费空间, 而不管是扩容还是缩容, rehash就是进行重新散列的操作

随着操作的不断执行， 哈希表保存的键值对会逐渐地增多或者减少， 为了让哈希表的负载因子（load factor）维持在一个合理的范围之内， **当哈希表保存的键值对数量太多或者太少时， 程序需要对哈希表的大小进行相应的扩展或者收缩。**

扩展和收缩哈希表的工作可以通过执行 rehash （重新散列）操作来完成， Redis 对字典的哈希表执行 rehash 的步骤如下：

1. 为字典的`ht[1]`哈希表分配空间， ht[1]哈希表的空间大小取决于要执行的操作， 以及`ht[0]`当前包含的键值对数量 （也即是`ht[0].used`属性的值）：
   - 如果执行的是扩展操作， 那么 `ht[1]` 的大小为第一个大于等于 `ht[0].used * 2` 的 2^n （`2` 的 `n` 次方幂）；
   - 如果执行的是收缩操作， 那么 `ht[1]` 的大小为第一个大于等于 `ht[0].used` 的 2^n 。
   - 总之, 哈希表的大小总是维持在2^n
2. 将保存在 `ht[0]` 中的所有键值对 rehash 到 `ht[1]` 上面： rehash 指的是重新计算键的哈希值和索引值， 然后将键值对放置到 `ht[1]` 哈希表的指定位置上。(因为此时哈希表大小变了~)
3. 当 `ht[0]` 包含的所有键值对都迁移到了 `ht[1]` 之后 （`ht[0]` 变为空表）， 释放 `ht[0]` ， 将 `ht[1]` 设置为 `ht[0]` ， 并在 `ht[1]` 新创建一个空白哈希表， 为下一次 rehash 做准备。

> 举例:
>
> 假设程序要对下图所示字典的 `ht[0]` 进行扩展操作， 那么程序将执行以下步骤：
>
> 1. `ht[0].used` 当前的值为 `4` ， `4 * 2 = 8` ， 而 `8` （2^3）恰好是第一个大于等于 `4` 的 `2` 的 `n` 次方， 所以程序会将 `ht[1]` 哈希表的大小设置为 `8` 。 
> 2. 将 `ht[0]` 包含的四个键值对都 rehash 到 `ht[1]`
> 3. 释放 `ht[0]` ，并将 `ht[1]` 设置为 `ht[0]` ，然后为 `ht[1]` 分配一个空白哈希表
>
> 至此， 对哈希表的扩展操作执行完毕， 程序成功将哈希表的大小从原来的 `4` 改为了现在的 `8` 。
>
> <img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230926211551716.png" alt="image-20230926211551716" style="zoom:50%;" />
>
> <img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230926211606749.png" alt="image-20230926211606749" style="zoom:50%;" />
>
> <img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230926211620899.png" alt="image-20230926211620899" style="zoom:50%;" />
>
> <img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230926211634711.png" alt="image-20230926211634711" style="zoom:50%;" />

## 哈希表的扩展和收缩/扩容与缩容

扩容:

- 服务器目前没有在执行 BGSAVE 命令或者 BGREWRITEAOF 命令， 且哈希表的负载因子 >= `1` ；

- 服务器目前正在执行 BGSAVE 命令或者 BGREWRITEAOF 命令， 且哈希表的负载因子>= `5` ；

缩容:

- 哈希表负载因子 < 0.1

<u>根据 BGSAVE 命令或 BGREWRITEAOF 命令是否正在执行， 服务器执行扩展操作所需的负载因子并不相同</u>， 这是因为在执行 BGSAVE 命令或 BGREWRITEAOF 命令的过程中， Redis 需要创建当前服务器进程的子进程， 而大多数操作系统都采用写时复制技术来优化子进程的使用效率， **所以在子进程存在期间， 服务器会提高执行扩展操作所需的负载因子， 从而尽可能地避免在子进程存在期间进行哈希表扩展操作**， 这可以避免不必要的内存写入操作， 最大限度地节约内存。(子进程写时拷贝, 如果此时进行rehash, 父进程会有大量内存进行写入操作, 而子进程虽然和rehash没有关系, 但是因为写时拷贝, 就会有大量内存进行没必要的写时拷贝, 因此提高哈希因子, 提高rehash的门槛)

> 其中哈希表的负载因子可以通过公式计算得出。
>
> ```
> # 负载因子 = 哈希表已保存节点数量 / 哈希表大小
> load_factor = ht[0].used / ht[0].size
> ```
>
> 又比如说， 对于一个大小为 `512` ， 包含 `256` 个键值对的哈希表来说， 这个哈希表的负载因子为：
>
> ```
> load_factor = 256 / 512 = 0.5
> ```

## 渐进式rehash

<u>扩展或收缩哈希表需要将 `ht[0]` 里面的所有键值对 rehash 到 `ht[1]` 里面， 因为这是一个比较耗时的操作, 故这个 rehash 动作并不是一次性、集中式地完成的， 而是分多次、渐进式地完成的。</u>

这样做的原因在于， 如果 `ht[0]` 里只保存着四个键值对， 那么服务器可以在瞬间就将这些键值对全部 rehash 到 `ht[1]` ； 但是， 如果哈希表里保存的键值对数量不是四个， 而是四百万、四千万甚至四亿个键值对， 那么要一次性将这些键值对全部 rehash 到 `ht[1]` 的话， **庞大的计算量可能会导致服务器在一段时间内停止服务。**(注意, Redis本身就是使用字典来存储所有键值对数据的)

**因此， 为了避免 rehash 对服务器性能造成影响**， 服务器不是一次性将 `ht[0]` 里面的所有键值对全部 rehash 到 `ht[1]` ， 而是分多次、渐进式地将 `ht[0]` 里面的键值对慢慢地 rehash 到 `ht[1]` 。

**以下是哈希表渐进式 rehash 的详细步骤：**

1. 为 `ht[1]` 分配空间， 让字典同时持有 `ht[0]` 和 `ht[1]` 两个哈希表。(分配空间的细节前面说了)
2. 在字典中维持一个索引计数器变量 `rehashidx` ， 并将它的值设置为 `0` ， 表示 rehash 工作正式开始。(不进行rehash时为-1)
3. 在 rehash 进行期间， 每次对字典执行添加、删除、查找或者更新操作时， 程序除了执行指定的操作以外， 还会顺带将 `ht[0]` 哈希表在 `rehashidx` 索引上的所有键值对 rehash 到 `ht[1]` ， 当 rehash 工作完成之后， 程序将 `rehashidx` 属性的值增一。便于下次对字典执行添加、删除、查找或者更新操作时, rehash下一个索引处的键值对.
4. 随着字典操作的不断执行， 最终在某个时间点上， `ht[0]` 的所有键值对都会被 rehash 至 `ht[1]` ， 这时程序将 `rehashidx` 属性的值设为 `-1` ， 表示 rehash 操作已完成。

**渐进式 rehash 的好处在于它采取分而治之的方式， 将 rehash 键值对所需的计算工作均滩到对字典的每个添加、删除、查找和更新操作上， 从而避免了集中式 rehash 而带来的庞大计算量。**

因此, 渐进式rehash具体完成的时间是不定的, 取决于字典的每个添加、删除、查找和更新操作的频率

**渐进式 rehash 执行期间的哈希表操作**

因为在进行渐进式 rehash 的过程中， 字典会同时使用 `ht[0]` 和 `ht[1]` 两个哈希表， 所以在渐进式 rehash 进行期间， 字典的删除（delete）、查找（find）、更新（update）等操作会在两个哈希表上进行： 比如说， 要在字典里面查找一个键的话， 程序会先在 `ht[0]` 里面进行查找， 如果没找到的话， 就会继续到 `ht[1]` 里面进行查找， 诸如此类。因为rehash期间并不确定当前这个操作的目标键值对是否已经rehash到了ht[1]中

<u>另外， 在渐进式 rehash 执行期间， 新添加到字典的键值对一律会被保存到 `ht[1]` 里面， 而 `ht[0]` 则不再进行任何添加操作： 这一措施保证了 `ht[0]` 包含的键值对数量会只减不增， 并随着 rehash 操作的执行而最终变成空表。</u>

> rehash的全过程示例:
>
> 初始状态: 负载因子为1, 需要进行扩容, 即rehash, 目前rehashidx还为-1
>
> <img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230926213018416.png" alt="image-20230926213018416" style="zoom:50%;" />
>
> 数据库进行了增删查改操作, rehash ht[0]中索引值为0的所有键值对(只有一个)
>
> <img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230926213028806.png" alt="image-20230926213028806" style="zoom:50%;" />
>
> 数据库进行了增删查改操作, rehash ht[0]中索引值为1的所有键值对(只有一个)
>
> <img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230926213039098.png" alt="image-20230926213039098" style="zoom:50%;" />
>
> 数据库进行了增删查改操作, rehash ht[0]中索引值为2的所有键值对(只有一个)
>
> <img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230926213048804.png" alt="image-20230926213048804" style="zoom:50%;" />
>
> 数据库进行了增删查改操作, rehash ht[0]中索引值为3的所有键值对(只有一个)
>
> <img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230926213100465.png" alt="image-20230926213100465" style="zoom:50%;" />
>
> 数据库进行了增删查改操作, rehash ht[0]中索引值为4的所有键值对(只有一个), 释放ht[0], 交换ht[0]和ht[1], 为ht[1]创建一个空白哈希表, rehashidx设为-1, 为下次rehash做准备
>
> <img src="https://cdn.jsdelivr.net/gh/DaysOfExperience/blogImage@main/img/image-20230926213113489.png" alt="image-20230926213113489" style="zoom:50%;" />

# 重点回顾

- 字典被广泛用于实现 Redis 的各种功能， 其中包括数据库和哈希键。
- Redis 中的字典使用哈希表作为底层实现， 每个字典带有两个哈希表， 一个用于平时使用， 另一个仅在进行 rehash 时使用。
- 当字典被用作数据库的底层实现， 或者哈希键的底层实现时， Redis 使用 MurmurHash2 算法来计算键的哈希值。
- 哈希表使用链地址法来解决键冲突， 被分配到同一个索引上的多个键值对会连接成一个单向链表。
- **在对哈希表进行扩展或者收缩操作时， 程序需要将现有哈希表包含的所有键值对 rehash 到新哈希表里面， 并且这个 rehash 过程并不是一次性地完成的， 而是渐进式地完成的。**
