---
title: jvm improve docker container detection and resource configuration usage
date: 2019-06-13 00:00:00
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

### implementation

```
 julong os::Linux::available_memory() {
   // values in struct sysinfo are "unsigned long"
   struct sysinfo si;
-  sysinfo(&si);
+  julong avail_mem;
 
-  return (julong)si.freeram * si.mem_unit;
+  if (OSContainer::is_containerized()) {
+    jlong mem_limit, mem_usage;
+    if ((mem_limit = OSContainer::memory_limit_in_bytes()) > 0) {
+      if ((mem_usage = OSContainer::memory_usage_in_bytes()) > 0) {
+        if (mem_limit > mem_usage) {
+          avail_mem = (julong)mem_limit - (julong)mem_usage;
+        } else {
+          avail_mem = 0;
+        }
+        log_trace(os)("available container memory: " JULONG_FORMAT, avail_mem);
+        return avail_mem;
+      } else {
+        log_debug(os,container)("container memory usage call failed: " JLONG_FORMAT, mem_usage);
+      }
+    } else {
+      log_debug(os,container)("container memory unlimited or failed: " JLONG_FORMAT, mem_limit);
+    }
+  }
+
+  sysinfo(&si);
+  avail_mem = (julong)si.freeram * si.mem_unit;
+  log_trace(os)("available memory: " JULONG_FORMAT, avail_mem);
+  return avail_mem;
 }
  
 julong os::physical_memory() {
-  return Linux::physical_memory();
+  if (OSContainer::is_containerized()) {
+    jlong mem_limit;
+    if ((mem_limit = OSContainer::memory_limit_in_bytes()) > 0) {
+      log_trace(os)("total container memory: " JLONG_FORMAT, mem_limit);
+      return (julong)mem_limit; 
+    } else {
+      if (mem_limit == OSCONTAINER_ERROR) {
+        log_debug(os,container)("container memory limit call failed");
+      }
+      if (mem_limit == -1) {
+        log_debug(os,container)("container memory unlimited, using host value");
+      }
+    }
+  }
+
+  jlong phys_mem = Linux::physical_memory();
+  log_trace(os)("total system memory: " JLONG_FORMAT, phys_mem);
+  return phys_mem;
 }
```

### link

JDK-8146115 : Improve docker container detection and resource configuration usage 

jira : [https://bugs.java.com/bugdatabase/view_bug.do?bug_id=JDK-8146115](https://bugs.java.com/bugdatabase/view_bug.do?bug_id=JDK-8146115)

patch : [http://cr.openjdk.java.net/~bobv/8146115/webrev.03/open.patch](http://cr.openjdk.java.net/~bobv/8146115/webrev.03/open.patch)