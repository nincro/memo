# 概述
> ### 垃圾回收机制

# 垃圾标记
## 引用记数法
>实现简单，无法解决循环引用的问题

## GC Root 可达性计算
> ### GC Root
可以作为 GC root 的对象有以下几种
static 引用的对象
虚拟机栈局部变量表 中 引用的对象
本地方法栈局部变量表 中 引用的对象

# 垃圾清理
## 复制
> ### 复制的前提是有两个区域
在年轻代比较多使用。
具体的实现方式就是将一个区域的标记对象全部顺序复制到另一个区域


## 直接清理
> ### 将没有标记的对象的内存空间直接标为可用。
维护一张可用内存的表，指向这些内存块，注明大小。
显然这会造成内存碎片。

## 整理（压缩）
> ### 将标记的对象按顺序挪到前排，然后其它内存区域标为可用。