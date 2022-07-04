---
title: Flink KeyGroup机制导致的数据倾斜问题
---

# Flink KeyGroup机制导致的数据倾斜问题

### 业务背景
数据源位于Kafka中，将相同Key的数据聚合成Batch，写入HBase中.

即：kafka(sourceStream) --> keyby(batch) --> hbase(sink)

核心代码如下
```java
sourceStream
    .keyBy(CustomerData::getCustomerKey)
    .process(new CustomerKeyedProcessFunction())
    .returns(CustomerBatch.class)
    .addSink(hbaseSink);
```

使用Flink版本：1.13.2

### 数据倾斜现象

测试Case：Flink并发度：180，SubTask高低倾斜率约为216:144 = 3：2

<!--more-->

![图片1][1]

### 原因分析
Key-Groups是Flink对Key进行分组。进入Flink的数据有无限种可能，把无限可能的Key通过某种算法分成有限个组。

key根据其HashCode会分配到某个keyGroup，算法如下：
```java
public static int assignToKeyGroup(Object key, int maxParallelism) {
    return computeKeyGroupForKeyHash(key.hashCode(), maxParallelism);
}
 
public static int computeKeyGroupForKeyHash(int keyHash, int maxParallelism) {
    // MathUtils.murmurHash(int hash) 方法是Flink提供的静态方法，将hash值再次散列，避免用户数据分布不均
    return MathUtils.murmurHash(keyHash) % maxParallelism;
}
```

KeyGroup数量等于最大并发度，最大并行度的算法如下：
```java
public static int computeDefaultMaxParallelism(int operatorParallelism) {
 
    checkParallelismPreconditions(operatorParallelism);
 
 return Math.min(
            Math.max(
                    MathUtils.roundUpToPowerOfTwo(
                            operatorParallelism + (operatorParallelism / 2)),
 DEFAULT_LOWER_BOUND_MAX_PARALLELISM),
 UPPER_BOUND_MAX_PARALLELISM);
}
```
即 min(32768, max(128, MathUtils.roundUpToPowerOfTwo(operatorParallelism*1.5)))

测试时，operatorParallelism = 180， 计算得到的MaxParallelism=512，总共有512个KeyGroup.

512个KeyGroup分配给180个SubTask，(512/180=2.84), 少数SubTask会被分配到2个KeyGroup，多数SubTask会被分配到3个KeyGroup。因此数据倾斜比为2：3，跟观测结果一致

### 解决
自定义MaxParallelism参数，不使用默认值
```java
env.getConfig().setMaxParallelism(4096);
```

> Flink官网提醒注意：调高最大并行度产生更多Key Groups组数，使状态元数据增大，Checkpoint快照也随之增大，降低性能。
> 实际测试发现，该部分影响很小（未发现checkpoint大小显著变化）,推测原因是状态总数据量大，导致状态元数据量的增长不明显


### Flink为何如此设计 - KeyGroup由来
Flink为何不根据自定义并发度进行计算？
在Flink早期版本中，并没有KeyGroup的概念，数据按照任务并发度来拆分。
但存在一个弊端：如果后续需要修改并发度，任务重启时，需要重新将CheckPoint中保存的Key进行计算，重新分配到各个subtask中，耗时较长。
为了解决这个问题，Flink才设计了KeyGroup机制。


### 参考文档
1. [https://www.yuque.com/deadwind/notes/flink-key-groups](https://www.yuque.com/deadwind/notes/flink-key-groups)
2. [https://issues.apache.org/jira/browse/FLINK-3755](https://issues.apache.org/jira/browse/FLINK-3755)
3. [https://blog.csdn.net/nazeniwaresakini/article/details/104220138](https://blog.csdn.net/nazeniwaresakini/article/details/104220138)


[1]: /images/flink-keygroup-dataskew.png "skew"
