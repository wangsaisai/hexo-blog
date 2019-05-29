---
title: jvm improve docker container detection and resource configuration usage
---

### background
Java startup normally queries the operating system in order to setup runtime defaults for things such as the number of GC threads and default memory limits.

While in docker container, JVM can't get the resource limit info.

This problem had been fixed in JDK8:8u191

This article make a test for the new feature

<!--more-->

### VM info

> a linux vm in google cloud, with 6 vCPUS, 20GB memory

### Test In VM

```bash
bamboo@privacy:~$ free -g
total used free shared buff/cache available
Mem: 19 6 0 0 13 12
Swap: 0 0 0
bamboo@privacy:~$ cat /proc/cpuinfo| grep "processor"| wc -l
6
bamboo@privacy:~$ java -version
java version "1.8.0_171"
Java(TM) SE Runtime Environment (build 1.8.0_171-b11)
Java HotSpot(TM) 64-Bit Server VM (build 25.171-b11, mixed mode)
bamboo@privacy:~$ java -XX:+PrintFlagsFinal | grep -E 'InitialHeapSize|MaxHeapSize'
uintx InitialHeapSize := 329252864 {product}
uintx MaxHeapSize := 5263851520 {product}
bamboo@privacy:~$ java -XX:+PrintFlagsFinal | grep -E ParallelGCThreads
uintx ParallelGCThreads = 6 {product}
```

### Test in docker container (openjdk:8u181) without resource limit

```bash
bamboo@privacy:~$ docker run -ti openjdk:8u181-jdk
root@c559e42aeae2:/# free -g
total used free shared buff/cache available
Mem: 19 6 0 0 12 12
Swap: 0 0 0
root@c559e42aeae2:/# cat /proc/cpuinfo| grep "processor"| wc -l
6
root@c559e42aeae2:/# java -version
openjdk version "1.8.0_181"
OpenJDK Runtime Environment (build 1.8.0_181-8u181-b13-2~deb9u1-b13)
OpenJDK 64-Bit Server VM (build 25.181-b13, mixed mode)
root@c559e42aeae2:/# java -XX:+PrintFlagsFinal | grep -E 'InitialHeapSize|MaxHeapSize'
uintx InitialHeapSize := 329252864 {product}
uintx MaxHeapSize := 5263851520 {product}
root@c559e42aeae2:/# java -XX:+PrintFlagsFinal | grep -E ParallelGCThreads
uintx ParallelGCThreads = 6 {product}
```

### Test in docker container (openjdk:8u181) with resource limit

> 8G memory, 2 cpus

```bash
bamboo@privacy:~$ docker run -it --cpus 2 -m 8g openjdk:8u181-jdk bash
root@5d34cdeb3d5c:/#
root@5d34cdeb3d5c:/# free -g
total used free shared buff/cache available
Mem: 19 6 0 0 12 12
Swap: 0 0 0
root@5d34cdeb3d5c:/# cat /proc/cpuinfo| grep "processor"| wc -l
6
root@5d34cdeb3d5c:/# java -version
openjdk version "1.8.0_181"
OpenJDK Runtime Environment (build 1.8.0_181-8u181-b13-2~deb9u1-b13)
OpenJDK 64-Bit Server VM (build 25.181-b13, mixed mode)
root@5d34cdeb3d5c:/# java -XX:+PrintFlagsFinal | grep -E 'InitialHeapSize|MaxHeapSize'
uintx InitialHeapSize := 329252864 {product}
uintx MaxHeapSize := 5263851520 {product}
root@5d34cdeb3d5c:/# java -XX:+PrintFlagsFinal | grep -E ParallelGCThreads
uintx ParallelGCThreads = 6 {product}
```

### Test in docker container (openjdk:8u212) without resource limit

```bash
bamboo@privacy:~$ docker run -ti openjdk:8u212-jdk
root@9fd6c496e060:/# free -g
total used free shared buff/cache available
Mem: 19 6 0 0 12 12
Swap: 0 0 0
root@9fd6c496e060:/# cat /proc/cpuinfo| grep "processor"| wc -l
6
root@9fd6c496e060:/# java -version
openjdk version "1.8.0_212"
OpenJDK Runtime Environment (build 1.8.0_212-b04)
OpenJDK 64-Bit Server VM (build 25.212-b04, mixed mode)
root@9fd6c496e060:/# java -XX:+PrintFlagsFinal | grep -E 'InitialHeapSize|MaxHeapSize'
uintx InitialHeapSize := 329252864 {product}
uintx MaxHeapSize := 5263851520 {product}
root@9fd6c496e060:/# java -XX:+PrintFlagsFinal | grep -E ParallelGCThreads
uintx ParallelGCThreads = 6 {product}
```

### Test in docker container (openjdk:8u212) with resource limit

> 8G memory, 2 cpus

```bash
bamboo@privacy:~$ docker run -it --cpus 2 -m 8g openjdk:8u212-jdk bash
root@6e90872b448b:/# free -g
total used free shared buff/cache available
Mem: 19 6 0 0 12 12
Swap: 0 0 0
root@6e90872b448b:/# cat /proc/cpuinfo| grep "processor"| wc -l
6
root@6e90872b448b:/# java -version
openjdk version "1.8.0_212"
OpenJDK Runtime Environment (build 1.8.0_212-b04)
OpenJDK 64-Bit Server VM (build 25.212-b04, mixed mode)
root@6e90872b448b:/# java -XX:+PrintFlagsFinal | grep -E 'InitialHeapSize|MaxHeapSize'
uintx InitialHeapSize := 134217728 {product}
uintx MaxHeapSize := 2147483648 {product}
root@6e90872b448b:/# java -XX:+PrintFlagsFinal | grep -E ParallelGCThreads
uintx ParallelGCThreads = 2 {product}
```

### compare table

| env\command | free -g | cat /proc/cpuinfo | jvm InitialHeapSize | jvm MaxHeapSize | jvm ParallelGCThreads |
| --- | --- | --- | --- | --- | --- |
| VM | 20G | 6 | 329252864 (314M) | 5263851520 (4.9G) | 6 |
| docker (openjdk:8u181) | 20G | 6 | 329252864 | 5263851520 | 6 |
| docker (openjdk:8u181) resource limit [cpu:2, memory:8G] | 20G | 6 | 329252864 | 5263851520 | 6 |
| docker (openjdk:8u212) | 20G | 6 | 329252864 | 5263851520 | 6 |
| docker (openjdk:8u181) resource limit [cpu:2, memory:8G] | 20G | 6 | 134217728 (128M) | 2147483648 (2G) | 2 |

### conclusion

- openjdk:8u212 can get the resource limit in docker container

### link

JDK-8146115 : Improve docker container detection and resource configuration usage [https://bugs.java.com/bugdatabase/view_bug.do?bug_id=JDK-8146115](https://bugs.java.com/bugdatabase/view_bug.do?bug_id=JDK-8146115)
