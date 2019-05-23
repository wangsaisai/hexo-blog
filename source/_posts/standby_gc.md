---
title: 生产环镜Standby NameNode GC严重问题记录
---

HDFS集群正常运行了将近一年，最近两个月出现一个诡异的问题：Standby节点严重的GC，Active节点正常。虽然不影响线上业务，但潜在的问题还是比较严重的，这里简单整理下问题定位和解决过程。

# 一、问题描述
HDFS版本：2.5.2 （问题与版本无关）

HDFS的active节点正常，Standby节点严重GC


# 二、集群的情况：

HDFS集群数据量在几百TB，Namespace和Block数量在千万级别，每天增长大约几十GB的数据量。Namespace和Block的信息消耗的内存会略微有部分增长，但不会发生太大变化。

# 三、问题分析

问题最诡异的地方在于，Standby节点GC异常，但Active节点GC正常。

HDFS的内存主要消耗是Namespace和Block信息。这些信息是常住内存的，不会发生太大变动。
一般来讲，Active节点的任务量要超过Standby节点，但由于内存主要消耗在Namespace和Block信息，Namespace和Block信息在Active/Standby节点是完全一致的，所以两个节点的内存消耗应该接近，Active节点略微比Standby节点多消耗一些内存。

<!--more-->

# 四、问题追踪

### Checkpoint角度分析

刚开始从Standby节点GC超过Active节点的角度进行分析，什么任务是Standby节点会做，而Active节点不做的? -- CheckPoint，即检查点

*注：CheckPoint并不是某个时刻，而是一个操作过程。指将HDFS的fsimage和editlog进行合并的过程。详细信息，可查看下HDFS的HA实现。*

初步怀疑是做检查点时，代码上存在内存泄漏，导致了Standby严重GC。由于是生产环镜，不能远程Debug，只能通过阅读源码形式分析，效率缓慢，没有进展。

### jvm角度分析

CheckPoint的思路没有进展。后来换个思路，追踪下Standby GC的详细情况。

jstat，jmap等jvm相关的知识可参考笔者另一篇文章：   [《jvm 简介》](/2017/05/04/jvm/)

通过打jstat发现，Standby节点fullgc频率异常，一天有几十次的fullgc，而且fullgc占用的时间很长，一次大约会使用100s的时间。另外，Standby节点fullgc过后，jvm内存使用率立刻降低。

然后，开始通过打jmap进行分析。考虑到，针对Standby节点打的jmap，并不影响Active节点的正常服务，也就不影响HDFS的服务可用性，再加上该问题一直没有进展，就选择了这个方案。

**打jmap对jvm进程影响非常大，可能导致jvm停顿。特别是对于hdfs这种内存占用较大的进程，可能会导致几分钟的停顿，所以生产环境一定要慎用。**

笔者共打了四次jmap

两个时间点：
1. 内存高位时，即将发生gc的时刻。
2. 刚刚gc过后

两种jmap方式（把全部的对象和活着的对象都打印出来）：
1. jmap -histo:live pid
2. jmap -histo pid

四次结果：
- jmap_low_all: 内存低位，所有对象
- jmap_low_live: 内存低位，活着对象
- jmap_high_all: 内存高位，所有对象
- jmap_high_live: 内存高位，活着对象

分析发现，jmap_low_all，jmap_low_live，jmap_high_live三次结果接近，无太大变化；jmap_high_all与其他3此差别较大，多出来的内存大致如下：

![图片1][1]

分析发现：多出的内存主要是PersistToken对象。
```java
PersistToken {
  Object owner_;
  Object renewer_;
  Object realUser_;
   ......
}
```

根据PersistToken类中的方法getOwner() getRenewer()，getRealUser()可知 ，owner_,  renewer_, realUser_ 都是String或者com.google.protobuf.ByteString类型，而LiteralByteString是com.google.protobuf.ByteString（ByteString是抽象类）的一个子类

```java
LiteralByteString {
   byte[] bytes;
}

String {
    char value[];
}
```

所以，推测内存的增长都是因为PersistToken对象导致的。

然后在分析检查点的代码
```java
CheckPointer {
  void doCheckpoint() {
      bnImage.saveFSImageInAllDirs(backupNode.getNamesystem(), txid);
  }
}
```

上面代码，一层层看下去，会调到
DelegationTokenSecretManager的saveSecretManagerState()方法

```java
public synchronized SecretManagerState saveSecretManagerState() {
...
ArrayList<SecretManagerSection.PersistToken> tokens = Lists
    .newArrayListWithCapacity(currentTokens.size());
...
for (Entry<DelegationTokenIdentifier, DelegationTokenInformation> e : currentTokens
   .entrySet()) {
 DelegationTokenIdentifier id = e.getKey();
 SecretManagerSection.PersistToken.Builder b = SecretManagerSection.PersistToken
     .newBuilder().setOwner(id.getOwner().toString())
     .setRenewer(id.getRenewer().toString())
     .setRealUser(id.getRealUser().toString())
     .setIssueDate(id.getIssueDate()).setMaxDate(id.getMaxDate())
     .setSequenceNumber(id.getSequenceNumber())
     .setMasterKeyId(id.getMasterKeyId())
     .setExpiryDate(e.getValue().getRenewDate());
 tokens.add(b.build());       //tokens太大，所以这部分内存直接进入old区，minor gc无法回收
}
...
}
```

通过log发现，currentTokens数量在千万级别。在做检查点时，这些tokens会转化成PersistToken,以方便将tokens序列化到fsimage中。统计下来，PersistToken会占用4-5GB的内存，这些内存超出了eden区的大小，因此会被直接存入old区，old区会快速的增长4-5GB，容易触发CMS进行回收，回收过程甚至可能导致内存不足，因此引发fullgc导致的STW（stop the world）。 检查点默认每个小时都会执行一次（可以配置成其他值），当editlog数量达到1百万时，也会触发检查点任务。周期性的检查点任务，导致了Standby节点gc严重的问题。

所以，Standby节点异常的原因就是因为tokens太多了，但问题并没有结束，tokens从哪里来的，为何产生这么多token，token如何回收？

tokens的来源是HDFS上层应用产生的，此处暂时不讲解这个问题（稍后笔者可能会在写篇博文进行介绍）。

tokens的回收有两种方式
- 一种是上层应用调用hdfs接口进行删除。
- 一种是等待tokens自动过期，然后清除。

tokens默认过期时间是7天，但配置文件中将相关参数调大了一万倍，7万天后才会自动清除（调大的原因是上层应用的一个bug，这个bug一时难以解决，就暂且调大了这个配置来规避这个bug）。由于集群运行了将近一年，所以tokens数量一直增长至千万级别。

注：tokens相关配置，及默认值：（hdfs-site.xml）
- dfs.namenode.delegation.token.max-lifetime = 604800000
- dfs.namenode.delegation.token.renew-interval = 86400000



# 五、问题解决

- 首先，是解决上层应用的bug
- 然后，把tokens过期时间的参数值调整回来
- 清理已有token。（已有的token其过期时间已经设定为7万天后，即使修改配置文件，也不会对已有token产生影响。所以，还要想办法清理已有的token）

# 六、经验

回头看问题追踪的整个过程，有几点值得总结的经验：

1. jvm提供了详细的工具帮助定位问题，一定要善于利用这些工具
2. jmap要慎用。而且打jmap要选择合适的时间点。
3. 遇到问题尽量解决，规避问题要考虑潜在的风险。
4. 问题的现象与具体的原因可能存在一定距离，必要是需要换换思路，不能钻牛角尖。


# 七、推荐阅读

1. [《HDFS NameNode内存详解》](https://tech.meituan.com/namenode-memory-detail.html)
2. [《HDFS NameNode内存全景》](https://tech.meituan.com/namenode.html)
3. [《jvm 简介》](/2017/05/04/jvm/)
4. [《HDFS High Availability》](http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-hdfs/HDFSHighAvailabilityWithNFS.html)


[1]: /images/jmap.png "jmap"
