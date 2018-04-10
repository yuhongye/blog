一般来说JVM的参数使用:-XX:[+-]parameters，其中`+`表示打开该参数，`-`表示关闭该参数;-XparamValue，比如`-Xmx10m`，设定堆空间最大10m
# 1. 参数
#### 1. DoEscapeAnalysis和EliminateAllocations
逃逸分析和标量替换

# 2. GC参数
#### PrintGC 每次GC打印一条log
```java
//                   GC前堆已使用 -> GC之后堆已使用(堆的总大小)
[GC (Allocation Failure)  8376K->8072K(31744K), 0.0034442 secs]
[GC (Allocation Failure)  16435K->16264K(40448K), 0.0033094 secs]
[Full GC (Ergonomics)  16264K->16149K(53248K), 0.0030093 secs]
```

#### PrintGCDetails 打印GC的详细信息
```java

// 新生代GC   GC前新生代已使用->GC后新生代已使用(新生代总大小) GC前堆已使用->GC后堆已使用(堆大小)，可以看出是新生代->老年代
[GC (Allocation Failure) [PSYoungGen: 32896K->640K(42496K)] 84376K->84376K(127488K), 0.0376845 secs] [Times: user=0.02 sys=0.01, real=0.04 secs] 
// Full GC，包括了新生代，老年代，元空间
[Full GC (Ergonomics) [PSYoungGen: 640K->0K(42496K)] [ParOldGen: 83736K->84250K(87552K)] 84376K->84250K(130048K), [Metaspace: 2643K->2643K(1056768K)], 0.0042683 secs] [Times: user=0.00 sys=0.01, real=0.00 secs] 
100
[Full GC (System.gc()) [PSYoungGen: 18718K->0K(42496K)] [ParOldGen: 84250K->51479K(87552K)] 102968K->51479K(130048K), [Metaspace: 2646K->2646K(1056768K)], 0.0092569 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
// 堆空间信息情况
Heap
 // 新生代        总大小         已使用 新生代地址空间的：下届          当前上届            上届
 PSYoungGen      total 42496K, used 829K [0x00000007bd580000, 0x00000007c0000000, 0x00000007c0000000)
  eden space 41472K, 2% used [0x00000007bd580000,0x00000007bd64f738,0x00000007bfe00000)
  from space 1024K, 0% used [0x00000007bff00000,0x00000007bff00000,0x00000007c0000000)
  to   space 1024K, 0% used [0x00000007bfe00000,0x00000007bfe00000,0x00000007bff00000)
 ParOldGen       total 87552K, used 51479K [0x00000007b8000000, 0x00000007bd580000, 0x00000007bd580000)
  object space 87552K, 58% used [0x00000007b8000000,0x00000007bb245fe0,0x00000007bd580000)
 Metaspace       used 2652K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 287K, capacity 386K, committed 512K, reserved 1048576K

已使用 = 当前上届 - 下届； 可用空间 = 上届 - 当前上届
```

#### PringHeapAtGC GC前后打印堆的详细信息
```java
{Heap before GC invocations=1 (full 0):
 PSYoungGen      total 9728K, used 8376K [0x00000007bd580000, 0x00000007be000000, 0x00000007c0000000)
  eden space 8704K, 96% used [0x00000007bd580000,0x00000007bddae2f0,0x00000007bde00000)
  from space 1024K, 0% used [0x00000007bdf00000,0x00000007bdf00000,0x00000007be000000)
  to   space 1024K, 0% used [0x00000007bde00000,0x00000007bde00000,0x00000007bdf00000)
 ParOldGen       total 22016K, used 0K [0x00000007b8000000, 0x00000007b9580000, 0x00000007bd580000)
  object space 22016K, 0% used [0x00000007b8000000,0x00000007b8000000,0x00000007b9580000)
 Metaspace       used 2642K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 286K, capacity 386K, committed 512K, reserved 1048576K
Heap after GC invocations=1 (full 0):
 PSYoungGen      total 9728K, used 928K [0x00000007bd580000, 0x00000007be880000, 0x00000007c0000000)
  eden space 8704K, 0% used [0x00000007bd580000,0x00000007bd580000,0x00000007bde00000)
  from space 1024K, 90% used [0x00000007bde00000,0x00000007bdee8010,0x00000007bdf00000)
  to   space 1024K, 0% used [0x00000007be780000,0x00000007be780000,0x00000007be880000)
 ParOldGen       total 22016K, used 7168K [0x00000007b8000000, 0x00000007b9580000, 0x00000007bd580000)
  object space 22016K, 32% used [0x00000007b8000000,0x00000007b87000e0,0x00000007b9580000)
 Metaspace       used 2642K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 286K, capacity 386K, committed 512K, reserved 1048576K
}
```

#### PrintGCTimeStamps 打印每次GC时相对JVM启动时的时间偏移量，单位s
```java
0.096: [GC (Allocation Failure) [PSYoungGen: 8376K->896K(9728K)] 8376K->8072K(31744K), 0.0052318 secs] [Times: user=0.00 sys=0.01, real=0.00 secs] 
0.102: [GC (Allocation Failure) [PSYoungGen: 9258K->864K(18432K)] 16435K->16232K(40448K), 0.0181788 secs] [Times: user=0.01 sys=0.00, real=0.02 secs] 
0.120: [Full GC (Ergonomics) [PSYoungGen: 864K->0K(18432K)] [ParOldGen: 15368K->16149K(34816K)] 16232K->16149K(53248K), [Metaspace: 2642K->2642K(1056768K)], 0.0057548 secs] [Times: user=0.01 sys=0.00, real=0.01 secs] 
```

#### PrintGCApplicationConcurrentTime 打印的执行时间
```java
Application time: 0.0180333 seconds
Application time: 0.0005275 seconds
Application time: 0.0043159 seconds
```

#### PrintGCApplicationStoppedTime 应用程序因GC而停顿的时间
```java
Application time: 0.0151071 seconds
Total time for which application threads were stopped: 0.0028481 seconds, Stopping threads took: 0.0000087 seconds
Application time: 0.0006099 seconds
Total time for which application threads were stopped: 0.0065048 seconds, Stopping threads took: 0.0000072 seconds
Application time: 0.0038402 seconds
```
综合以上各个参数来看，注意GC线程和应用线程时可以并发执行的
```java
// app先运行了16.1108ms
0.074: Application time: 0.0161108 seconds
0.074: [GC (Allocation Failure) [PSYoungGen: 8376K->880K(9728K)] 8376K->8056K(31744K), 0.0031251 secs] [Times: user=0.01 sys=0.00, real=0.01 secs] 
// 0.077 - 0.074, 应用程序应为GC停顿了大约3ms
0.077: Total time for which application threads were stopped: 0.0032323 seconds, Stopping threads took: 0.0000088 seconds
0.078: Application time: 0.0005470 seconds
0.078: [GC (Allocation Failure) [PSYoungGen: 9242K->880K(18432K)] 16419K->16248K(40448K), 0.0030174 secs] [Times: user=0.00 sys=0.01, real=0.00 secs] 
0.081: [Full GC (Ergonomics) [PSYoungGen: 880K->0K(18432K)] [ParOldGen: 15368K->16149K(34816K)] 16248K->16149K(53248K), [Metaspace: 2642K->2642K(1056768K)], 0.0031848 secs] [Times: user=0.01 sys=0.00, real=0.00 secs] 
0.084: Total time for which application threads were stopped: 0.0063161 seconds, Stopping threads took: 0.0000056 seconds
0.088: Application time: 0.0045070 seconds
0.089: [GC (Allocation Failure) [PSYoungGen: 17075K->640K(18432K)] 33224K->32661K(53248K), 0.0067187 secs] [Times: user=0.01 sys=0.01, real=0.01 secs] 
0.095: [Full GC (Ergonomics) [PSYoungGen: 640K->0K(18432K)] [ParOldGen: 32021K->32534K(61952K)] 32661K->32534K(80384K), [Metaspace: 2642K->2642K(1056768K)], 0.0049105 secs] [Times: user=0.01 sys=0.00, real=0.00 secs] 
0.100: Total time for which application threads were stopped: 0.0117982 seconds, Stopping threads took: 0.0000108 seconds
0.102: Application time: 0.0013442 seconds
```
#### PrintReferenceGC  跟踪系统内的弱引用、软引用，Finallize队列

#### -Xloggc:gc输出文件

# 3. 类加载/卸载
`-verbose:class`跟踪类的加载和卸载；或者使用`-XX:+TraceClassLoading`跟踪类加载，`-XX:+TraceClassUnloading`跟踪类卸载。使用`ASM`生成一个类，并反复加载.
```java
[Opened /Library/Java/JavaVirtualMachines/jdk1.8.0_151.jdk/Contents/Home/jre/lib/rt.jar]
// Object是所有类的父类，所以先加载它
[Loaded java.lang.Object from /Library/Java/JavaVirtualMachines/jdk1.8.0_151.jdk/Contents/Home/jre/lib/rt.jar]
......
[Loaded Example from __JVM_DefineClass__]
[Loaded Example from __JVM_DefineClass__]
[Unloading class Example 0x00000007c006a028]
[Loaded Example from __JVM_DefineClass__]
[Unloading class Example 0x00000007c006a828]
[Loaded Example from __JVM_DefineClass__]
[Unloading class Example 0x00000007c006a028]
[Loaded Example from __JVM_DefineClass__]
[Unloading class Example 0x00000007c006a828]
[Loaded Example from __JVM_DefineClass__]
[Unloading class Example 0x00000007c006a028]
[Loaded Example from __JVM_DefineClass__]
[Unloading class Example 0x00000007c006a828]
[Loaded Example from __JVM_DefineClass__]
[Unloading class Example 0x00000007c006a028]
[Loaded Example from __JVM_DefineClass__]
[Unloading class Example 0x00000007c006a828]
[Loaded Example from __JVM_DefineClass__]
[Unloading class Example 0x00000007c006a028]
```
# 4. JVM启动参数
#### -XX:+PrintVMOptions 打印显式传递给JVM的参数

#### -XX:+PrintCommandLineFlags 打印显示和隐式（系统默认）的JVM参数

#### -XX:+PrintFlagsFinal 打印JVM全部参数的设定，500多个

# 5. Heap参数
#### [Java Available Memory](https://stackoverflow.com/questions/12807797/java-get-available-memory)
可用内存的两个概念：

* 操作系统分配给java application的内存
* JVM本身可以使用的内存

Runtime中的三个内存相关方法：

* `maxMemory()`: JVM最大可使用的内存(在堆上分配的内存，会被`-mx`或`-Xmx`限制)，理论上就是heap最大内存限制
* `totalMemory()`: 操作系统分配给java application的内存，比如设定的堆的初始大小20MB，已经分配了10MB，total memory可能就是20多MB
* `freeMemory()`: 相对于total memory而言的free memory， used = total - free

堆上可用内存 = 堆的最大内存 - 堆已使用内存 = 堆的最大内存 - (total memory - free memory) 

#### heap参数
几个重要的参数：

* -Xmx1024m：heap的max memory为1024MB
* -Xms512m: heap的初始大小
* -Xmn256m: 新生代的大小，一般设为整个堆大小的1/3或者1/4
* -XX:SurvivorRatio=eden/from = eden/to
* -XX:NewRatio=老年代/新生代
```java
// JVM规范中指明：方法区逻辑上属于heap的
                NewRatio = o / y
        |<---------- o ------------>|<---------- y --------->| 
                                    |<--------- Xmn -------->|
+-------+---------------------------+----------+------+------+
|       |                           |          |      |      |
| meta  |           Old             |   Eden   | from |  to  |
| space |                           |          |      |      |
|       |                           |          |      |      |
+-------+---------------------------+----------+------+------+
                                    |<--- e -->|<- f->|<-t ->|
                                     SurvivoRatio= e/f = e/t
```

#### OOM时dump headp

* -XX:+HeapDumpOnOutOfMemoryError: 当发生OOM时dump heap
* -XX:HeadDumpPath=path: dump的存储路径
* -XX:OnOutOfMemoryError=脚本： 当发生错误时执行的脚本

```java
// 假设ex.sh的内如如下
#!/bin/bash
echo "hello world!" >hello.txt

-Xmx20m -Xms20m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=a.dump -XX:OnOutOfMemoryError="sh ex.sh"
```
当发生oom时，会生成a.dump,同时会执行ex.sh