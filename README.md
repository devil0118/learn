## 1. 是什么

Redis「**R**emote **D**ictionary **S**ervice」是基于内存的存储中间件，通常用于数据库、缓存、消息队列。

## 2. 基础 - 数据结构

Redis 有 5 中数据结构：string、list、hash、set 和 zset（有序集合），分别对应 Java 中的 String、LinkedList、HashMap、Set、SortedSet。

### redisObject

```c
typedef struct redisObject {
    // 对外的类型 string list set hash zset等 4bit
    unsigned type:4;
    // 底层存储方式 4bit
    unsigned encoding:4;
    // LRU 时间 24bit
    unsigned lru:LRU_BITS; 
    // 引用计数  4byte
    int refcount;
    // 指向对象的指针  8byte
    void *ptr;
} robj;
```

type 、encoding 和 ptr 是redisObject最重要的三个属性。

##### type

type字段表示对象的类型，占4个bit，它记录了对象所保存的值的类型。看见type这个单词，是不是感觉有些熟悉，没错，我们平时查看使用客户端查看对象类型时的命令。如下所示：

<img src="/Users/wangbing/Library/Application Support/typora-user-images/image-20210219095952001.png" alt="image-20210219095952001" style="zoom:50%;" />

type的常见取值如下：

```c
/*
* 对象类型
*/
#define REDIS_STRING 0 // 字符串
#define REDIS_LIST 1   // 列表
#define REDIS_SET 2    // 集合
#define REDIS_ZSET 3   // 有序集
#define REDIS_HASH 4   // 哈希表
```

##### encoding

encoding表示对象的内部编码，占4个bit，其实也就是该对象底层到底使用的哪种数据结构来存储。由于Redis对内存进行了极致的利用，为了节省空间，Redis底层会根据对象大小使用不同的数据结构来存储。通过encoding属性，Redis可以根据不同的使用场景来为对象设置不同的编码，大大提高了Redis的灵活性和效率。例如，常见的string类型，其底层实现就有int、embstr和raw三种编码方式。使用 object encoding key就可以查看对象的编码方式。

<img src="/Users/wangbing/Library/Application Support/typora-user-images/image-20210219100054481.png" alt="image-20210219100054481" style="zoom:50%;" />

5种对象类型对应的编码方式将在后面细说，这里先留下一个印象。而在redisObject中，encoding的取值有以下这些：

```c
/*
* 对象编码
*/
#define REDIS_ENCODING_RAW 0    // 编码为字符串
#define REDIS_ENCODING_INT 1    // 编码为整数
#define REDIS_ENCODING_HT 2     // 编码为哈希表
#define REDIS_ENCODING_ZIPMAP 3 // 编码为 zipmap(2.6 后不再使用)
#define REDIS_ENCODING_LINKEDLIST 4 // 编码为双端链表
#define REDIS_ENCODING_ZIPLIST 5    // 编码为压缩列表
#define REDIS_ENCODING_INTSET 6     // 编码为整数集合
#define REDIS_ENCODING_SKIPLIST 7    // 编码为跳跃表
```

##### ptr

ptr指针指向具体的数据，如 `set test hello`，ptr指向的就是存储`hello`的字符串。

##### lru

lru记录的是对象最后一次被命令程序访问的时间，占用的bit位因版本不同而不同，2.6版本22bit、4.0版本24bit。

通过对比lru时间与当前时间，可以计算某个对象的空转时间；object idletime命令可以显示该空转时间（单位是秒）。object idletime命令的一个特殊之处在于它不改变对象的lru值。

<img src="/Users/wangbing/Library/Application Support/typora-user-images/image-20210219100315077.png" alt="image-20210219100315077" style="zoom:50%;" />

另外，说到lru，是不是感觉特别熟悉。没错，Redis的缓存淘汰策略中，就有`allkeys-lru`和`volatile-lru`两种，当redis.conf中设置了maxmemory选项时，若是Redis内存占用超过maxmemory指定的值时，Redis会优先选择空转时间长的对象进行释放。

##### refcount

refcount记录的是该对象被引用的次数，类型为整型。refcount的作用主要在于对象的引用计数和内存回收。当创建对象时，refcount初始化为1；当有新程序使用该对象时，refcount++；当对象不再被一个新程序使用时，refcount--；当refcount变为0时，对象占用的内存会被释放。

Redis中被多次使用的对象(refcount>1)的被称为共享对象。Redis为了节省内存，当有一些重复对象出现时，新的程序不会创建新的对象，而是直接使用重复对象，然后refcount++。需要注意的是，**目前共享对象仅仅支持整数值的字符串对象。**

之所以只支持整数型共享对象，是出于对内存和CPU（时间）的平衡：共享对象虽然会降低内存消耗，但是判断两个对象是否相等却需要消耗额外的时间。对于整数值，判断操作复杂度为O(1)；对于普通字符串，判断复杂度为O(n)；而对于哈希、列表、集合和有序集合，判断的复杂度为O(n^2)。

就目前的实现来说，Redis服务器在初始化时，会创建10000个字符串对象，值分别是09999的整数值；当Redis需要使用值为09999的字符串对象时，可以直接使用这些共享对象。10000这个数字可以通过调整参数REDIS_SHARED_INTEGERS（4.0中是OBJ_SHARED_INTEGERS）的值进行改变。

共享对象的引用次数可以通过object refcount命令查看，如下图所示。命令执行的结果页佐证了只有0~9999之间的整数会作为共享对象。

<img src="/Users/wangbing/Library/Application Support/typora-user-images/image-20210219100407166.png" alt="image-20210219100407166" style="zoom:50%;" />

##### 总结

综上所述，redisObject的结构与对象类型、编码、内存回收、共享对象都有关系，一个redisObject的大小约为16字节:4 bit+4 bit+24 bit +4 byte+8 byte=16 byte。

### string 字符串

string 是 Redis 中最常见的数据结构，也称 SDS「**S**imple **D**ynamic **S**tring」，通常作为 key 或 value（最大长度 512 M）。图中粉色部分为 Redis 对象的通用头部，ptr 指向 SDS。string 按长短分以 embstr（len <= 44） 和 raw 的形式存储。

<img src="/Users/wangbing/Library/Application Support/typora-user-images/image-20210209122857763.png" alt="image-20210209122857763" style="zoom:45%;" />

```c
struct sdsshr<T>{
    T len;//数组长度
    T alloc;//数组容量
    unsigned  flags;//sdshdr类型
    char buf[];//数组内容
}
```

可以看出，`SDS`的结构有点类似于`Java`中的`ArrayList`。`buf[]`表示真正存储的字符串内容，`alloc`表示所分配的数组的长度，`len`表示字符串的实际长度，并且由于`len`这个属性的存在，`Redis`可以在`O(1)`的时间复杂度内获取数组长度。

为了追求对于内存的极致优化，对于不同长度的字符串，`Redis`底层会采用不同的结构体来表示。在`Redis`中的`sds.h`源码中存在着五种`sdshdr`，分别如下：

```c
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* used */
    uint16_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* used */
    uint32_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```

上面说了，`Redis`底层会根据字符串的长度来决定具体使用哪种类型的`sdshdr`。可以看出，`sdshdr5`明显区别于其他四种结构，它一般只用于存储长度不会变化，且长度小于32个字符的字符串。但现在一般都不再使用该结构，**因为其结构没有`len`和`alloc`这两个属性,不具备动态扩容操作**，一旦预分配的内存空间使用完，就需要重新分配内存并完成数据的复制和迁移，类似于`ArrayList`的扩容操作，这种操作对性能的影响很大。

上面介绍`sdshdr`属性的时候说过，`flag`这个属性用于标识使用哪种`sdshdr`类型，`flag`的低三位标识当前`sds`的类型，分别如下所示：

```c
#define SDS_TYPE_5  0
#define SDS_TYPE_8  1
#define SDS_TYPE_16 2
#define SDS_TYPE_32 3
#define SDS_TYPE_64 4
```

同时，注意到在每个`sdshdr`的头定义上都有一个`attribute((packed))`，这个是为了告诉`gcc`**取消优化对齐**，这样，每个字段分配的内存地址就是**紧紧排列在一起的**，`Redis`中字符串参数的传递直接使用`char*`指针，其实现原理在于，由于`sdshdr`内存分配禁止了优化对齐，所以`sds[-1]`指向的就是`flags`属性的内存地址，而通过`flags`属性又可以确定`sdshdr`的属性，进而可以读取头部字段确定`sds`的相关属性。

<img src="/Users/wangbing/Library/Application Support/typora-user-images/image-20210219100719972.png" alt="image-20210219100719972" style="zoom:50%;" />

##### sdshdr的优势

相比较于`C`语言原始的字符串，`sdshdr`的具备一些优势。

##### 长度获取

由于`sdshdr`中存在`len`这个属性，所以可以在`O(1)`的时间复杂度下获得长度；而传统的`C`语言得使用`strlen`这个标准函数获取，时间复杂度为`O(N)`。

##### 避免频繁的内存分配

原始的`C`语言一直使用与长度匹配的内存，这样在追加字符串导致字符串长度发生变化时，就必须进行内存的重新分配。内存重新分配涉及到复杂算法和系统调用，耗费性能和时间。对于`Redis`来说，它是单线程的，如果使用原始的字符串结构，势必会引发频繁的内存重分配，这个显然是不合理的。

因而，`sds`每次进行内存分配时，都会通过内存的预分配来减少因为修改字符串而引发的内存重分配次数。这个原理可以参数`Java`中的`ArrayList`，一般在使用`ArrayList`时都会建议使用带有容量的构造方式，这样可以避免频繁`resize`。

对于`SDS`来说，当其使用`append`进行字符串追加时，程序会用 **alloc-len 比较下剩下的空余内存是否足够分配追加的内容**，如果不够自然触发内存重分配，而如果剩余未使用内存空间足够放下，那么将直接进行分配，无需内存重分配。其扩容策略为，**当字符串占用大小小于1M时，每次分配为`len` \* 2，也就是保留100%的冗余；大于1M后，为了避免浪费，只多分配1M的空间。**

通过这种预分配策略， SDS 将连续增长 N 次字符串所需的内存重分配次数**从必定 N 次降低为最多 N 次。**

##### 缓冲区溢出

**缓冲区溢出是指当某个数据超过了处理程序限制的范围时，程序出现的异常操作。**原始的`C`语言中，是由编码者自己来分配字符串的内存，当出现内存分配不足时就会发生**缓存区溢出**。而`sds`的修改函数在修改前会判断内存，动态的分配内存，杜绝了**缓冲区溢出**的可能性。

##### 二进制安全

对于原始的`C`语言字符串来说，它会通过判断当前字符串中是否存在空字符`\0`来确定是否已经是字符串的结尾。因而在某些情况下，如使用空格进行分割一段字符串时，或者是图片或者视频等二进制文件中存在`\0`等，就会出问题。而`sds`不是通过空字符串来判断字符串是否已经到结尾，而是通过`len`这个字段的值。所以说，`sds`还具备**二进制安全**这个特性，即可以安全的存储具有特殊格式的二进制数据。

##### 总结

<img src="/Users/wangbing/Library/Application Support/typora-user-images/image-20210219101722867.png" alt="image-20210219101722867" style="zoom:50%;" />

### list 列表

> `Redis`中另一个常用的数据结构就是`list`，其底层有`linkedList`、`zipList`和`quickList`这三种存储方式。

#### 链表linkedList

与`Java`中的`LinkedList`类似，`Redis`中的`linkedList`是一个双向链表，也是由一个个节点组成的。`Redis`中借助`C`语言实现的链表节点结构如下所示：

```c
//定义链表节点的结构体 
typedf struct listNode{
    //前一个节点
    struct listNode *prev;
    //后一个节点
    struct listNode *next;
    //当前节点的值的指针
    void *value;
}listNode;
```

`pre`指向前一个节点，`next`指针指向后一个节点，`value`保存着当前节点对应的数据对象。`listNode`的示意图如下所示：

<img src="/Users/wangbing/Library/Application Support/typora-user-images/image-20210219102406029.png" alt="image-20210219102406029" style="zoom:50%;" />

链表的结构如下：

```c
typedf struct list{
    //头指针
    listNode *head;
    //尾指针
    listNode *tail;
    //节点拷贝函数
    void *(*dup)(void *ptr);
    //释放节点函数
    void *(*free)(void *ptr);
    //判断两个节点是否相等的函数
    int (*match)(void *ptr,void *key);
    //链表长度
    unsigned long len;
}
```

`head`指向链表的头节点，`tail`指向链表的尾节点，`dup`函数用于链表转移复制时对节点`value`拷贝的一个实现，一般情况下使用**等号**足以，但在某些特殊情况下可能会用到节点转移函数，默认可以给这个函数赋值`NULL`即表示使用等号进行节点转移。`free`函数用于释放一个节点所占用的内存空间，默认赋值`NULL`的话，即使用`Redis`自带的`zfree`函数进行内存空间释放。`match`函数是用来比较两个链表节点的`value`值是否相等，相等返回1，不等返回0。`len`表示这个链表共有多少个节点，这样就可以在`O(1)`的时间复杂度内获得链表的长度。

链表的结构如下所示：

<img src="/Users/wangbing/Library/Application Support/typora-user-images/image-20210219102529169.png" alt="image-20210219102529169" style="zoom:50%;" />

#### zipList

`Redis`的`zipList`结构如下所示：

```c
typedf struct ziplist<T>{
    //压缩列表占用字符数
    int32 zlbytes;
    //最后一个元素距离起始位置的偏移量，用于快速定位最后一个节点
    int32 zltail_offset;
    //元素个数
    int16 zllength;
    //元素内容
    T[] entries;
    //结束位 0xFF
    int8 zlend;
}ziplist
```

`zipList`的结构如下所示：

<img src="/Users/wangbing/Library/Application Support/typora-user-images/image-20210219102647469.png" alt="image-20210219102647469" style="zoom:50%;" />

注意到`zltail_offset`这个参数，有了这个参数就可以快速定位到最后一个`entry`节点的位置，然后开始倒序遍历，也就是说`zipList`支持双向遍历。

下面是`entry`的结构：

```c
typede struct entry{
    //前一个entry的长度
    int<var> prelen;
    //元素类型编码
    int<var> encoding;
    //元素内容
    optional byte[] content;
}entry
```

`prelen`保存的是前一个`entry`节点的长度，这样在倒序遍历时就可以通过这个参数定位到上一个`entry`的位置。`encoding`保存了`content`的编码类型。`content`则是保存的元素内容，它是`optional`类型的，表示这个字段是可选的。当`content`是很小的整数时，它会内联到`content`字段的尾部。`entry`结构的示意图如下所示：

<img src="/Users/wangbing/Library/Application Support/typora-user-images/image-20210219102758814.png" alt="image-20210219102758814" style="zoom:50%;" />

好了，那现在我们思考一个问题，为什么有了`linkedList`还有设计一个`zipList`呢？就像`zipList`的名字一样，它是一个压缩列表，是为了节约内存而开发的。相比于`linkedList`，其少了`pre`和`next`两个指针。在`Redis`中，`pre`和`next`指针就要占用16个字节（64位系统的一个指针就是8个字节）。另外，`linkedList`的每个节点的内存都是单独分配，加剧内存的碎片化，影响内存的管理效率。与之相对的是，`zipList`是由连续的内存组成的，这样一来，由于内存是连续的，就减少了许多内存碎片和指针的内存占用，进而节约了内存。

`zipList`遍历时，先根据`zlbytes`和`zltail_offset`定位到最后一个`entry`的位置，然后再根据最后一个`entry`里的`prelen`时确定前一个`entry`的位置。

##### 连锁更新

上面说到了，`entry`中有一个`prelen`字段，它的长度要么是1个字节，要么都是5个字节：

- 前一个节点的长度小于254个字节，则`prelen`长度为1字节；
- 前一个节点的长度大于254字节，则`prelen`长度为5字节；

假设现在有一组压缩列表，长度都在250~253字节之间，突然新增一个`entry`节点，这个`entry`节点长度大于等于254字节。由于新的`entry`节点大于等于254字节，这个`entry`节点的`prelen`为5个字节，随后会导致其余的所有`entry`节点的`prelen`增大为5字节。

<img src="/Users/wangbing/Library/Application Support/typora-user-images/image-20210219102823422.png" alt="image-20210219102823422" style="zoom:50%;" />

同样地，删除操作也会导致出现**连锁更新**这种情况，假设在某一时刻，插入一个长度大于等于254个字节的`entry`节点，同时删除其后面的一个长度小于254个字节的`entry`节点，由于小于254的`entry`节点的删除，大于等于254个字节的`entry`节点将会与后面小于254个字节的`entry`节点相连，此时就与新增一个长度大于等于254个字节的`entry`节点时的情况一样，将会发生连续更新。发生连续更新时，`Redis`需要不断地对压缩列表进行**内存分配工作**，直到结束。

#### linkedList与zipList的对比

- 当列表对象中元素的长度较小或者数量较少时，通常采用`zipList`来存储；当列表中元素的长度较大或者数量比较多的时候，则会转而使用双向链表`linkedList`来存储。
- 双向链表`linkedList`便于在表的两端进行`push`和`pop`操作，在插入节点上复杂度很低，但是它的内存开销比较大。首先，它在每个节点上除了要保存数据之外，还有额外保存两个指针；其次，双向链表的各个节点都是单独的内存块，地址不连续，容易形成内存碎片。
- `zipList`存储在一块连续的内存上，所以存储效率很高。但是它不利于修改操作，插入和删除操作需要频繁地申请和释放内存。特别是当`zipList`长度很长时，一次`realloc`可能会导致大量的数据拷贝。

#### quickList

在`Redis`3.2版本之后，`list`的底层实现方式又多了一种，`quickList`。`qucikList`是由`zipList`和双向链表`linkedList`组成的混合体。它将`linkedList`按段切分，每一段使用`zipList`来紧凑存储，多个`zipList`之间使用双向指针串接起来。示意图如下所示：

<img src="/Users/wangbing/Library/Application Support/typora-user-images/image-20210219102919814.png" alt="image-20210219102919814" style="zoom:50%;" />



节点`quickListNode`的定义如下：

```c
typedf struct quicklistNode{
    //前一个节点
    quicklistNode* prev;
    //后一个节点
    quicklistNode* next;
    //压缩列表
    ziplist* zl;	
    //ziplist大小
    int32 size;		
    //ziplist 中元素数量
    int16 count;
    //编码形式 存储 ziplist 还是进行 LZF 压缩储存的zipList
    int2 encoding;			
    ...
}quickListNode
```

`quickList`的定义如下所示：

```c
typedf struct quicklist{
    //指向头结点
    quicklistNode* head;
    //指向尾节点
    quicklistNode* tail;
    //元素总数
    long count;
    //quicklistNode节点的个数
    int nodes;	
    //压缩算法深度
    int compressDepth;		
    ...
}quickList
```

上述代码简单地表示了`quickList`的大致结构，为了进一步节约空间，`Redis`还会对`zipList`进行压缩存储，使用**LZF**算法进行压缩，可以选择压缩深度。

#### 每个zipList可以存储多少个元素

想要了解这个问题，就得打开`redis.conf`文件了。在`DVANCED CONFIG`下面有着清晰的记载。

```
# Lists are also encoded in a special way to save a lot of space.
# The number of entries allowed per internal list node can be specified
# as a fixed maximum size or a maximum number of elements.
# For a fixed maximum size, use -5 through -1, meaning:
# -5: max size: 64 Kb  <-- not recommended for normal workloads
# -4: max size: 32 Kb  <-- not recommended
# -3: max size: 16 Kb  <-- probably not recommended
# -2: max size: 8 Kb   <-- good
# -1: max size: 4 Kb   <-- good
# Positive numbers mean store up to _exactly_ that number of elements
# per list node.
# The highest performing option is usually -2 (8 Kb size) or -1 (4 Kb size),
# but if your use case is unique, adjust the settings as necessary.
list-max-ziplist-size -2
```

`quickList`内部默认单个`zipList`长度为**8k**字节，即`list-max-ziplist-size`的值设置为**-2**，超出了这个阈值，就会重新生成一个`zipList`来存储数据。根据注释可知，性能最好的时候就是就是`list-max-ziplist-size`为**-1**和**-2**，即分别是**4kb和8kb**的时候，当然，这个值也可以被设置为正数，当`list-max-ziplist-szie`为**正数n**时，表示每个`quickList`节点上的`zipList`最多包含**n个**数据项。

#### 压缩深度

上面提到过，`quickList`中可以使用压缩算法对`zipList`进行进一步的压缩，这个算法就是**[LZF算法](https://blog.csdn.net/u012319493/article/details/83653860)**，这是一种无损压缩算法，具体可以参考这里。使用压缩算法对`zipList`进行压缩后，`zipList`的结构如下所示：

```c
typedf struct ziplist_compressed{
    //元素个数
    int32 size;
    //元素内容
    byte[] compressed_data
}
```

此时`quickList`的示意图如下所示：

<img src="/Users/wangbing/Library/Application Support/typora-user-images/image-20210219102946590.png" alt="image-20210219102946590" style="zoom:50%;" />

当然，在`redis.conf`文件中的`DVANCED CONFIG`下面也可以对压缩深度进行配置。

```
# Lists may also be compressed.
# Compress depth is the number of quicklist ziplist nodes from *each* side of
# the list to *exclude* from compression.  The head and tail of the list
# are always uncompressed for fast push/pop operations.  Settings are:
# 0: disable all list compression
# 1: depth 1 means "don't start compressing until after 1 node into the list,
#    going from either the head or tail"
#    So: [head]->node->node->...->node->[tail]
#    [head], [tail] will always be uncompressed; inner nodes will compress.
# 2: [head]->[next]->node->node->...->node->[prev]->[tail]
#    2 here means: don't compress head or head->next or tail->prev or tail,
#    but compress all nodes between them.
# 3: [head]->[next]->[next]->node->node->...->node->[prev]->[prev]->[tail]
# etc.
list-compress-depth 0
```

`list-compress-depth`这个参数表示**一个`quickList`两端不被压缩的节点个数。**需要注意的是，这里的节点个数是指`quicklist`双向链表的节点个数，而不是指`ziplis`t里面的数据项个数。实际上，一个`quicklist`节点上的`ziplist`，如果被压缩，就是整体被压缩的。

- `quickList`默认的压缩深度为**0**，也就是不开启压缩
- 当`list-compress-depth`为1，表示`quickList`的两端各有1个节点不进行压缩，中间结点进行压缩；
- 当`list-compress-depth`为2，表示`quickList`的首尾2个节点不进行压缩，中间结点进行压缩；
- 以此类推

从上面可以看出，对于`quickList`来说，其首尾两个节点永远不会被压缩。

#### 总结 

<img src="/Users/wangbing/Documents/reader/1593641-20200831230342556-333372313.png" alt="1593641-20200831230342556-333372313" style="zoom:75%;" />

### hash 字典

hash 是 Redis 中的字典结构，内部实现与 Java 中的 HashMap 一致（数组 + 链表）。字典内部有两个 HashTable，Rehash（扩容或缩容）时用于存储新旧两份数据。

<img src="/Users/wangbing/Library/Application Support/typora-user-images/image-20210209123213441.png" alt="image-20210209123213441" style="zoom: 43%;" />

扩容时机：元素个数等于数组长度（bgsave 时 5 倍数组长度时扩容）
扩容方式：2 倍数组长度
缩容时机：元素个数少于数组长度 10%

> `hash`是日常开发过程中使用`Redis`的一个数据结构，其底层实现方式有两种，如下所示。一种是`zipList`，这种是当`hash`结构的`V`值较小的时候使用的编码方式。这个已经介绍过了。这篇文章主要讲解一下另外一种实现方式，字典`dict`，当`hash`结构的`V`值较大时采用的编码方式。

这里又要开始鞭尸`C`语言了，字典`dict`作为一种常用的数据结构，`C`语言内部并不具备，因而`Redis`的开发人员自己设计和开发了`Redis`中的`dict`结构，其定义如下：

```c
typedf struct dict{
    dictType *type;//类型特定函数，包括一些自定义函数，这些函数使得key和
                   //value能够存储
    void *private;//私有数据
    dictht ht[2];//两张hash表 
    int rehashidx;//rehash索引，字典没有进行rehash时，此值为-1
    unsigned long iterators; //正在迭代的迭代器数量
}dict;
```

- `type`和`private`这两个属性是为了实现字典多态而设置的，当字典中存放着不同类型的值，对应的一些复制，比较函数也不一样，这两个属性配合起来可以实现多态的方法调用；
- `ht[2]`，两个`hash`表
- `rehashidx`，这是一个辅助变量，用于记录`rehash`过程的进度，以及是否正在进行`rehash`等信息，当此值为**-1**时，表示该`dict`此时没有`rehash`过程
- `iterators`，记录此时`dict`有几个迭代器正在进行遍历过程

#### dictht

由上面可以看出，`dict`本质上是对哈希表`dictht`的一个简单封装，`dictht`的定义如下所示：

```c
typedf struct dictht{
    dictEntry **table;//存储数据的数组 二维
    unsigned long size;//数组的大小
    unsigned long sizemask;//哈希表的大小的掩码，用于计算索引值，总是等于 
                           //size-1
    unsigned long used;//// 哈希表中中元素个数
}dictht;
```

`**table`是一个`dictEntry`类型的数组，用于真正存储数据；`size`表示`**table`这个数组的大小；`sizemask`用于计算索引位置，且总是等于`size-1`；`used`表示`dictht`中已有的节点数量，其示意图如下所示：

<img src="/Users/wangbing/Library/Application Support/typora-user-images/image-20210219104431831.png" alt="image-20210219104431831" style="zoom:50%;" />

#### dictEntry

上面分析`dictht`时说到，真正存储数据的结构是`dictEntry`数组，其结构定义如下：

```c
typedf struct dictEntry{
    void *key;//键
    union{
        void val;
        unit64_t u64;
        int64_t s64;
        double d;
    }v;//值
    struct dictEntry *next；//指向下一个节点的指针
}dictEntry;
```

#### 扩容与缩容

当哈希表中元素数量逐渐增加时，此时产生`hash冲突`的概率逐渐增大，且由于`dict`也是采用**拉链法**解决`hash冲突`的，随着`hash冲突`概率上升，链表会越来越长，这就会导致查找效率下降。相反，当元素不断减少时，元素占用`dict`的空间就越少，出于对内存的极致利用，此时就需要进行缩容操作。

既然说到扩容和缩容，熟悉`Java`集合的小伙伴是不是想到了什么。不错，那就是**负载因子**。**负载因子一般用于描述集合当前被填充的程度**。在`Redis`的字典`dict`中，**负责因子=哈希表中已保存节点数量/哈希表的大小**，即：

```
load factor = ht[0].used / ht[0].size
```

`Redis`中，三条关于扩容和缩容的规则：

- 没有执行BGSAVE和BGREWRITEAOF指令的情况下，哈希表的负载因子大于等于1时进行扩容；
- 正在执行BGSAVE和BGREWRITEAOF指令的情况下，哈希表的负载因大于等于5时进行扩容；
- 负载因子小于0.1时，`Redis`自动开始对哈希表进行收缩操作；

其中，扩容和缩容的数量大小也有一定的规则：

- 扩容：**扩容后的`dictEntry`数组数量为第一个大于等于`ht[0].used \* 2`的`2^n`**；
- 缩容：**缩容后的`dictEntry`数组数量为第一个大于等于`ht[0].used`的`2^n`**；

#### rehash

与`Java`中的`HashMap`类似，当`Redis`中的`dict`进行扩容或者缩容，会发生`reHash`过程。`Java`中`HashMap`的`rehash`过程如下：新建一个哈希表，一次性将当前所有节点进行`rehash`然后复制到新哈希表相应的位置上，之后释放掉原有的`hash`表，而持有新的表，这个过程是一个时间复杂度为`O(n)`的操作。而对于单线程的`Redis`而言很难承受这么高时间复杂度的操作，因而其`rehash`的过程有所不同，使用的是一种称之为**渐进式rehash**的方式，一点一点地进行搬迁。其过程如下：

- 假设当前数据在`dictht[0]`中，那么首先为`dictht[1]`分配足够的空间，如果是扩容，则`dictht[1]`大小就按照扩容规则设置；如果是缩减，则`dictht[1]`大小就按照缩减规则进行设置；
- 在字典`dict`中维护一个变量，**rehashidx=0**，表示`rehash`正式开始；
- `rehash`进行期间，每次对字典执行添加、删除、查找或者更新操作时，程序除了执行指定的操作以外，还会顺带将`dictht[0]`哈希表在`rehashidx`索引上的所有键值对`rehash`到`dictht[1]`，当一次`rehash`工作完成之后，程序将`rehashidx`属性的值+1。同时在`serverCron`中调用`rehash`相关函数，在1`ms`的时间内，进行`rehash`处理，每次仅处理少量的转移任务(100个元素)；
- 随着字典操作的不断执行，最终在某个时间点上，`dictht[0]`的所有键值对都会被`rehash`至`dictht[1]`，这时程序将`rehashidx`属性的值设为-1，表示rehash操作已完成；

上述就是`Redis`中`dict`的**渐进式rehash**过程，但在这个过程会存在两个明显问题。第一，第三步说了，每次对字典执行增删改查时才会触发`rehash`过程，万一某一时间段并没有任何请求命令呢？此时应该怎么办？第二，在维护两个`dictht`的时候，此时哈希表如何正常对外提供服务？

`Redis`的设计人员在设计时就已经考虑到了这两个问题。对于第一个问题，`Redis`在有一个**定时器，会定时去判断`rehash`是否完成**，如果没有完成，则继续进行`rehash`。定时函数如下所示：

```c
// 服务器定时任务
void databaseCron() {
         ...
         if (server.activerehashing) {
            for (j = 0; j < dbs_per_call; j++) {
                 int work_done = incrementallyRehash(rehash_db);//rehash方法
                 if (work_done) {
                       /* If the function did some work, stop here, we'll do
                        * more at the next cron loop. */
                       break;
                  } else {
                 /* If this db didn't need rehash, we'll try the next one. */
                      rehash_db++;
                      rehash_db %= server.dbnum;
                  }
             }
        }
}
```

对于第二个问题，对于**添加操作**，会将新的数据直接添加到`dictht[1]`上面，这样就可以保证`dictht[0]`上的数量只减少不增加。而对于**删除、更改、查询操作**，会直接在`dictht[0]`上进行，尤其是这三个操作，都会涉及到查询，**当在`dictht[0]`上查询不到时，会接着去`dictht[1]`上查找**，如果再找不到，则表明不存在该`K-V`值。

#### 渐进式rehash的优缺点

**优点**：采用了**分而治之**的思想，将 **`rehash` 操作分散到每一个对该哈希表的操作上以及定时函数上**，避免了集中式`rehash` 带来的性能压力；

**缺点**：在 rehash 的时间内，需要保存两个 hash 表，对内存的占用稍大，而且如果在 redis 服务器本来内存满了的时候，**突然进行 rehash 会造成大量的 key 被抛弃**；

#### 思考题

为什么扩容的时候要考虑`BIGSAVE`的影响，而缩容时不需要？

- `BIGSAVE`时，`dict`要是进行扩容，则此时就需要为`dictht[1]`分配内存，若是`dictht[0]`的数据量很大时，就会占用更多系统内存，造成内存页过多分离，所以为了避免系统耗费更多的开销去回收内存，此时最好不要进行扩容；
- 缩容时，结合缩容的条件，此时负载因子<0.1，说明此时`dict`中数据很少，就算为`dictht[1]`分配内存，也消耗不了多少资源；

<img src="/Users/wangbing/Library/Application Support/typora-user-images/image-20210219104551185.png" alt="image-20210219104551185" style="zoom:50%;" />

### set 集合

set 是没有排序的字符串集合，不允许出现重复元素，内部结构与 hash 字典一致，只是 value 为 null。

> 与`Java`中的`HashSet`一样，无序且存储元素不重复。其底层有两种实现方式，当`value`是整数值时，且数据量不大时使用`inset`来存储，其他情况都是用字典`dict`来存储。

#### intset

`Redis`中`inset`的结构定义如下所示：

```c
typedf struct intset{
    uint32_t encoding;//编码方式 有三种 默认 INSET_ENC_INT16
    uint32_t length;//集合元素个数
    int8_t contents[];//实际存储元素的数组 
                      //元素类型并不一定是ini8_t类型，柔性数组不占intset结构体大小，并且数组中的元
                      //素从小到大排列
}inset;
#define INTSET_ENC_INT16 (sizeof(int16_t))   //16位，2个字节，表示范围-32,768~32,767
#define INTSET_ENC_INT32 (sizeof(int32_t))   //32位，4个字节，表示范
                                             //围-2,147,483,648~2,147,483,647
#define INTSET_ENC_INT64 (sizeof(int64_t))   //64位，8个字节，表示范
//围-9,223,372,036,854,775,808~9,223,372,036,854,775,807
```

**编码格式`encoding`**：共有三种，`INTSET_ENC_INT16`、`INSET_ENC_INT32`和`INSET_ENC_INT64`三种，分别对应不同的范围。`Redis`为了尽可能地节省内存，会根据插入数据的大小选择不一样的类型来进行存储。

**元素数量`length`**：记录了保存数据的数组`contents`中共有多少个元素，这样获取个数的时间复杂度就是`O(1)`。

**数组`contents`**：真正存储数据的地方，数组是按照**从小到大**有序排列的，并且**不包含任何重复项**。

`intset`的示意图如下所示：

<img src="/Users/wangbing/Library/Application Support/typora-user-images/image-20210219104826391.png" alt="image-20210219104826391" style="zoom:50%;" />

#### intset中整数的升级过程

这个过程可以参考这位小姐姐写的[文章](https://juejin.im/post/5eec5a8fe51d45742944f0f4#heading-7)，配图食用，效果更加。整体流程总结如下：

- **了解旧的存储格式**，计算出目前已有元素占用内存大小，计算规则是**length \* encoding**，如 4* 16=64；
- **确定新的编码格式**，当原有的编码格式不能存储下新增的数据时，此时就要选择新的合适的编码格式；
- **根据新的编码格式计算出需要新增的内存大小**，然后从尾部将数据插入；
- **根据新的编码格式重置之前的值**，此时`contents`存在两种编码格式设置的值，就需要进行统一，从插入新数据的起始位置开始，从后向前将之前的数据按照新的编码格式进行移动和设置。**从后往前是为了防止数据被覆盖**。

**优点**：根据存入的数据大小选择合适的编码方式，且只在必要的时候进行升级操作，节省内存

**缺点**：升级过程耗费系统资源，还有就是**不支持降级**，一旦升级就不可以降级

#### 总结

<img src="/Users/wangbing/Library/Application Support/typora-user-images/image-20210219104914857.png" alt="image-20210219104914857" style="zoom:50%;" />

### zset 有序集合

zset 是有序集合，内部实现为跳跃链表，用于支持随机地插入和删除。

> set`是`Redis`提供的一个非常特别的数据结构，常用作排行榜等功能，以用户`id`为`value`，关注时间或者分数作为`score`进行排序。与其他数据结构相似，`zset`也有两种不同的实现，分别是`zipList`和`skipList

节点更新的过程是先删除再添加。

具体使用哪种结构进行存储，规则如下：

- ```
  zipList：满足以下两个条件
  ```

  - `[score,value]`键值对数量少于128个；
  - 每个元素的长度小于64字节；

- ```
  skipList：不满足以上两个条件时使用跳表、组合了hash和skipList
  ```

  - `hash`用来存储`value`到`score`的映射，这样就可以在`O(1)`时间内找到`value`对应的分数；
  - `skipList`按照**从小到大**的顺序存储分数
  - `skipList`每个元素的值都是`[socre,value]`对

使用`zipList`的示意图如下所示：

<img src="/Users/wangbing/Library/Application Support/typora-user-images/image-20210219105157307.png" alt="image-20210219105157307" style="zoom:50%;" />

使用跳表时的示意图：

<img src="/Users/wangbing/Library/Application Support/typora-user-images/image-20210219105216377.png" alt="image-20210219105216377" style="zoom:50%;" />

#### 跳表 skipList

跳表`skipList`在`Redis`中的运用场景只有一个，那就是作为有序列表`zset`的底层实现。跳表可以保证增、删、查等操作时的时间复杂度为`O(logN)`，这个性能可以与平衡树相媲美，但实现方式上却更加简单，唯一美中不足的就是跳表占用的空间比较大，其实就是一种**空间换时间**的思想。跳表的结构如下所示：

<img src="/Users/wangbing/Library/Application Support/typora-user-images/image-20210219105330617.png" alt="image-20210219105330617" style="zoom:50%;" />

`Redis`中跳表一个节点最高可以达到**64**层，一个跳表中最多可以存储**2^64**个元素。跳表中，每个节点都是一个`skiplistNode`，

每个跳表的节点也都会维护着一个`score`值，这个值在跳表中是按照**从小到大**的顺序排列好的。

跳表的结构定义如下所示：

```c
typedf struct zskiplist{
    //头节点
    struct zskiplistNode *header;
    //尾节点
    struct zskiplistNode *tail;
    // 跳表中元素个数
    unsigned long length;
    //目前表内节点的最大层数
    int level;
}zskiplist;
```

**header**：指向跳表的头节点，通过这个指针可以直接找到表头，时间复杂度为`O(1)`；

**tail**：指向跳表的尾节点，通过这个指针可以直接找到表尾，时间复杂度为`o(1)`；

**length**：记录跳表的长度，即不包括头节点，整个跳表中有多少个元素；

**level**：记录当前跳表内，所有节点中层数最大的`level`；

`zskiplist`的示意图如下所示：

<img src="/Users/wangbing/Library/Application Support/typora-user-images/image-20210219105353536.png" alt="image-20210219105353536" style="zoom:50%;" />

`zskiplistNode`的结构定义如下：

```c
typedf struct zskiplistNode{
    sds ele;// 具体的数据
    double score;// 分数
    struct zskiplistNode *backward;//后退指针
    struct zskiplistLevel{  
        struct zskiplistNode *forward;//前进指针forward
        unsigned int span;//跨度span
    }level[];//层级数组 最大32
}zskiplistNode;
```

**ele**：真正的数据，每个节点的数据都是唯一的，但节点的分数`score`可以是一样的。两个相同分数`score`的节点是**按照元素的字典序进行排列**的；

**score**：各个节点中的数字是节点所保存的分数`score`，在跳表中，节点按照各自所保存的分数从小到大排列；

**backward**：用于从表尾向表头遍历，每个节点只有一个后退指针，即每次只能后退一步；

**层级数组**：这个数组中的每个节点都有两个属性，`forward`指向下一个节点，`span`跨度用来计算当前节点在跳表中的一个排名，这就为`zset`提供了一个**查看排名**的方法。数组中的每个节点中用1、2、3等字样标记节点的各个层，`L1`代表第一层，`L2`代表第二层，`L3`代表第三层；，以此类推；

`skiplistNode`的示意图如下所示：

<img src="/Users/wangbing/Library/Application Support/typora-user-images/image-20210219105412355.png" alt="image-20210219105412355" style="zoom:50%;" />

#### 增删改查

以下图为例，讲解一下`skiplist`的增删改查过程。

<img src="/Users/wangbing/Library/Application Support/typora-user-images/image-20210219105442601.png" alt="image-20210219105442601" style="zoom:50%;" />

#### 查

假设现在要查找7这个节点，步骤如下：

- 从`head`开始遍历，指针指向4这个节点，由于4<7，且同层的下一个指针指向`NULL`，所以下级一层；
- 跳到6节点所在的层，同理，6<7，且同层的下一个指针指向`NULL`，再下降一层；
- 此时到了第一层，第一层是一个双向链表，由于6<7，所以开始向后遍历，查找到7就返回，不然就返回`NULL`；

#### 删

删除的过程前期与查找相似，先定位到元素所在的位置，再进行删除，最后更新一下指针、更新一下最高的层数。

#### 改

先是判断这个 **value** 是否存在，如果存在就是更新的过程，如果不存在就是插入过程。在更新的过程是，如果找到了Value，先删除掉，再新增，这样的弊端是会做两次的搜索，在性能上来讲就比较慢了，在 **Redis 5.0** 版本中，**Redis** 的作者 **Antirez** 优化了这个更新的过程，目前的更新过程是如果判断这个 **value**是否存在，如果存在的话就直接更新，然后再调整整个跳跃表的 score 排序，这样就不需要两次的搜索过程。

#### 增

比如要插入的值为 6

- 从 head 节点开始，先是在 head 开始降层来查找到最后一个比 6 小的节点；
- 等到查到最后一个比 6 小的节点的时候(假设为 5 )；
- 然后需要引入一个**随机层数算法**来为这个节点随机地建立层数；
- 把这个节点插入进去以后，同时更新一遍最高的层数即可；

#### 随机层数算法

```c
int zslRandomLevel(void) {
    int level = 1;
    while ((random()&0xFFFF) < (ZSKIPLIST_P * 0xFFFF))
        level += 1;
    return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
}
 #define ZSKIPLIST_MAXLEVEL 32
#define ZSKIPLIST_P 0.25
```

#### 总结

<img src="/Users/wangbing/Library/Application Support/typora-user-images/image-20210219105526987.png" alt="image-20210219105526987" style="zoom:50%;" />

## 3、Redis 内存淘汰机制了解么？

> 相关问题：MySQL 里有 2000w 数据，Redis 中只存 20w 的数据，如何保证 Redis 中的数据都是热点数据?

#### Redis是如何判断数据是否过期的呢？

Redis 通过一个叫做过期字典（可以看作是hash表）来保存数据过期的时间。过期字典的键指向Redis数据库中的某个key(键)，过期字典的值是一个long long类型的整数，这个整数保存了key所指向的数据库键的过期时间（毫秒精度的UNIX时间戳）。

<img src="/Users/wangbing/Library/Application Support/typora-user-images/image-20210209124915510.png" alt="image-20210209124915510" style="zoom:40%;" />

过期字典是存储在redisDb这个结构里的：

```c
typedef struct redisDb {
    ...
    
    dict *dict;     //数据库键空间,保存着数据库中所有键值对
    dict *expires   // 过期字典,保存着键的过期时间
    ...
} redisDb;
```

#### Redis 提供 6 种数据淘汰策略：

1. **volatile-lru（least recently used）**：从已设置过期时间的数据集（server.db[i].expires）中挑选最近最少使用的数据淘汰
2. **volatile-ttl**：从已设置过期时间的数据集（server.db[i].expires）中挑选将要过期的数据淘汰
3. **volatile-random**：从已设置过期时间的数据集（server.db[i].expires）中任意选择数据淘汰
4. **allkeys-lru（least recently used）**：当内存不足以容纳新写入数据时，在键空间中，移除最近最少使用的 key（这个是最常用的）
5. **allkeys-random**：从数据集（server.db[i].dict）中任意选择数据淘汰
6. **no-eviction**：禁止驱逐数据，也就是说当内存不足以容纳新写入数据时，新写入操作会报错。这个应该没人使用吧！

4.0 版本后增加以下两种：

1. **volatile-lfu（least frequently used）**：从已设置过期时间的数据集(server.db[i].expires)中挑选最不经常使用的数据淘汰
2. **allkeys-lfu（least frequently used）**：当内存不足以容纳新写入数据时，在键空间中，移除最不经常使用的 key

## 4、Redis 持久化机制(怎么保证 Redis 挂掉之后再重启数据可以进行恢复

很多时候我们需要持久化数据也就是将内存中的数据写入到硬盘里面，大部分原因是为了之后重用数据（比如重启机器、机器故障之后恢复数据），或者是为了防止系统故障而将数据备份到一个远程位置。

Redis 不同于 Memcached 的很重要一点就是，Redis 支持持久化，而且支持两种不同的持久化操作。**Redis 的一种持久化方式叫快照（snapshotting，RDB），另一种方式是只追加文件（append-only file, AOF）**。这两种方法各有千秋，下面我会详细这两种持久化方法是什么，怎么用，如何选择适合自己的持久化方法。

**快照（snapshotting）持久化（RDB）**

Redis 可以通过创建快照来获得存储在内存里面的数据在某个时间点上的副本。Redis 创建快照之后，可以对快照进行备份，可以将快照复制到其他服务器从而创建具有相同数据的服务器副本（Redis 主从结构，主要用来提高 Redis 性能），还可以将快照留在原地以便重启服务器的时候使用。

快照持久化是 Redis 默认采用的持久化方式，在 Redis.conf 配置文件中默认有此下配置：

```conf
save 900 1     #在900秒(15分钟)之后，如果至少有1个key发生变化，Redis就会自动触发BGSAVE命令创建快照。

save 300 10    #在300秒(5分钟)之后，如果至少有10个key发生变化，Redis就会自动触发BGSAVE命令创建快照。

save 60 10000  #在60秒(1分钟)之后，如果至少有10000个key发生变化，Redis就会自动触发BGSAVE命令创建快照。
```

**AOF（append-only file）持久化**

与快照持久化相比，AOF 持久化 的实时性更好，因此已成为主流的持久化方案。默认情况下 Redis 没有开启 AOF（append only file）方式的持久化，可以通过 appendonly 参数开启：

```conf
appendonly yes
```

开启 AOF 持久化后每执行一条会更改 Redis 中的数据的命令，Redis 就会将该命令写入硬盘中的 AOF 文件。AOF 文件的保存位置和 RDB 文件的位置相同，都是通过 dir 参数设置的，默认的文件名是 appendonly.aof。

在 Redis 的配置文件中存在三种不同的 AOF 持久化方式，它们分别是：

```conf
appendfsync always    #每次有数据修改发生时都会写入AOF文件,这样会严重降低Redis的速度
appendfsync everysec  #每秒钟同步一次，显示地将多个写命令同步到硬盘
appendfsync no        #让操作系统决定何时进行同步
```

为了兼顾数据和写入性能，用户可以考虑 appendfsync everysec 选项 ，让 Redis 每秒同步一次 AOF 文件，Redis 性能几乎没受到任何影响。而且这样即使出现系统崩溃，用户最多只会丢失一秒之内产生的数据。当硬盘忙于执行写入操作的时候，Redis 还会优雅的放慢自己的速度以便适应硬盘的最大写入速度。

**拓展：Redis 4.0 对于持久化机制的优化**

Redis 4.0 开始支持 RDB 和 AOF 的混合持久化（默认关闭，可以通过配置项 `aof-use-rdb-preamble` 开启）。

如果把混合持久化打开，AOF 重写的时候就直接把 RDB 的内容写到 AOF 文件开头。这样做的好处是可以结合 RDB 和 AOF 的优点, 快速加载同时避免丢失过多的数据。当然缺点也是有的， AOF 里面的 RDB 部分是压缩格式不再是 AOF 格式，可读性较差。

**补充内容：AOF 重写**

AOF 重写可以产生一个新的 AOF 文件，这个新的 AOF 文件和原有的 AOF 文件所保存的数据库状态一样，但体积更小。

AOF 重写是一个有歧义的名字，该功能是通过读取数据库中的键值对来实现的，程序无须对现有 AOF 文件进行任何读入、分析或者写入操作。

在执行 BGREWRITEAOF 命令时，Redis 服务器会维护一个 AOF 重写缓冲区，该缓冲区会在子进程创建新 AOF 文件期间，记录服务器执行的所有写命令。当子进程完成创建新 AOF 文件的工作之后，服务器会将重写缓冲区中的所有内容追加到新 AOF 文件的末尾，使得新旧两个 AOF 文件所保存的数据库状态一致。最后，服务器用新的 AOF 文件替换旧的 AOF 文件，以此来完成 AOF 文件重写操作

## 5、Redis 事务

Redis 可以通过 **MULTI，EXEC，DISCARD 和 WATCH** 等命令来实现事务(transaction)功能。

```basic
> MULTI
OK
> INCR foo
QUEUED
> INCR bar
QUEUED
> EXEC
1) (integer) 1
2) (integer) 1
```

使用 [MULTI](https://redis.io/commands/multi)命令后可以输入多个命令。Redis不会立即执行这些命令，而是将它们放到队列，当调用了[EXEC](https://redis.io/commands/exec)命令将执行所有命令。

但是，Redis 的事务和我们平时理解的关系型数据库的事务不同。我们知道事务具有四大特性： **1. 原子性**，**2. 隔离性**，**3. 持久性**，**4. 一致性**。

1. **原子性（Atomicity）：** 事务是最小的执行单位，不允许分割。事务的原子性确保动作要么全部完成，要么完全不起作用；
2. **隔离性（Isolation）：** 并发访问数据库时，一个用户的事务不被其他事务所干扰，各并发事务之间数据库是独立的；
3. **持久性（Durability）：** 一个事务被提交之后。它对数据库中数据的改变是持久的，即使数据库发生故障也不应该对其有任何影响。
4. **一致性（Consistency）：** 执行事务前后，数据保持一致，多个事务对同一个数据读取的结果是相同的；

**Redis 是不支持 roll back 的，因而不满足原子性的（而且不满足持久性）。**

Redis官网也解释了自己为啥不支持回滚。简单来说就是Redis开发者们觉得没必要支持回滚，这样更简单便捷并且性能更好。Redis开发者觉得即使命令执行错误也应该在开发过程中就被发现而不是生产过程中。

你可以将Redis中的事务就理解为 ：**Redis事务提供了一种将多个命令请求打包的功能。然后，再按顺序执行打包的所有命令，并且不会被中途打断。**

## 6、如何保证缓存和数据库数据的一致性？

细说的话可以扯很多，但是我觉得其实没太大必要（小声BB：很多解决方案我也没太弄明白）。我个人觉得引入缓存之后，如果为了短时间的不一致性问题，选择让系统设计变得更加复杂的话，完全没必要。

下面单独对 **Cache Aside Pattern（旁路缓存模式）** 来聊聊。

Cache Aside Pattern 中遇到写请求是这样的：更新 DB，然后直接删除 cache 。

如果更新数据库成功，而删除缓存这一步失败的情况的话，简单说两个解决方案：

1. **缓存失效时间变短（不推荐，治标不治本）** ：我们让缓存数据的过期时间变短，这样的话缓存就会从数据库中加载数据。另外，这种解决办法对于先操作缓存后操作数据库的场景不适用。
2. **增加cache更新重试机制（常用）**： 如果 cache 服务当前不可用导致缓存删除失败的话，我们就隔一段时间进行重试，重试次数可以自己定。如果多次重试还是失败的话，我们可以把当前更新失败的 key 存入队列中，等缓存服务可用之后，再将缓存中对应的 key 删除即可。
