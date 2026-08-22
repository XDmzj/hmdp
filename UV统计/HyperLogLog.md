

# 概念讲解


![[Pasted image 20260822143706.png]]


HyperLogLog用法
Hyperloglog(HLL)是从Loglog算法派生的概率算法，用于确定非常大的集合的基数，而不需要存储其所有值。相关算法原理大家可以参考：https://juejin.cn/post/6844903785744056333#heading-0

Redis中的HLL是基于string结构实现的，单个HLL的内存永远小于16kb，内存占用低的令人发指！作为代价，其测量结果是概率性的，有小于**0.81％**的误差。不过对于UV统计来说，这完全可以忽略。

~~~HyperLogLog常用指令

PFADD key element [element...] ：添加指定元素到 HyperLogLog 中

PFCOUNT key [key ...]：返回给定 HyperLogLog 的基数估算值

PFMERGE destkey sourcekey [sourcekey ...]：将多个 HyperLogLog 合并为一个 HyperLogLog
~~~
HyperLogLog的作用：做海量数据的统计工作

HyperLogLog的优缺点：

优点：内存占用极低、性能非常好

缺点：有一定的误差


# 实现
