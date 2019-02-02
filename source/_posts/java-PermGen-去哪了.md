---
title: java PermGen 去哪了
date: 2016-05-14 15:42:45
categories:
- jvm

tags: [java,jvm]


---
### Java PermGen 去哪里了?

[原文链接](http://www.infoq.com/articles/Java-PERMGEN-Removed)：原文作者：Monica Beckwith  以下为本人翻译，仅用于交流学习，版权归原作者和InfoQ所有，转载注明出处，请不要用于商业用途

在Java虚拟机(JVM)内部，class文件中包括类的版本、字段、方法、接口等描述信息，还有运行时常量池，用于存放编译器生成的各种字面量和符号引用。

在过去（自定义类加载器还不是很常见的时候），类大多是”static”的，很少被卸载或收集，因此被称为“永久的(Permanent)”。同时，由于类class是JVM实现的一部分，并不是由应用创建的，所以又被认为是“非堆(non-heap)”内存。

<!-- more -->
在JDK8之前的HotSpot JVM，存放这些”永久的”的区域叫做“永久代(permanent generation)”。永久代是一片连续的堆空间，在JVM启动之前通过在命令行设置参数-XX:MaxPermSize来设定永久代最大可分配的内存空间，默认大小是64M（64位JVM由于指针膨胀，默认是85M）。永久代的垃圾收集是和老年代(old generation)捆绑在一起的，因此无论谁满了，都会触发永久代和老年代的垃圾收集。不过，一个明显的问题是，当JVM加载的类信息容量超过了参数-XX：MaxPermSize设定的值时，应用将会报OOM的错误(对于这句话，译者的理解是：32位的JVM默认MaxPermSize是64M，而JDK8里的Metaspace，也可以通过参数-XX:MetaspaceSize 和-XX:MaxMetaspaceSize设定大小，但如果不指定MaxMetaspaceSize的话，Metaspace的大小仅受限于native memory的剩余大小。也就是说永久代的最大空间一定得有个指定值，而如果MaxPermSize指定不当，就会OOM)。

注：在JDK7之前的版本，对于HopSpot JVM，interned-strings存储在永久代（又名PermGen），会导致大量的性能问题和OOM错误。从PermGen移除interned strings的更多信息查看[这里](http://bugs.java.com/view_bug.do?bug_id=6962931)。

译者注：从JDK7开始永久代的移除工作，贮存在永久代的一部分数据已经转移到了Java Heap或者是Native Heap。但永久代仍然存在于JDK7，并没有完全的移除：符号引用(Symbols)转移到了native heap;字面量(interned strings)转移到了java heap;类的静态变量(class statics)转移到了java heap。

在JDK7 update 4即随后的版本中，提供了完整的支持对于Garbage-First(G1)垃圾收集器，以取代在JDK5中发布的CMS收集器。使用G1，PermGen仅仅在FullGC（stop-the-word,STW）时才会被收集。G1仅仅在PermGen满了或者应用分配内存的速度比G1并发垃圾收集速度快的时候才触发FullGC。

而对于CMS收集器，通过开启布尔参数-XX:+CMSClassUnloadingEnabled来并发对PermGen进行收集。对于G1没有类似的选项，G1只能通过FullGC，stop the world,来对PermGen进行收集。

永久代在JDK8中被完全的移除了。所以永久代的参数-XX:PermSize和-XX：MaxPermSize也被移除了。

在JDK8中,classe metadata(the virtual machines internal presentation of Java class),被存储在叫做Metaspace的native memory。一些新的flags被加入：
-XX:MetaspaceSize，class metadata的初始空间配额，以bytes为单位，达到该值就会触发垃圾收集进行类型卸载，同时GC会对该值进行调整：如果释放了大量的空间，就适当的降低该值；如果释放了很少的空间，那么在不超过MaxMetaspaceSize（如果设置了的话），适当的提高该值。
-XX：MaxMetaspaceSize，可以为class metadata分配的最大空间。默认是没有限制的。
-XX：MinMetaspaceFreeRatio,在GC之后，最小的Metaspace剩余空间容量的百分比，减少为class metadata分配空间导致的垃圾收集
-XX:MaxMetaspaceFreeRatio,在GC之后，最大的Metaspace剩余空间容量的百分比，减少为class metadata释放空间导致的垃圾收集

默认情况下，class metadata的分配仅受限于可用的native memory总量。可以使用MaxMetaspaceSize来限制可为class metadata分配的最大内存。当class metadata的使用的内存达到MetaspaceSize(32位clientVM默认12Mbytes,32位ServerVM默认是16Mbytes)时就会对死亡的类加载器和类进行垃圾收集。设置MetaspaceSize为一个较高的值可以推迟垃圾收集的发生。

Native Heap，就是C-Heap。对于32位的JVM，C-Heap的容量=4G-Java Heap-PermGen；对于64位的JVM，C-Heap的容量=物理服务器的总RAM+虚拟内存-Java Heap-PermGen

这里科普下，在Windows下称为虚拟内存(virtual memory),在Linux下称为交换空间(swap space),用于当系统需要更多的内存资源而物理内存已经满了的情况下，将物理内存中不活跃的页转移到磁盘上的交换空间中。

在JDK8，Native Memory，包括Metaspace和C-Heap。

IBM的J9和Oracle的JRockit(收购BEA公司的JVM)都没有永久代的概念，而Oracle移除HotSpot中的永久代的原因之一是为了与JRockit合并，以充分利用各自的特点。

### <a name="t1" style="text-decoration: underline; outline: none; color: rgb(0, 161, 158);"></a>再见，再见PermGen，你好Metaspace

随着JDK8的到来，JVM不再有PermGen。但类的元数据信息（metadata）还在，只不过不再是存储在连续的堆空间上，而是移动到叫做“Metaspace”的本地内存（Native memory）中。

类的元数据信息转移到Metaspace的原因是PermGen很难调整。PermGen中类的元数据信息在每次FullGC的时候可能会被收集，但成绩很难令人满意。而且应该为PermGen分配多大的空间很难确定，因为PermSize的大小依赖于很多因素，比如JVM加载的class的总数，常量池的大小，方法的大小等。

此外，在HotSpot中的每个垃圾收集器需要专门的代码来处理存储在PermGen中的类的元数据信息。从PermGen分离类的元数据信息到Metaspace,由于Metaspace的分配具有和Java Heap相同的地址空间，因此Metaspace和Java Heap可以无缝的管理，而且简化了FullGC的过程，以至将来可以并行的对元数据信息进行垃圾收集，而没有GC暂停。
![](http://i.imgur.com/WS0hb1g.jpg)

### <a name="t2" style="text-decoration: underline; outline: none; color: rgb(0, 161, 158);"></a>永久代的移除对最终用户意味着什么？

由于类的元数据可以在本地内存(native memory)之外分配,所以其最大可利用空间是整个系统内存的可用空间。这样，你将不再会遇到OOM错误，溢出的内存会涌入到交换空间。最终用户可以为类元数据指定最大可利用的本地内存空间，JVM也可以增加本地内存空间来满足类元数据信息的存储。

注：永久代的移除并不意味者类加载器泄露的问题就没有了。因此，你仍然需要监控你的消费和计划，因为内存泄露会耗尽整个本地内存，导致内存交换(swapping)，这样只会变得更糟。

### <a name="t3" style="text-decoration: underline; outline: none; color: rgb(0, 161, 158);"></a>移动到Metaspace和它的内存分配

Metaspace VM利用内存管理技术来管理Metaspace。这使得由不同的垃圾收集器来处理类元数据的工作，现在仅仅由Metaspace VM在Metaspace中通过C++来进行管理。Metaspace背后的一个思想是，类和它的元数据的生命周期是和它的类加载器的生命周期一致的。也就是说，只要类的类加载器是存活的，在Metaspace中的类元数据也是存活的，不能被释放。

之前我们不严格的使用这个术语“Metaspace”。更正式的，每个类加载器存储区叫做“a metaspace”。这些metaspaces一起总体称为”the Metaspace”。仅仅当类加载器不在存活，被垃圾收集器声明死亡后，该类加载器对应的metaspace空间才可以回收。Metaspace空间没有迁移和压缩。但是元数据会被扫描是否存在Java引用。

Metaspace VM使用一个块分配器(chunking allocator)来管理Metaspace空间的内存分配。块的大小依赖于类加载器的类型。其中有一个全局的可使用的块列表（a global free list of chunks）。当类加载器需要一个块的时候，类加载器从全局块列表中取出一个块，添加到它自己维护的块列表中。当类加载器死亡，它的块将会被释放，归还给全局的块列表。块（chunk）会进一步被划分成blocks,每个block存储一个元数据单元(a unit of metadata)。Chunk中Blocks的分配线性的（pointer bump）。这些chunks被分配在内存映射空间(memory mapped(mmapped) spaces)之外。在一个全局的虚拟内存映射空间（global virtual mmapped spaces）的链表，当任何虚拟空间变为空时，就将该虚拟空间归还回操作系统。
![](http://i.imgur.com/JFH75Lz.jpg)

上面这幅图展示了Metaspace使用metachunks在mmapeded virual spaces分配的情形。类加载器1和3描述的是反射或匿名类加载器，使用“特定的”chunk尺寸。类加载器2和4使用小还是中等的chunk尺寸取决于加载的类数量。

### <a name="t4" style="text-decoration: underline; outline: none; color: rgb(0, 161, 158);"></a>Metaspace大小的调整和可以使用的工具

正如前面提到了，Metaspace VM管理Metaspace空间的增长。但有时你会想通过在命令行显示的设置参数-XX:MaxMetaspaceSize来限制Metaspace空间的增长。默认情况下，-XX:MaxMetaspaceSize并没有限制，因此，在技术上，Metaspace的尺寸可以增长到交换空间，而你的本地内存分配将会失败。

对于64位的服务器端JVM，-XX：MetaspaceSize的默认大小是21M。这是初始的限制值(the initial high watermark)。一旦达到这个限制值，FullGC将会被触发进行类卸载(当这些类的类加载器不再存活时)，然后这个high watermark被重置。新的high watermark的值依赖于空余Metaspace的容量。如果没有足够的空间被释放，high watermark的值将会上升；如果释放了大量的空间，那么high watermark的值将会下降。如果初始的watermark设置的太低，这个过程将会进行多次。你可以通过垃圾收集日志来显示的查看这个垃圾收集的过程。所以，一般建议在命令行设置一个较大的值给XX:MetaspaceSize来避免初始时的垃圾收集。

每次垃圾收集之后，Metaspace VM会自动的调整high watermark，推迟下一次对Metaspace的垃圾收集。

这两个参数，-XX：MinMetaspaceFreeRatio和-XX：MaxMetaspaceFreeRatio,类似于GC的FreeRatio参数，可以放在命令行。

针对Metaspace，JDK自带的一些工具做了修改来展示Metaspace的信息：

*   **jmap -clstats **:打印类加载器的统计信息(取代了在JDK8之前打印类加载器信息的permstat)。一个例子的输出当运行DaCapo’s Avrora基准测试：

```
$ jmap -clstats 
Attaching to process ID 6476, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.5-b02
finding class loader instances ..done.
computing per loader stat ..done.
please wait.. computing liveness.liveness analysis may be inaccurate ...
class_loader classes bytes parent_loader alive? type 
<bootstrap\> 655 1222734 null live <internal> 
0x000000074004a6c0000x000000074004a708dead java/util/ResourceBundle$RBClassLoader@0x00000007c0053e20
0x000000074004a76000 null dead sun/misc/Launcher$ExtClassLoader@0x00000007c002d248 0x00000007401189c8 1 1471
0x00000007400752f8dead sun/reflect/DelegatingClassLoader@0x00000007c0009870 0x000000074004a708116 3160530x000000074004a760 dead sun/misc/Launcher$AppClassLoader@0x00000007c0038190 
0x00000007400752f8538 7738540x000000074004a708 dead org/dacapo/harness/DacapoClassLoader@0x00000007c00638b0 
total = 6 1310 2314112 N/A alive=1, dead=5 N/A
```

*   **jstat -gc **:Metaspace的信息也会被打印出来，如下面的例子所示：
    ![](http://i.imgur.com/oEE3iA5.jpg)
*   **jcmd GC.class_stats**:这是一个新的诊断命令，可以使用户连接到存活的JVM，转储Java类元数据的详细统计。

注：在JDK8 build 13下，需要开启参数-XX：+UnlockDiagnosticVMOptions

```
$ jcmd  help GC.class_stats
9522:
GC.class_stats 
Provide statistics about Java class meta data. Requires -XX:+UnlockDiagnosticVMOptions. 
Impact: High: Depends on Java heap size and content. 
Syntax : GC.class_stats [options] [<columns>] 
Arguments: 
  columns : [optional] Comma-separated list of all the columns to show. If not specified, the following columns are shown: InstBytes,KlassBytes,CpAll,annotations,MethodCount,Bytecodes,MethodAll,ROAll,RWAll,Total (STRING, no default value) 
Options: (options must be specified using the <key> or <key>=<value> syntax) 
  -all : [optional] Show all columns (BOOLEAN, false) 
  -csv : [optional] Print in CSV (comma-separated values) format for spreadsheets (BOOLEAN, false) 
  -help : [optional] Show meaning of all the columns (BOOLEAN, false
```

注：对于列的更多信息，请查看[这里](https://bugs.openjdk.java.net/secure/attachment/11600/ver_010_help.txt)。
一个输出列子：

```
$ jcmd  GC.class_stats 
7140:
Index Super InstBytes KlassBytes annotations CpAll MethodCount Bytecodes MethodAll ROAll RWAll Total ClassName 
1 -1 426416 480 0 0 0 0 0 24 576 600 [C 
2 -1 290136 480 0 0 0 0 0 40 576 616 [Lavrora.arch.legacy.LegacyInstr; 
3 -1 269840 480 0 0 0 0 0 24 576 600 [B 
4 43 137856 648 0 19248 129 4886 25288 16368 30568 46936 java.lang.Class 
5 43 136968 624 0 8760 94 4570 33616 12072 32000 44072 java.lang.String 
6 43 75872 560 0 1296 7 149 1400 880 2680 3560 java.util.HashMap$Node 
7 836 57408 608 0 720 3 69 1480 528 2488 3016 avrora.sim.util.MulticastFSMProbe 
8 43 55488 504 0 680 1 31 440 280 1536 1816 avrora.sim.FiniteStateMachine$State 
9 -1 53712 480 0 0 0 0 0 24 576 600 [Ljava.lang.Object; 
10 -1 49424 480 0 0 0 0 0 24 576 600 [I 
11 -1 49248 480 0 0 0 0 0 24 576 600 [Lavrora.sim.platform.ExternalFlash$Page; 
12 -1 24400 480 0 0 0 0 0 32 576 608 [Ljava.util.HashMap$Node; 
13 394 21408 520 0 600 3 33 1216 432 2080 2512 avrora.sim.AtmelInterpreter$IORegBehavior 
14 727 19800 672 0 968 4 71 1240 664 2472 3136 avrora.arch.legacy.LegacyInstr$MOVW 
…<snipped> 
…<snipped> 
1299 1300 0 608 0 256 1 5 152 104 1024 1128 sun.util.resources.LocaleNamesBundle 
1300 1098 0 608 0 1744 10 290 1808 1176 3208 4384 sun.util.resources.OpenListResourceBundle 
1301 1098 0 616 0 2184 12 395 2200 1480 3800 5280 sun.util.resources.ParallelListResourceBundle 
 2244312 794288 2024 2260976 12801 561882 3135144 1906688 4684704 6591392 Total 
 34.0% 12.1% 0.0% 34.3% - 8.5% 47.6% 28.9% 71.1% 100.0% 
Index Super InstBytes KlassBytes annotations CpAll MethodCount Bytecodes MethodAll ROAll RWAll Total ClassName

```
### 当前的问题

先前提到的，Metaspace VM使用块分配器(chunking allocator)。chunk的大小取决于类加载器的类型。由于类class并没有一个固定的尺寸，这就存在这样一种可能：可分配的chunk的尺寸和需要的chunk的尺寸不相等，这就会导致内存碎片。Metaspace VM还没有使用压缩技术，所以内存碎片是现在的一个主要关注的问题。
![](http://i.imgur.com/5424Rjo.jpg)

[原文地址](http://ifeve.com/java-permgen-removed/)




