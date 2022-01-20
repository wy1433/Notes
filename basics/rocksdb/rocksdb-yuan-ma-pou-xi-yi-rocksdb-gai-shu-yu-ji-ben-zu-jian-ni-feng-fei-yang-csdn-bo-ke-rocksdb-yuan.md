# Rocksdb源码剖析一----Rocksdb概述与基本组件\_逆风飞扬-CSDN博客\_rocksdb源码分析

另外，rocksdb同样可以使用tcmalloc与jemalloc，在性能方面还是会有不小的提升.

3.4memtable

leveldb中，memtable在内存中核心s的数据结构为skiplist，而在rocksdb中,memtable在内存中的形式有三种：skiplist、hash-skiplist、hash-linklist，从字面中就可以看出数据结构的大体形式，hash-skiplist就是每个hash bucket中是一个skiplist，hash-linklist中,每个hash bucket中是一个link-list，启用何用数据结构可在配置中选择，下面是skiplist的数据结构：

![](https://img-blog.csdn.net/20151129155327074)\


图1-4

下面是hash-skiplist的结构，

![](https://img-blog.csdn.net/20151129155517757)\


图1-5

下面是hash-linklist的框架图,

![](https://img-blog.csdn.net/20151129155749362)\


图1-6

3.5 Cache

rocksdb内部根据双向链表实现了一个标准的LRUCache，由于LRUCache的设计实现比较通用经典，这里详细分析一下LRUCache的实现过程，根据LRUCache的从小到大的顺序来看基本组件，

A. LRUHandle结构体，Cache中最小粒度的元素，代表了一个k/v存储对，下面是LRUHandle的所有信息，

struct LRUHandle {

&#x20; void\* value;  // value信息

&#x20; void (\*deleter)(const Slice&, void\* value); //删除元素时，可调用的回调函数

&#x20; LRUHandle\* next\_hash; //解决hash冲突时，使用链表法

&#x20; LRUHandle\* next;//next/prev构成了双链，由LRU算法使用

&#x20; LRUHandle\* prev;

&#x20; size\_t charge;      // TODO(opt): Only allow uint32\_t?

&#x20; size\_t key\_length; //key的长度

&#x20; uint32\_t refs;      // a number of refs to this entry

&#x20;                     // cache itself is counted as 1

&#x20; bool in\_cache;      // true, if this entry is referenced by the hash table

&#x20; uint32\_t hash;      // Hash of key(); used for fast sharding and comparisons

&#x20; char key\_data\[1];   // Beginning of key

&#x20; Slice key() const {

&#x20;   // For cheaper lookups, we allow a temporary Handle object

&#x20;   // to store a pointer to a key in "value".

&#x20;   if (next == this) {

&#x20;     return \*(reinterpret\_cast\<Slice\*>(value));

&#x20;   } else {

&#x20;     return Slice(key\_data, key\_length);

&#x20;   }

&#x20; }

&#x20; void Free() {

&#x20;   assert((refs == 1 && in\_cache) || (refs == 0 && !in\_cache));

&#x20;   (\*deleter)(key(), value);

&#x20;   free(this);

&#x20; }

};

B. 实现了rocksdb自己的HandleTable,其实就是实现了自己的hash table,  速度号称比g++4.4.3版本自带的hash table的速度要快不少

class HandleTable {

&#x20;public:

&#x20; HandleTable() : length\_(0), elems\_(0), list\_(nullptr) { Resize(); }

&#x20; template \<typename T>

&#x20; void ApplyToAllCacheEntries(T func) {

&#x20;   for (uint32\_t i = 0; i < length\_; i++) {

&#x20;     LRUHandle\* h = list\_\[i];

&#x20;     while (h != nullptr) {

&#x20;       auto n = h->next\_hash;

&#x20;       assert(h->in\_cache);

&#x20;       func(h);

&#x20;       h = n;

&#x20;     }

&#x20;   }

&#x20; }

&#x20; \~HandleTable() {

&#x20;   ApplyToAllCacheEntries(\[]\(LRUHandle\* h) {

&#x20;     if (h->refs == 1) {

&#x20;       h->Free();

&#x20;     }

&#x20;   });

&#x20;   delete\[] list\_;

&#x20; }

&#x20; LRUHandle\* Lookup(const Slice& key, uint32\_t hash) {

&#x20;   return \*FindPointer(key, hash);

&#x20; }

&#x20; LRUHandle\* Insert(LRUHandle\* h) {

&#x20;   LRUHandle\*\* ptr = FindPointer(h->key(), h->hash);

&#x20;   LRUHandle\* old = \*ptr;

&#x20;   h->next\_hash = (old == nullptr ? nullptr : old->next\_hash);

&#x20;   \*ptr = h;

&#x20;   if (old == nullptr) {

&#x20;     \++elems\_;

&#x20;     if (elems\_ > length\_) {

&#x20;       // Since each cache entry is fairly large, we aim for a small

&#x20;       // average linked list length (<= 1).

&#x20;       Resize();

&#x20;     }

&#x20;   }

&#x20;   return old;

&#x20; }

&#x20; LRUHandle\* Remove(const Slice& key, uint32\_t hash) {

&#x20;   LRUHandle\*\* ptr = FindPointer(key, hash);

&#x20;   LRUHandle\* result = \*ptr;

&#x20;   if (result != nullptr) {

&#x20;     \*ptr = result->next\_hash;

&#x20;     \--elems\_;

&#x20;   }

&#x20;   return result;

&#x20; }

&#x20;private:

&#x20; // The table consists of an array of buckets where each bucket is

&#x20; // a linked list of cache entries that hash into the bucket.

&#x20; uint32\_t length\_;

&#x20; uint32\_t elems\_;

&#x20; LRUHandle\*\* list\_;

&#x20; // Return a pointer to slot that points to a cache entry that

&#x20; // matches key/hash.  If there is no such cache entry, return a

&#x20; // pointer to the trailing slot in the corresponding linked list.

&#x20; LRUHandle\*\* FindPointer(const Slice& key, uint32\_t hash) {

&#x20;   LRUHandle\*\* ptr = \&list\_\[hash & (length\_ - 1)];

&#x20;   while (\*ptr != nullptr &&

&#x20;          ((\*ptr)->hash != hash || key != (\*ptr)->key())) {

&#x20;     ptr = &(\*ptr)->next\_hash;

&#x20;   }

&#x20;   return ptr;

&#x20; }

&#x20; void Resize() {

&#x20;   uint32\_t new\_length = 16;

&#x20;   while (new\_length < elems\_ \* 1.5) {

&#x20;     new\_length \*= 2;

&#x20;   }

&#x20;   LRUHandle\*\* new\_list = new LRUHandle\*\[new\_length];

&#x20;   memset(new\_list, 0, sizeof(new\_list\[0]) \* new\_length);

&#x20;   uint32\_t count = 0;

&#x20;   for (uint32\_t i = 0; i < length\_; i++) {

&#x20;     LRUHandle\* h = list\_\[i];

&#x20;     while (h != nullptr) {

&#x20;       LRUHandle\* next = h->next\_hash;

&#x20;       uint32\_t hash = h->hash;

&#x20;       LRUHandle\*\* ptr = \&new\_list\[hash & (new\_length - 1)];

&#x20;       h->next\_hash = \*ptr;

&#x20;       \*ptr = h;

&#x20;       h = next;

&#x20;       count++;

&#x20;     }

&#x20;   }

&#x20;   assert(elems\_ == count);

&#x20;   delete\[] list\_;

&#x20;   list\_ = new\_list;

&#x20;   length\_ = new\_length;

&#x20; }

};

HandleTable的结构也是很简单，就是连续一些hash slot，然后用链表法解决hash 冲突，

图1-7

C. LRUCahe&#x20;

LRUCache是由LRUHandle与HandleTable组成，并且LRUCache内部是有锁的，所以外部多线程可以安全使用。

HandleTable很好理解，就是把Cache中的数据hash散列存储，可以加快查找速度；

LRUHandle lru\_是个dummy pointer,也就是双链表的头，也就是LRU是由双链表保存的，队头是最早进入Cache的，队尾是最后进入Cache的，所以，在Cache满了需要释放空间的时候是从队头开始的，队尾是刚进入Cache的元素

class LRUCache {

&#x20;public:

&#x20; LRUCache();

&#x20; \~LRUCache();

&#x20; // Separate from constructor so caller can easily make an array of LRUCache

&#x20; // if current usage is more than new capacity, the function will attempt to

&#x20; // free the needed space

&#x20; void SetCapacity(size\_t capacity);

&#x20; // Like Cache methods, but with an extra "hash" parameter.

&#x20; Cache::Handle\* Insert(const Slice& key, uint32\_t hash,

&#x20;                       void\* value, size\_t charge,

&#x20;                       void (\*deleter)(const Slice& key, void\* value));

&#x20; Cache::Handle\* Lookup(const Slice& key, uint32\_t hash);

&#x20; void Release(Cache::Handle\* handle);

&#x20; void Erase(const Slice& key, uint32\_t hash);

&#x20; // Although in some platforms the update of size\_t is atomic, to make sure

&#x20; // GetUsage() and GetPinnedUsage() work correctly under any platform, we'll

&#x20; // protect them with mutex\_.

&#x20; size\_t GetUsage() const {

&#x20;   MutexLock l(\&mutex\_);

&#x20;   return usage\_;

&#x20; }

&#x20; size\_t GetPinnedUsage() const {

&#x20;   MutexLock l(\&mutex\_);

&#x20;   assert(usage\_ >= lru\_usage\_);

&#x20;   return usage\_ - lru\_usage\_;

&#x20; }

&#x20; void ApplyToAllCacheEntries(void (\*callback)(void\*, size\_t),

&#x20;                             bool thread\_safe);

&#x20;private:

&#x20; void LRU\_Remove(LRUHandle\* e);

&#x20; void LRU\_Append(LRUHandle\* e);

&#x20; // Just reduce the reference count by 1.

&#x20; // Return true if last reference

&#x20; bool Unref(LRUHandle\* e);

&#x20; // Free some space following strict LRU policy until enough space

&#x20; // to hold (usage\_ + charge) is freed or the lru list is empty

&#x20; // This function is not thread safe - it needs to be executed while

&#x20; // holding the mutex\_

&#x20; void EvictFromLRU(size\_t charge,

&#x20;                   autovector\<LRUHandle\*>\* deleted);

&#x20; // Initialized before use.

&#x20; size\_t capacity\_;

&#x20; // Memory size for entries residing in the cache

&#x20; size\_t usage\_;

&#x20; // Memory size for entries residing only in the LRU list

&#x20; size\_t lru\_usage\_;

&#x20; // mutex\_ protects the following state.

&#x20; // We don't count mutex\_ as the cache's internal state so semantically we

&#x20; // don't mind mutex\_ invoking the non-const actions.

&#x20; mutable port::Mutex mutex\_;

&#x20; // Dummy head of LRU list.

&#x20; // lru.prev is newest entry, lru.next is oldest entry.

&#x20; // LRU contains items which can be evicted, ie reference only by cache

&#x20; LRUHandle lru\_;

&#x20; HandleTable table\_;

};

到这，我们从设计现实就能看出一个标准的LRUCache已经成形了，接下来更有意思的是rocksdb又实现了一个ShardedLRUCache，它就是一个封装类，实现了分片LRUCache,在多线程使用的时候，根据key散列到不同的分片LRUCache中，以降低锁的竞争，尽量提高性能。下面一行的代码是一目了然，

LRUCache shard\_\[kNumShards]

D. 另一个很有用的就是ENV，基于不同的平台继承实现了不同的ENV，提供了系统级的各种实现，功能很是强大，对于想做跨平台软件的同学很有借鉴意义。ENV的具体实现就不贴了，主要就是太多。对于其它的工具类，具体可参考src下的相关实现。
