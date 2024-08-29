# tinyredis

## 协议

`nstr`字符串的数量， `len`是后续字符串的长度。两者都是 32 位整数。

```
+------+-----+------+-----+------+-----+-----+------+
| nstr | len | str1 | len | str2 | ... | len | strn |
+------+-----+------+-----+------+-----+-----+------+
```

4字节的小端整数（表示请求长度）+ 可变长度的请求 

```
+-----+------+-----+------+--------
| len | msg1 | len | msg2 | more...
+-----+------+-----+------+--------
```

响应是一个 32 位状态代码，后跟响应字符串。

```
+-----+---------+
| res | data... |
+-----+---------+
```

## Conn

读写缓冲，state决定如何处理连接  `STATE_REQ`用于读取请求， `STATE_RES`用于发送响应

计时器，用于清除空闲的TCP连接

## 事件循环

poll



# Hashtables

链地址法处理冲突

using Intrusive Data Structures   

结构独立于数据   

通过偏移节点指针来找到数据 通过宏来完成container_of

```c++
Node *my_node = some_lookup_function();
MyData *my_data = container_of(my_node, MyData, node);
//                             ^ptr     ^type   ^member
```

`container_of`通常定义为：

```c++
#define container_of(ptr, T, member) ({                  \
    const typeof( ((T *)0)->member ) *__mptr = (ptr);    \
    (T *)( (char *)__mptr - offsetof(T, member) );})
```

- 第一行检查是否与 GCC 扩展`ptr`匹配 。`T::member``typeof`也可用decltype

- 第二行偏移`ptr`从`T::member`到 `T`。

  忽略类型检查可简化为：

```c++
#define container_of(ptr, T, member) \
    (T *)( (char *)ptr - offsetof(T, member) )
```



```C++
struct HNode {
    HNode *next = NULL;
    uint64_t hcode = 0;
};
struct HTab {
    HNode **tab = NULL; 
    size_t mask = 0;    // 2^n - 1
    size_t size = 0;
};
```

使用 2 的幂来表示增长，使用位掩码代替慢速模数运算符，因为对 2 的幂取模与获取低位相同。

```c++
struct HMap {
    HTab ht1;   // newer
    HTab ht2;   // older
    size_t resizing_pos = 0;
};
```

使用的真正的哈希表包含 2 个固定大小的表，用于逐步调整大小

resizing_pos用于记录上次迁移位置，由于每次迁移数据量不宜过大

```c++
struct Entry {
    struct HNode node;
    std::string key;
    std::string val;
};
static struct {
    HMap db;
    std::vector<Conn *> fd2conn;//map of all client connections, keyed by fd
    DList idle_list;    //timers for idle connections
    std::vector<HeapItem> heap;	  // timers for TTLs
    TheadPool tp;   // the thread pool
} g_data;
```

## 数据序列化

```c++
enum {
    SER_NIL = 0,    // Like `NULL`
    SER_ERR = 1,    // An error code and message
    SER_STR = 2,    // A string
    SER_INT = 3,    // A int64
    SER_ARR = 4,    // Array
};
```

包含五种类型

序列化方案可以概括为 “type-length-value”  (TLV)：“T”表示值的类型；“L”适用于字符串或数组等可变长度数据；“V”是最后编码的内容。

## AVL树

针对有序集合

```c++
struct AVLNode {
    uint32_t depth = 0;     // subtree height
    uint32_t cnt = 0;       // subtree size
    AVLNode *left = NULL;
    AVLNode *right = NULL;
    AVLNode *parent = NULL;
};
struct Data {
    AVLNode node;
    uint32_t val = 0;
};

struct Container {
    AVLNode *root = NULL;
};
```

cnt存储子树大小，用于rank query

##### 有序集合数据类型

```c++
struct ZSet {
    AVLNode *tree = NULL;
    HMap hmap;
};

struct ZNode {
    AVLNode tree;   
    HNode hmap;    
    double score = 0;
    size_t len = 0;
    char name[0];  // variable length
};
struct Entry {
    struct HNode node;
    std::string key;
    uint32_t type = 0;
    std::string val;   
    ZSet *zset = NULL;  
};
```

## 计时器

##### 链表实现 

用于清除空闲的TCP连接

```c++
struct DList {
    DList *prev = NULL;
    DList *next = NULL;
};
```

添加到服务器和连接结构

```c++
static struct {
    HMap db; 
    std::vector<Conn *> fd2conn;
    DList idle_list;
} g_data;

```

##### 堆实现

对时间戳排序，用于设置TTL

```c++
struct HeapItem {
    uint64_t val = 0;
    size_t *ref = NULL;
};

struct Entry {
    struct HNode node;
    std::string key;
    std::string val;
    uint32_t type = 0;
    ZSet *zset = NULL;
    // for TTLs
    size_t heap_idx = -1;
};
```

## 线程池

生产者-消费者模型

```c++
struct Work {
    void (*f)(void *) = NULL;
    void *arg = NULL;
};

struct TheadPool {
    std::vector<pthread_t> threads;
    std::deque<Work> queue;
    pthread_mutex_t mu;
    pthread_cond_t not_empty;
};
```