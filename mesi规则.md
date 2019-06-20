# 概述
> jmm 内定的一些规则，用于保证执行顺序

# 具体如下
## 单线程下statment按顺序执行（statement会被拆成多个指令）
## 管程的unlock先于lock执行
## 线程的start先于新线程的statement执行
## 线程的join慢于新线程的返回而返回
## 构造方法先于finalize方法执行
## 对volatile变量的写statement先于读statement
