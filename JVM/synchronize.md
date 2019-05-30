
# 版本 1.8
# 概述


# 别人的总结
>HotSpot虚拟机中，对象在内存中存储的布局可以分为三块区域：对象头（Header）、实例数据（Instance Data）和对齐填充（Padding）。

>HotSpot虚拟机的对象头(Object Header)包括两部分信息，第一部分用于存储对象自身的运行时数据， 如哈希码（HashCode）、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等等，这部分数据的长度在32位和64位的虚拟机（暂 不考虑开启压缩指针的场景）中分别为32个和64个Bits，官方称它为“Mark Word”。

>对象头的另外一部分是类型指针，即是对象指向它的类的元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例。并不是所有的虚拟机实现都必须在对象数据上保留类型指针，换句话说查找对象的元数据信息并不一定要经过对象本身。另外，如果对象是一个Java数组，那在对象头中还必须有一块用于记录数组长度的数据，因为虚拟机可以通过普通Java对象的元数据信息确定Java对象的大小，但是从数组的元数据中无法确定数组的大小。  

res 在堆中的内存分配由三个部分组成，分别是 对象头，实例数据，对齐填充 padding。对象头中的markword是我们本次要说的重点。

# 我的思考

## 假设一个场景
* ### 线程对象 thread1， thread2
* ### 资源对象 res

### 从无锁（01）升级到偏向锁（01）
>thread1 发现
1 markword中锁标识为01，
2 且没有指向的threadid，而是该对象的hashcode，
3 且偏向标志为可以偏向。
那么通过cas操作，对该markword进行修改，将hashcode改为thread1的ThreadID。至此 res 偏向thread1。
之后如果thread1再次请求该锁，那么只需要验证res的指针仍然指向thread1即可，无需再次进行CAS操作。这说明偏向锁也是一种可重入锁。

### 从偏向锁（01）降级为无锁（01）再升级到偏向锁（01）
>thread2 发现
1 markword中锁标识为01，
2 且有指向的threadid，该id不是thread2的，而是thread1的
3 且偏向标志为可以偏向。
那么通知虚拟机查看该threadid指向的thread1是否已经停止执行。
1 如果已经停止执行，那么偏向锁会降级为无锁，然后thread2再次尝试使用CAS修改res偏向自己。
2 如果还在执行，

### 2 从偏向锁（01）升级到轻量级锁（00），这里需要分为两种情况讨论。
>1 thread1 对 markword 的 cas 操作失败，则通知撤销偏向锁，升级为轻量级锁。

>1 thread2 发现 markword 标识处于偏向锁状态， 偏向的threadID 不是 thread2的， 通知撤销偏向锁，升级为轻量级锁。

>接下来是共同的部分。
>虚拟机运行到safe point的时候（此时没有任何字节码在运行），Stop The World， 然后将markword设置为偏向锁的状态


### 3 从轻量级锁（00）升级到重量级锁（10）