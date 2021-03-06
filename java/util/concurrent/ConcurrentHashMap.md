# 版本 1.8
# 概要
* ### 专为并发设计的集合类
>读操作不加锁，但是读操作只反映最近完成的结果，也可能和它开始执行的那一瞬间的状态有出入

* ### 和Hashtable比较
>Hashtable使用synchronized来保证线程安全，在线程竞争不十分激烈的情况下，Hashtable的效率较低。
而竞争十分激烈的情况下，ConcurrentHashMap自旋过久，浪费cpu，所以还是使用Hashtable

* ### 采用锁分段技术 来保证高效的并发操作
>它没有全局锁（表锁），使用的是Segment的锁
>把容器分为多个 segment（片段），每个片段有一把锁。
>当多线程访问容器里不同数据段的数据时，线程间就不会存在竞争关系；一个线程占用锁访问一个segment的数据时，并不影响另外的线程访问其他segment中的数据。
> 类似于 数据库 的行锁概念
> ### 1.8 中不再存在segment对象，hash桶实现了segment的概念
hash桶的第一个元素如果存在，就作为这个哈希桶的锁，使用synchronized

* ### 获取桶下标的时候和HashMap一样做了扰动
>用高16位与低16位异或。再与上HASH_BITS取余。HASH_BITS为0x7f ff ff ff

* ### 牺牲了强一致性
> ConcurrentHashMap在多线程之间可能会存在不一致状态
> 但是可以保证的是 多个状态最终可以到达一致性

* ### jdk1.8的ConcurrentHashMap不再使用Segment代理Map操作这种设计

# Constant

## MAXIMUM_CAPACITY
> 最大元素数量 1 << 30;

## DEFAULT_CAPACITY
> 默认哈希桶数组长度 16;

## MAX_ARRAY_SIZE
> 哈希桶数组最大长度 Integer.MAX_VALUE - 8;

## LOAD_FACTOR
> 加载因子  0.75f

## TREEIFY_THRESHOLD
> 树化阈值 8

## UNTREEIFY_THRESHOLD
> 链表化阈值 6;

# Field
## Node<K,V>[] table
```java
 transient volatile Node<K,V>[] table;
 ```
>

## Node<K,V>[] nextTable
```java
private transient volatile Node<K,V>[] nextTable;
```
>扩容后的新的table数组，只有在扩容时才有用
nextTable != null，说明扩容方法还没有真正退出，一般可以认为是此时还有线程正在进行扩容



## Segment<K,V>[] segments
> 把哈希桶数组分为多个，每个segment中有部分的table

## int sizeCtl
```java
private transient volatile int sizeCtl;
```
> 保存 initialCapacity 和 loadFactor 信息的重要成员变量，有四种状态
* ### -1，表示有线程正在进行初始化操作
* ### -(1 + nThreads)，表示有n个线程正在一起扩容
* ### 0，默认值，后续在真正初始化的时候使用默认容量
* ### > 0，初始化或扩容完成后下一次的扩容门槛




# 构造函数
## ConcurrentHashMap(int initialCapacity,float loadFactor, int concurrencyLevel)
> ### 和HashMap对比
* initialCapacity 和 loadFactor 最后转化为 sizeCtl
sizeCtl 之后会在 iniTable 函数中编程哈希桶数组的初始长度


```java
public ConcurrentHashMap(int initialCapacity,
                             float loadFactor, int concurrencyLevel) {
        if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
            throw new IllegalArgumentException();
            //初始哈希桶数组长度至少要和并发的等级一样
        if (initialCapacity < concurrencyLevel)   // Use at least as many bins
            initialCapacity = concurrencyLevel;   // as estimated threads
            //对期望的初始容量，加载因子做手脚
        long size = (long)(1.0 + (long)initialCapacity / loadFactor);
        //这里是对哈希桶长度补1然后加1获得length，方便之后对key进行hash操作
        //只需要直接和len-1做与操作
        int cap = (size >= (long)MAXIMUM_CAPACITY) ?
            MAXIMUM_CAPACITY : tableSizeFor((int)size);
        this.sizeCtl = cap;
    }
```

# 重要函数
## tabAt(Node<K,V>[] tab, int i)
## casTabAt(Node<K,V>[] tab, int i, Node<K,V> c, Node<K,V> v)


## spread
> concurrentHashmap的hash函数
```java
static final int spread(int h) {
    return (h ^ (h >>> 16)) & HASH_BITS;
}
```
>添加元素的时候获取哈希桶下标的函数为
```java
int hash = spread(key.hashCode());
```


## initTable()
> ### 初始化哈希桶array
在table为空或者length为0的条件下进入下面的 **循环**
>> 首先获取sizeCtl
>> ### 如果sizeCtl 小于0
说明有其他线程正在调用该函数，当前线程调用yield让出cpu
>> ### 否则，CAS操作尝试将sizectl修改为-1，也就是当前线程正在占用的意思。
> 然后根据CAS操作的结果分类讨论
>> ### CAS 失败
说明有其他线程先一步进入初始化了，当前线程进入下一轮 **循环** 。
下一轮的循环有两种可能性，
1. **sizectl<0** 另外的线程仍然在初始化，此时 sizectl<0,则当前线程让出cpu
2. **length>0** 另外的线程初始化完成，此时length大于0，当前线程跳出循环。
>>
>> ### CAS 成功
>那么当前线程开始进入初始化逻辑
再次检查 table 是否为 空，如果不为空则继续初始化
如果sizectl为0，说明没有指定初始容量，使用默认的容量，也就是16
然后为哈希桶数组分配内存，赋值给类对象的table
然后设置扩容阈值为当前哈希桶数组长度的**0.75**倍，此处直接写死了。
然后把扩容阈值赋值给sizectl。
**这里与HashMap十分类似。所谓的表示阈值的变量一开始都是用作初始化数组长度**
> ### 要点
* ### 使用CAS锁控制只有一个线程执行初始化桶数组的逻辑
* ### sizeCtl在初始化前存储的是初始容量
* ### sizeCtl在初始化后存储的是扩容阈值

## putVal(K key, V value, boolean onlyIfAbsent)
> **外层循环**
>### 如果桶数组未初始化
则走initTable初始化；
>### 根据要插入的桶的状态分类讨论
>> ### 如果为空
则 **CAS尝试** 把此元素直接插入到桶的第一个位置；
>>> ### 如果失败
那么走 **外层循环** 直到添加成功为止
>>> ### 如果成功
那么跳出 **外层循环**，添加完成
>> ### 如果不为空，但是第一个元素的hash为MOVED
说明该桶被标记为 MOVED。正在迁移元素
那么当前线程走 helpTransfer(tab, f) ，帮忙迁移该桶的元素
>> ### 如果不为空，也没有在迁移数据
那么首先锁住这个桶
synchronized(表头)
>>> ### 如果第一个元素的hash值>=0
说明没有在迁移，也不是树，而是链表
>>> ### 然后查找要插入的元素是否存在
>>>> ### 如果存在
>>>>> ### 如果 onlyIfAbsent 为true
那么不修改
>>>>> ### 否则
进行value的替换
>>>> 然后跳出循环
>>>> ### 如果不存在
那么将该元素插入尾部。
>>> ### 第一个元素的hash < 0且使用instanceof 验证为 TreeBin
说明是红黑树节点
>>> ### 调用红黑树算法寻找该元素
>>>> ### 如果没找到
会获得null返回值
那么说明已经添加完毕
>>>> ### 找到了
会获得该节点的引用
那么根据 onlyIfAbsent 判断是否要替换即可
> ### 最后判断 binCount
如果是链表操作，该值有可能会增加
如果此时超过 TREEIFY_THRESHOLD，那么进行树化
调用addCount根据需要扩容
如果添加的元素key已经存在返回旧value

## addCount(long x, int check)


## transfer

## helpTransfer(Node<K,V>[] tab, Node<K,V> f)
> 协助扩容中的迁移步骤

## put(K key, V value)
```java
public V put(K key, V value) {
    return putVal(key, value, false);
}

```

## putAll
## clear



# 重要内部类
## Segment<K,V>
```java
static class Segment<K,V> extends ReentrantLock implements Serializable {
    }
```
> ### 集成 ReentrantLock

### 重要 attr


#### HashEntry<K,V>[] table
> 真正存放数据的地方


### 构造函数


### 重要函数

## Node<K,V>
```java
static class Node<K,V> implements Map.Entry<K,V> {


}
```
### 概述
> ### 这是链表Node的实现
> ### next 是 volatile修饰的
> ### 没有重写hashCode
因此调用的应当是Object的hashCode

### Constant
### Field
#### volatile V val
>对多线程可见，因此线程修改value可以直接使用cas
#### volatile Node<K,V> next

### 构造函数
### 重要函数
#### find(int h, Object k)
```java
static class Node<K,V> implements Map.Entry<K,V> {
    Node<K,V> e = this;
    if (k != null) {
        do {
            K ek;
            if (e.hash == h &&
                ((ek = e.key) == k || (ek != null && k.equals(ek))))
                return e;
        } while ((e = e.next) != null);
    }
    return null;
}
```
>从此节点开始查找k对应的节点
**一般作用于头结点，各种特殊的子类有自己独特的实现**
不过主体代码中进行链表查找时，因为要特殊判断下第一个节点，所以很少直接用下面这个方法，而是直接写循环遍历链表，子类的查找则是用子类中重写的find方法

## TreeNode<K,V>
```java
static final class TreeNode<K,V> extends Node<K,V> {
}
```
### 概述
> ### 这是 Node 的一种实现 红黑树节点
> ### 结构和1.8的HashMap的TreeNode一样，一些方法的实现代码也基本一样
> ### ConcurrentHashMap对此节点的操作，都会由TreeBin来代理执行
可以把这里的TreeNode看出是有一半功能的HashMap.TreeNode，另一半功能在ConcurrentHashMap.TreeBin中。
### Constant
### Field
### Constructor
### Method
#### find(int h, Object k)
```java
Node<K,V> find(int h, Object k) {
    return findTreeNode(h, k, null);
}
// 以当前节点 this 为根节点开始遍历查找，跟HashMap.TreeNode.find实现一样
final TreeNode<K,V> findTreeNode(int h, Object k, Class<?> kc) {
    if (k != null) {
        TreeNode<K,V> p = this;
        do  {
            int ph, dir; K pk; TreeNode<K,V> q;
            TreeNode<K,V> pl = p.left, pr = p.right;
            if ((ph = p.hash) > h)
                p = pl;
            else if (ph < h)
                p = pr;
            else if ((pk = p.key) == k || (pk != null && k.equals(pk)))
                return p;
            else if (pl == null)
                p = pr;
            else if (pr == null)
                p = pl;
            else if ((kc != null || (kc = comparableClassFor(k)) != null) && (dir = compareComparables(kc, k, pk)) != 0)
                p = (dir < 0) ? pl : pr;
            else if ((q = pr.findTreeNode(h, k, kc)) != null) // 对右子树进行递归查找
                return q;
            else
                p = pl; // 前面递归查找了右边子树，这里循环时只用一直往左边找
        } while (p != null);
    }
    return null;
}
```

##  TreeBin<K,V>
```java
static final class TreeBin<K,V> extends Node<K,V> {
}
```
### 概述
> ### 代理一部分对红黑树的操作
持有指向红黑树的根节点引用，对红黑树的操作通过 TreeBin 的接口
> ### 业务判断桶类型的时候，根据TreeBin进行判断


## ForwardingNode
```java
static final class ForwardingNode<K,V> extends Node<K,V> {
}
```
### 概述
> ### ForwardingNode是一种临时节点
在扩容进行中才会出现，hash值固定为-1
> ### 它不存储实际的数据数据。
