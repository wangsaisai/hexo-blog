---
title: java 9 turorial
---

## JShell

> 交互式执行java代码

```java
jshell> int a=10
a ==> 10

jshell> a*7+5
$30 ==> 75

jshell> System.out.println("hello")
hello

jshell> void method(int x, int y) {
   ...>   System.out.println(x+y);
   ...> }
|  已修改 方法 method(int,int)

jshell> method(10,20);
30

```

<!--more-->

## 不可变List, Set, Map

#### Immutable List
```java
jshell> List l = List.of(1,2,3);
l ==> [1, 2, 3]

jshell> l.add(4);
|  警告:
|  对作为原始类型java.util.List的成员的add(E)的调用未经过检查
|  l.add(4);
|  ^------^
|  java.lang.UnsupportedOperationException thrown:
|        at ImmutableCollections.uoe (ImmutableCollections.java:71)
|        at ImmutableCollections$AbstractImmutableList.add (ImmutableCollections.java:77)
|        at (#36:1)
```

#### Immutable Set
```java
jshell> Set set = Set.of(1,2,3);
set ==> [1, 3, 2]

jshell> set.add(4);
|  警告:
|  对作为原始类型java.util.Set的成员的add(E)的调用未经过检查
|  set.add(4);
|  ^--------^
|  java.lang.UnsupportedOperationException thrown:
|        at ImmutableCollections.uoe (ImmutableCollections.java:71)
|        at ImmutableCollections$AbstractImmutableSet.add (ImmutableCollections.java:281)
|        at (#38:1)


```

#### Immutable Map
```java
jshell> Map.of()
$39 ==> {}

jshell> Map map = Map.of(1, "a", 2, "b", 3, "c");
map ==> {1=a, 3=c, 2=b}

jshell> map.put(4, "d");
|  警告:
|  对作为原始类型java.util.Map的成员的put(K,V)的调用未经过检查
|  map.put(4, "d");
|  ^-------------^
|  java.lang.UnsupportedOperationException thrown:
|        at ImmutableCollections.uoe (ImmutableCollections.java:71)
|        at ImmutableCollections$AbstractImmutableMap.put (ImmutableCollections.java:558)
|        at (#41:1)

```

## 接口私有方法 Private Method in interface

com.interfaces.PrivateMethod

## 模块化 Module System

- com.hello
- com.hello.client

## Process API
```java
jshell> ProcessHandle currentProcess = ProcessHandle.current();
currentProcess ==> 3214

jshell> System.out.println("Current Process Id: = " + currentProcess.pid());
Current Process Id: = 3214

```

## 资源对象的try/catch

```java
jshell> void tryResource() throws IOException {
   ...>   BufferedReader reader1 = new BufferedReader(new FileReader("/tmp/a.txt"));
   ...>   try (reader1) {
   ...>     System.out.println(reader1.readLine());
   ...>   }
   ...> }
|  已修改 方法 tryResource()

```

## Stream api improvements

com.stream.StreamTest

## optional

```java
jshell> Optional.of(1)
$46 ==> Optional[1]

jshell> Optional.empty();
$47 ==> Optional.empty

jshell> Optional.of(1).ifPresentOrElse(x -> System.out.println("get:" + x), () -> System.out.println("cannot get"))
get:1

jshell> Optional.empty().ifPresentOrElse(x -> System.out.println("get:" + x), () -> System.out.println("cannot get"))
cannot get

jshell> Optional.empty().or(() -> Optional.of(2))
$50 ==> Optional[2]

```

## HTTP 2 Client API

api改进
支持web socket 等

## GC

- CMS垃圾回收算法不再推荐使用; 推荐性能更好的G1算法
- GC算法之间的绑定关系被取消

## other
java docs改进

## 源代码
本文全部源代码：https://github.com/wangsaisai/java-9-tutorial.git
