
**大纲**


**1\.动手模拟出频繁Young GC的场景**


**2\.JVM的Young GC日志应该怎么看**


**3\.代码模拟动态年龄判定规则进入老年代**


**4\.代码模拟S区放不下部分进入老年代**


**5\.JVM的Full GC日志应该怎么看**


**6\.问题汇总**


 


**1\.动手模拟出频繁Young GC的场景**


**(1\)程序的JVM参数示范**


**(2\)如何打印出JVM GC日志**


**(3\)示例程序代码**


**(4\)对象是如何分配在Eden区内的**


**(5\)采用指定JVM参数运行程序**


 


**(1\)程序的JVM参数示范**


平时我们系统运行创建的对象，通常都优先分配在新生代中的Eden区。除非是大对象，大对象会直接进入老年代或者大对象专属Region区域。新生代有两块S区，默认Eden区占新生代80%，每块S区占新生代10%。比如用以下JVM参数来运行代码：



```
 -XX:NewSize=5242880 -XX:MaxNewSize=5242880 
 -XX:InitialHeapSize=10485760 -XX:MaxHeapSize=10485760 
 -XX:SurvivorRatio=8 -XX:PretenureSizeThreshold=10485760 
 -XX:+UseParNewGC -XX:+UseConcMarkSweepGC
```

上述参数都是基于JDK 1\.8来设置的，不同的JDK版本对应的参数名称不太一样，但基本意思类似。



```
 -XX:InitialHeapSize和-XX:MaxHeapSize就是初始堆大小和最大堆大小；
 -XX:NewSize和-XX:MaxNewSize是初始新生代大小和最大新生代大小；
 -XX:PretenureSizeThreshold=10485760指定了大对象阈值是10M；
```

上面的JVM参数，相当于是给堆内存分配10M内存空间。其中新生代是5M内存空间，Eden区占4M，每个Survivor区占0\.5M。大对象必须超过10M才会直接进入老年代，年轻代使用ParNew垃圾回收器，老年代使用CMS垃圾回收器。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/ef5678e5c1844864adfdf4cd098ee1ce~tplv-obj.image?lk3s=ef143cfe&traceid=202501012231235488236301D55701100D&x-expires=2147483647&x-signature=IPj%2BbxC30C0PlGOX8cSL0bjmHnQ%3D)
**(2\)如何打印出JVM GC日志**


接着需要在系统的JVM参数中加入GC日志的打印选型，如下所示：



```
-XX:+PrintGCDetils：打印详细的GC日志；
-XX:+PrintGCTimeStamps：这个参数可以打印出每次GC发生的时间；
-Xloggc:gc.log：这个参数可以设置将GC日志写入一个磁盘文件；
```

加上打印GC日志参数后，JVM参数如下所示：



```
 -XX:NewSize=5242880 -XX:MaxNewSize=5242880 
 -XX:InitialHeapSize=10485760 -XX:MaxHeapSize=10485760 
 -XX:SurvivorRatio=8 -XX:PretenureSizeThreshold=10485760 
 -XX:+UseParNewGC -XX:+UseConcMarkSweepGC 
 -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -Xloggc:gc.log
```

**(3\)示例程序代码**



```
public class Demo1 {
    public static void main(String[] args) {
        byte[] array1 = new byte[1024 * 1024];
        array1 = new byte[1024 * 1024];
        array1 = new byte[1024 * 1024];
        array1 = null;
        byte[] array2 = new byte[2 * 1024 * 1024];
    }
}
```

**(4\)对象是如何分配在Eden区内的**


上述代码先通过new byte\[1024 \* 1024]连续分配3个数组，每个数组1M；然后通过array1这个局部变量依次引用这三个对象；最后还把array1这个局部变量指向了null。


 


那么在JVM中上述代码是如何运行的呢？


一.首先来看第一行代码：byte\[] array1 \= new byte\[1024 \* 1024];


这行代码一旦运行，就会在JVM的Eden区内放入一个1M的对象，同时在main线程的虚拟机栈中会压入一个main()方法的栈帧。在main()方法的栈帧内部，会有一个名为array1的局部变量。这个array1局部变量指向了堆内存Eden区的那个1M的数组，如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/e20711730e414b1d9b5f1ad535790968~tplv-obj.image?lk3s=ef143cfe&traceid=202501012231235488236301D55701100D&x-expires=2147483647&x-signature=Y6SU%2FxC7kGCc3M9hDVrnAEu9ZMg%3D)
二.接着看第二行代码：array1 \= new byte\[1024 \* 1024];


这行代码会在Eden区创建第二个数组，并让array1变量指向第二个数组。然后第一个数组就没被引用了，成了垃圾对象。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/59ce7a8918924e1f8d2c9d7971698472~tplv-obj.image?lk3s=ef143cfe&traceid=202501012231235488236301D55701100D&x-expires=2147483647&x-signature=0LcFeEFlQkCHa3LQzVXFJ2CgQfo%3D)
三.然后看第三行代码：array1 \= new byte\[1024 \* 1024];


这行代码会在Eden区创建第三个数组，并让array1变量指向第三个数组。此时前面两个数组都没有被引用了，都成了垃圾对象，如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/800b128998b942f0a7dc759057f46a04~tplv-obj.image?lk3s=ef143cfe&traceid=202501012231235488236301D55701100D&x-expires=2147483647&x-signature=jJSoIUyvfvzYvzYxf1EgvhFo%2FY0%3D)
四.接着看第四行代码：array1 \= null;


这行代码一执行，就会让array1局部变量什么都不指向了，此时会导致之前创建的3个数组全部变成垃圾对象。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/21f9d6e181f34534a8ec1cd43a6141b4~tplv-obj.image?lk3s=ef143cfe&traceid=202501012231235488236301D55701100D&x-expires=2147483647&x-signature=zdXiU1EXVcjMJr7xYqB4Qisa74Y%3D)
五.最后看第五行代码：byte\[] array2 \= new byte\[2 \* 1024 \* 1024];


此时会分配一个2M大小的数组，尝试放入Eden区中。这时Eden区明显已经不能放下这个数组了，因为Eden区总共4M，里面已经放入3个1M的数组，剩余空间只有1M。此时再放一个2M的数组是放不下的，所以这个时候就会触发年轻代的Young GC；


 


**(5\)采用指定JVM参数运行程序**


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/640567e62b9a4de58efe59c3ffc1fa8a~tplv-obj.image?lk3s=ef143cfe&traceid=202501012231235488236301D55701100D&x-expires=2147483647&x-signature=JR%2FrtRfq3cfE34i8o8civnU2UrU%3D)
![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/8542e92edf734a929df419aed43b41b4~tplv-obj.image?lk3s=ef143cfe&traceid=202501012231235488236301D55701100D&x-expires=2147483647&x-signature=Itm4SUkm28xZEtx1qBhafjIbNDw%3D)
![](https://p26-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/19a32bb45a0c4b09b00edf3b1244df18~tplv-obj.image?lk3s=ef143cfe&traceid=202501012231235488236301D55701100D&x-expires=2147483647&x-signature=0w3%2FSjpkrymmo9jtEXruk%2Fk3EIQ%3D)
![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/baa55e84abbf454c94aaedda25d64fa2~tplv-obj.image?lk3s=ef143cfe&traceid=202501012231235488236301D55701100D&x-expires=2147483647&x-signature=opFefhG%2Fdm4prI438UWCEEq6S%2Fc%3D)
![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/8adecf1531e141108ed96b83b6849304~tplv-obj.image?lk3s=ef143cfe&traceid=202501012231235488236301D55701100D&x-expires=2147483647&x-signature=xTtiQTTiBmLTvjEyqoY0cqZkwYo%3D)
然后点击运行即可。运行完毕后，会在工程目录中出现一个gc.log文件，gc.log文件里面就是本次程序运行的gc日志。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/b44ed020906c49339d01610a6df06953~tplv-obj.image?lk3s=ef143cfe&traceid=202501012231235488236301D55701100D&x-expires=2147483647&x-signature=mjz0k8HsftUzSp2Ot6Wlo6sPaJY%3D)
打开gc.log文件，会看到如下所示的gc日志：



```
CommandLine flags: -XX:InitialHeapSize=10485760 -XX:MaxHeapSize=10485760 -XX:MaxNewSize=5242880 
  -XX:NewSize=5242880 -XX:OldPLABSize=16 -XX:PretenureSizeThreshold=10485760 -XX:+PrintGC 
  -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:SurvivorRatio=8 -XX:+UseCompressedClassPointers 
  -XX:+UseCompressedOops -XX:+UseConcMarkSweepGC -XX:+UseParNewGC 
0.268: [GC (Allocation Failure) 0.269: [ParNew: 4030K->512K(4608K), 0.0015734 secs] 4030K->574K(9728K), 0.0017518 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
Heap
 par new generation   total 4608K, used 2601K [0x00000000ff600000, 0x00000000ffb00000, 0x00000000ffb00000)
  eden space 4096K,  51% used [0x00000000ff600000, 0x00000000ff80a558, 0x00000000ffa00000)
  from space 512K, 100% used [0x00000000ffa80000, 0x00000000ffb00000, 0x00000000ffb00000)
  to   space 512K,   0% used [0x00000000ffa00000, 0x00000000ffa00000, 0x00000000ffa80000)
 concurrent mark-sweep generation total 5120K, used 62K [0x00000000ffb00000, 0x0000000100000000, 0x0000000100000000)
 Metaspace       used 2782K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 300K, capacity 386K, committed 512K, reserved 1048576K
```

 


**2\.JVM的Young GC日志应该怎么看**


**(1\)程序运行采用的默认JVM参数如何查看**


**(2\)一次GC的概要说明**


**(3\)图解GC执行过程**


**(4\)GC过后的堆内存使用情况**


**(5\)Metaspace中的capacity、committed、reserved**


 


**(1\)程序运行采用的默认JVM参数如何查看**


在GC日志中，可以看到如下内容：



```
CommandLine flags: -XX:InitialHeapSize=10485760 -XX:MaxHeapSize=10485760 
  -XX:MaxNewSize=5242880 -XX:NewSize=5242880 -XX:OldPLABSize=16 
  -XX:PretenureSizeThreshold=10485760 -XX:+PrintGC -XX:+PrintGCDetails 
  -XX:+PrintGCTimeStamps -XX:SurvivorRatio=8 -XX:+UseCompressedClassPointers 
  -XX:+UseCompressedOops -XX:+UseConcMarkSweepGC -XX:+UseParNewGC
```

这展示了这次运行程序采取的JVM参数，包括设置的和默认的参数。


 


所以如果没有设置JVM参数，应该怎么看系统默认使用的JVM参数？


 


方法就是：给JVM加一段打印GC日志的参数，这样在GC日志里就可以看到默认给JVM进程分配多大的内存空间了。


 


**(2\)一次GC的概要说明**


接着看GC日志中的如下一行：该行日志概要说明了本次GC的执行情况。



```
0.268: [GC (Allocation Failure) 0.269: [ParNew: 4030K->512K(4608K), 0.0015734 secs] 4030K->574K(9728K), 0.0017518 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
```

一.为什么会发生一次GC


从上图可知，因为最后要分配一个2M的数组，结果Eden区内存不够。所以就出现了GC (Allocation Failure)，也就是对象分配失败，所以此时就要触发一次Young GC。


 


二.这次GC是什么时候发生的


很简单，看一个数字——0\.268，这指的是系统运行后过了多少秒就发生本次GC。比如这里的日志就是系统运行后过了268毫秒就发生了本次GC。


 


三.新生代GC使用的是ParNew


ParNew: 4030K\-\>512K(4608K)。ParNew的意思是：由于这次触发的是Young GC，于是用指定的ParNew垃圾回收器执行。


 


四.GC前新生代使用空间、GC后存活对象、新生代可用空间


ParNew: 4030K\-\>512K(4608K)。这个代表的意思是：新生代可用空间是4608K，也就是4\.5M。为什么新生代可用空间是4608K？因为Eden区是4M，两个S区只有一个可放存活对象，另一个要保持空闲。所以新生代的可用空间就是Eden区 \+ 1个S区的大小，即4608K \= 4\.5M。


 


然后4030K\-\>512K，表示的是对新生代执行了一次GC。GC前已经使用了4030K，但是GC后只有512K的对象存活下来。


 


五.本次新生代GC消耗时间，单位精确到微妙


0\.0015734 secs。这个就是本次新生代GC耗费的时间，大概耗费1\.5ms，回收3M的对象。


 


六.GC前和GC后的Java堆内存使用了多少


4030K\-\>574K(9728K), 0\.0017518 secs。这个代表的是整个Java堆内存的情况，整个Java堆内存的总可用空间是9728K \= 新生代4\.5M \+ 老年代5M。GC前整个Java堆内存使用了4030K，GC后Java堆内存使用了574K。


 


七.本次GC消耗时间，单位到精确到10毫秒


\[Times: user\=0\.00 sys\=0\.00, real\=0\.00 secs]。这个代表的是本次GC消耗的时间，这里最小单位是小数点之后两位。这里全部是0\.00 secs，也就是本次GC就耗费了几毫秒，所以是0\.00s。


 


**(3\)图解GC执行过程**


**第一：看这行日志**


ParNew: 4030K\-\>512K(4608K), 0\.0015734 secs。在GC前，明明在Eden区只放了3个1M的数组，大小一共是3M\=3072K。那么GC前新生代应该是使用了3072K，为什么显示使用了4030K内存？


 


对于这个问题，需要明白两点：


一.虽然创建的数组是1M，但为了存储它，JVM还会附带一些其他信息。所以每个数组实际占用的内存是大于1M的。


 


二.除了自己创建的对象以外，可能还有一些看不见的对象在Eden区。至于这些看不见的未知对象是什么，可通过专门的工具分析堆内存快照。


 


所以如下图示：GC前三个数组和其他一些未知对象加起来，就是占据了4030K的内存。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/2c0592b546634ba0b71f290603f806f5~tplv-obj.image?lk3s=ef143cfe&traceid=202501012231235488236301D55701100D&x-expires=2147483647&x-signature=KrB%2FzqCxFAGoDTGy71Ogt1IUWZw%3D)
接着想要在Eden区分配一个2M的数组，此时就会触发Allocation Failure。Allocation Failure表示对象分配失败，于是触发Young GC。然后使用ParNew垃圾回收器进行垃圾回收，回收掉之前创建的三个数组。因为此时这三个数组都没被引用，而成为垃圾对象了。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/783427a160b34eb187651c96e439c2ad~tplv-obj.image?lk3s=ef143cfe&traceid=202501012231235488236301D55701100D&x-expires=2147483647&x-signature=0mSv1cCvH2XqoU8pcpRQw1cGScQ%3D)
**第二：看这行日志**



```
ParNew: 4030K->512K(4608K), 0.0015734 secs
```

新生代GC回收后，新生代中已使用的内存从4030K降低到了512K。也就是说这次YGC有512K的对象存活下来，从Eden区转移到了S区。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/2af727808435470593505ff42398352c~tplv-obj.image?lk3s=ef143cfe&traceid=202501012231235488236301D55701100D&x-expires=2147483647&x-signature=z1mW3ifeDBZo9P2gdev8sPb88TY%3D)
以上就是本次GC的全过程。


 


**(4\)GC过后的堆内存使用情况**


接着看下面的GC日志，这段日志是在JVM退出时打印出来的当前堆内存的使用情况。



```
Heap
 par new generation total 4608K, used 2601K [0x00000000ff600000, 0x00000000ffb00000, 0x00000000ffb00000)
  eden space 4096K,  51% used [0x00000000ff600000, 0x00000000ff80a558, 0x00000000ffa00000)
  from space 512K, 100% used [0x00000000ffa80000, 0x00000000ffb00000, 0x00000000ffb00000)
  to   space 512K,   0% used [0x00000000ffa00000, 0x00000000ffa00000, 0x00000000ffa80000)
 concurrent mark-sweep generation total 5120K, used 62K [0x00000000ffb00000, 0x0000000100000000, 0x0000000100000000)
 Metaspace       used 2782K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 300K, capacity 386K, committed 512K, reserved 1048576K
```

一.当前新生代总共可用内存和已经使用内存



```
par new generation total 4608K, used 2601K
```

这代表ParNew垃圾回收器负责的新生代总共有4608K(4\.5M)可用内存，目前已经使用了的内存是2601K(2\.5M)。此时在JVM退出前，为什么新生代占用了2\.5M的内存？


 


因为在GC后，会通过如下代码又分配一个2M的数组：byte\[] array2 \= new byte\[2 \* 1024 \* 1024];


 


所以此时在Eden区中一定会有一个2M的数组，也就是2048K。然后上次GC后在Survivor From区中存活了512K的对象。由于每个数组会额外占据一些内存来存放一些自己这个对象的元数据，所以可以认为多出来的41K是数组对象额外使用的内存空间。因此GC后新生代占用的大小是：2048K \+ 512K \+ 41K \= 2601K。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/0ad60d2a26984eacaf511eef2f55ed4b~tplv-obj.image?lk3s=ef143cfe&traceid=202501012231235488236301D55701100D&x-expires=2147483647&x-signature=%2BH0GDxEfs31EGqaVxALqJAbw3es%3D)
二.接着看以下GC日志



```
 eden space 4096K,  51% used [0x00000000ff600000, 0x00000000ff80a558, 0x00000000ffa00000)
 from space 512K, 100% used [0x00000000ffa80000, 0x00000000ffb00000, 0x00000000ffb00000)
 to   space 512K,   0% used [0x00000000ffa00000, 0x00000000ffa00000, 0x00000000ffa80000)
```

其中清晰地表示，此时：Eden区4M的内存被使用了51%，因为有一个2M的数组在里面。然后Survivor From区，显示512K，100%的使用率。也就是Survivor From区被这次GC后存活下来的512K对象全占据了。


 


三.接着看以下GC日志


如下日志指的是CMS管理的老年代内存空间一共5M，此时已用62K。



```
concurrent mark-sweep generation total 5120K, used 62K，
```

而下面两段日志的意思是：Metaspace元数据空间和Class空间的总容量、使用内存等，Metaspace元数据空间会存放一些类的信息、常量池信息等。



```
Metaspace       used 2782K, capacity 4486K, committed 4864K, reserved 1056768K
 class space    used 300K, capacity 386K, committed 512K, reserved 1048576K
```

**(5\)Metaspace中的capacity、committed、reserved**


Java8取消了永久代PermGen，取而代之的是元数据区MetaSpace。方法区在Java8以后移至元数据区MetaSpace，JDK8开始把类的元数据放到本地内存(Native Heap)，称为MetaSpace。理论上本地内存剩余多少，MetaSpace就可以有多大。


 


当然我们不可能无限制增大MetaSpace，需要用\-XX:MaxMetaSpaceSize指定MetaSpace大小。


 


关于used capacity commited和reserved：MetaSpace由一个或多个Virtual Space(虚拟空间)组成。虚拟空间是操作系统的连续存储空间，虚拟空间是按需分配的。分配时，虚拟空间会向OS预留(reserve)空间，但还没被提交(commit)。


 


一.MetaSpace的预留空间(reserved)是全部虚拟空间的大小


虚拟空间的最小分配单元是MetaChunk(也可以说是Chunk)，当新的Chunk被分配至虚拟空间时，与Chunk相关的内存空间会被提交。


 


二.MetaSpace的committed指的是所有Chunk占有的空间


每个Chunk占据空间不同，当一个类加载器(Class Loader)被GC时：所有与之关联的Chunk被释放(freed)，这些释放的Chunk会维护在一个全局的释放数组里。


 


三.MetaSpace的capacity指的是所有未被释放的Chunk占据的空间


假如从GC日志发现committed是4864K，capacity4486K。说明有一部分的Chunk已经被释放了，代表有类加载器被回收了。附上原文链接：



```
https://stackoverflow.com/questions/40891433/understanding-metaspace-line-in-jvm-heap-printout
```

 


**3\.代码模拟动态年龄判定规则进入老年代**


**(1\)动态年龄判定规则**


**(2\)动态年龄判定规则的示例代码**


**(3\)示例代码运行后产生的GC日志**


**(4\)GC日志分析**


**(5\)完善示例代码**


**(6\)分析最终版的GC日志**


**(7\)总结**


 


**(1\)动态年龄判定规则**


**对象进入老年代的四个常见时机如下：**


一.躲过15次新生代GC后(年龄达到15岁)


二.动态年龄判定规则


如果在S区内，年龄1\+年龄2\+年龄3\+年龄n的对象总和大于S区的50%。此时年龄n及以上的对象会进入老年代，不一定需要达到15岁。所以动态年龄判断规则有个推论：如果S区中的同龄对象大小超过S区内存的一半，就要直接升入老年代。


三.如果一次YGC后存活对象太多无法放入S区，就会直接放入老年代


四.大对象直接进入老年代


 


首先通过代码模拟出最常见的一种进入老年代的情况：如果S区内年龄1 \+ 年龄2 \+ 年龄3 \+ 年龄n的对象总和大于S区的50%，此时年龄n及以上的对象会进入老年代，也就是动态年龄判定规则。


 


示例程序的JVM参数如下：



```
 -XX:NewSize=10485760 -XX:MaxNewSize=10485760 
 -XX:InitialHeapSize=20971520 -XX:MaxHeapSize=20971520 
 -XX:SurvivorRatio=8  -XX:MaxTenuringThreshold=15 
 -XX:PretenureSizeThreshold=10485760 
 -XX:+UseParNewGC -XX:+UseConcMarkSweepGC 
 -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -Xloggc:gc.log
```

在这些参数里需要注意几点：


Java堆的总大小通过\-XX:InitialHeapSize设置为20M，新生代通过\-XX:NewSize设置为10M，所以老年代是10M。然后通过\-XX:SurvivorRatio参数可知，Eden区是8M，每个S区是1M。接着大对象必须超过10M才会直接进入老年代，\-XX:MaxTenuringThreshold\=15设置对象年龄达到15岁会进入老年代。


 


一切准备就绪，先看当前的内存分配情况：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/bc1b80f760714a52bbbf8b23ff115285~tplv-obj.image?lk3s=ef143cfe&traceid=202501012231235488236301D55701100D&x-expires=2147483647&x-signature=YsQAo0lxR7n%2BfTi8j122dSz3KSc%3D)
**(2\)动态年龄判定规则的示例代码**



```
public class Demo {
    public static void main(String[] args) {
        byte[] array1 = new byte[2 * 1024 * 1024];
        array1 = new byte[2 * 1024 * 1024];
        array1 = new byte[2 * 1024 * 1024];
        array1 = null;
        byte[] array2 = new byte[128 * 1024];
        byte[] array3 = new byte[2 * 1024 * 1024];
    }
}
```

接下来运行示例代码，然后通过打印出的GC日志分析上述代码执行后JVM中的对象分配情况。


 


**(3\)示例代码运行后产生的GC日志**


把上述示例代码以及给出的JVM参数配合起来运行，此时会看到如下的GC日志。



```
CommandLine flags: -XX:InitialHeapSize=20971520 -XX:MaxHeapSize=20971520 -XX:MaxNewSize=10485760 
  -XX:MaxTenuringThreshold=15 -XX:NewSize=10485760 -XX:OldPLABSize=16 
  -XX:PretenureSizeThreshold=10485760 -XX:+PrintGC -XX:+PrintGCDetails 
  -XX:+PrintGCTimeStamps -XX:SurvivorRatio=8 -XX:+UseCompressedClassPointers 
  -XX:+UseCompressedOops -XX:+UseConcMarkSweepGC -XX:+UseParNewGC 
0.297: [GC (Allocation Failure) 0.297: [ParNew: 7260K->715K(9216K), 0.0012641 secs] 7260K->715K(19456K), 0.0015046 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
Heap
 par new generation   total 9216K, used 2845K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  eden space 8192K,  26% used [0x00000000fec00000, 0x00000000fee14930, 0x00000000ff400000)
  from space 1024K,  69% used [0x00000000ff500000, 0x00000000ff5b2e10, 0x00000000ff600000)
  to   space 1024K,   0% used [0x00000000ff400000, 0x00000000ff400000, 0x00000000ff500000)
 concurrent mark-sweep generation total 10240K, used 0K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
Metaspace       used 2782K, capacity 4486K, committed 4864K, reserved 1056768K
 class space    used 300K, capacity 386K, committed 512K, reserved 1048576K
```

**(4\)GC日志分析**


一.首先看如下几行代码



```
byte[] array1 = new byte[2 * 1024 * 1024];
array1 = new byte[2 * 1024 * 1024];
array1 = new byte[2 * 1024 * 1024];
array1 = null;
```

这里连续创建了3个2M的数组，最后还把局部变量array1设置为null，所以此时的内存如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/00b89485e55a4ba7bcb0ecb4edfbfe40~tplv-obj.image?lk3s=ef143cfe&traceid=202501012231235488236301D55701100D&x-expires=2147483647&x-signature=WWHP%2FjxcQbeQ4rAn1kIEUo93HUg%3D)
二.接着执行下面这行代码：



```
byte[] array2 = new byte[128 * 1024];
```

此时会在Eden区创建一个128K的数组同时由array2局部变量来引用，如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/3c2dff8d012444b9811192a8693e56d5~tplv-obj.image?lk3s=ef143cfe&traceid=202501012231235488236301D55701100D&x-expires=2147483647&x-signature=mlrOG75LAhqxq%2B7f4Fnz8JWTSKU%3D)
三.然后会执行下面的代码：



```
byte[] array3 = new byte[2 * 1024 * 1024];
```

此时希望在Eden区再次分配一个2M的数组，由于此时Eden区里已有3个2M数组和1个128K数组，大小都超过6M了。Eden区总共才8M，此时是不可能在Eden区再次分配一个2M的数组的，因此一定会触发一次Young GC。


 


四.接着开始看GC日志：



```
ParNew: 7260K->715K(9216K), 0.0012641 secs
```

这行日志清晰表明，在GC前新生代占用了7260K的内存。大概就是6M的3个数组 \+ 128K的1个数组 \+ 几百K的未知对象 \= 7260K，如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/2e09b0aea9e743d4b9c8979f8b16fb77~tplv-obj.image?lk3s=ef143cfe&traceid=202501012231235488236301D55701100D&x-expires=2147483647&x-signature=cAmMqS9rUsVeV%2FS2KSsJB2OGxCM%3D)
五.接着看回上述GC日志：



```
7260K->715K(9216K)
```

这表明，一次Young GC过后，剩余的存活对象是715K。由于新生代刚开始会有512K左右的未知对象，此时再加上128K的数组，差不多就是715K。


 


六.接着看如下GC日志：



```
par new generation   total 9216K, used 2845K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
 eden space 8192K,  26% used [0x00000000fec00000, 0x00000000fee14930, 0x00000000ff400000)
 from space 1024K,  69% used [0x00000000ff500000, 0x00000000ff5b2e10, 0x00000000ff600000)
 to   space 1024K,   0% used [0x00000000ff400000, 0x00000000ff400000, 0x00000000ff500000)
concurrent mark-sweep generation total 10240K, used 0K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
```

从上面的日志可以清晰看出：此时From Survivor区域被占据了69%的内存，大概就是700K左右。这就是一次Young GC后存活下来的对象，它们都进入From Survivor区。


 


同时Eden区域内被占据了26%的空间，大概就是2M左右。这就是执行代码"byte\[] array3 \= new byte\[2 \* 1024 \* 1024]"时，在Young GC后分配在Eden区内的数组。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/34c64377f531409daeb9a2dfa3a68cf8~tplv-obj.image?lk3s=ef143cfe&traceid=202501012231235488236301D55701100D&x-expires=2147483647&x-signature=NJE7EG0FC3vRELF7N4pzVtxoJIE%3D)
现在Survivor From区里的那700K对象，是1岁。它们熬过一次GC，年龄就会增长1岁。此时S区总大小是1M，存活对象已经有700K了，已经超过了50%。


 


**(5\)完善示例代码**



```
public class Demo {
    public static void main(String[] args) {
        byte[] array1 = new byte[2 * 1024 * 1024];
        array1 = new byte[2 * 1024 * 1024];
        array1 = new byte[2 * 1024 * 1024];
        array1 = null;
        byte[] array2 = new byte[128 * 1024];
        byte[] array3 = new byte[2 * 1024 * 1024];
        
        array3 = new byte[2 * 1024 * 1024];
        array3 = new byte[2 * 1024 * 1024];
        array3 = new byte[128 * 1024];
        array3 = null;
        
        byte[] array4 = new byte[2 * 1024 * 1024];
    }
}
```

把示例代码给完善一下变成上述的样子，接下来要触发第二次YGC，然后看S区内的动态年龄判定规则能否生效。


 


一.接着前面代码执行的分析，继续看如下代码：



```
array3 = new byte[2 * 1024 * 1024];
array3 = new byte[2 * 1024 * 1024];
array3 = new byte[128 * 1024];
array3 = null;
```

这几行代码运行后，会接着分配2个2M的数组。然后再分配一个128K的数组，最后让array3变量指向null。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/4a3b6aaed85745df9a0eff7ec868cd22~tplv-obj.image?lk3s=ef143cfe&traceid=202501012231235488236301D55701100D&x-expires=2147483647&x-signature=HOnymk%2BnMKJJgz86NmnZpz4Ghbw%3D)
二.此时接着会运行下面的代码：



```
byte[] array4 = new byte[2 * 1024 * 1024];
```

这时会发现，Eden区如果要再放一个2M数组进去，是放不下的，所以此时会触发一次YGC。使用上述JVM参数运行这段程序会看到如下GC日志：



```
0.269: [GC (Allocation Failure) 0.269: [ParNew: 7260K->713K(9216K), 0.0013103 secs] 7260K->713K(19456K), 0.0015501 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
0.271: [GC (Allocation Failure) 0.271: [ParNew: 7017K->0K(9216K), 0.0036521 secs] 7017K->700K(19456K), 0.0037342 secs] [Times: user=0.06 sys=0.00, real=0.00 secs]
Heap
par new generation   total 9216K, used 2212K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
 eden space 8192K,  27% used [0x00000000fec00000, 0x00000000fee290e0, 0x00000000ff400000)
 from space 1024K,   0% used [0x00000000ff400000, 0x00000000ff400000, 0x00000000ff500000)
 to   space 1024K,   0% used [0x00000000ff500000, 0x00000000ff500000, 0x00000000ff600000)
concurrent mark-sweep generation total 10240K, used 700K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
Metaspace       used 2782K, capacity 4486K, committed 4864K, reserved 1056768K
 class space    used 300K, capacity 386K, committed 512K, reserved 1048576K
```

**(6\)分析最终版的GC日志**


一.首先第一次GC的日志如下：



```
0.269: [GC (Allocation Failure) 0.269: [ParNew: 7260K->713K(9216K), 0.0013103 secs] 7260K->713K(19456K), 0.0015501 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
```

这个过程前面已经分析过了。


 


二.接着第二次GC的日志如下：



```
0.271: [GC (Allocation Failure) 0.271: [ParNew: 7017K->0K(9216K), 0.0036521 secs] 7017K->700K(19456K), 0.0037342 secs] [Times: user=0.06 sys=0.00, real=0.00 secs] 
```

第二次触发Yuong GC，就是第一次赋值给局部变量array4的时候。此时的日志"ParNew: 7017K\-\>0K(9216K)"表明：这次GC过后，新生代直接就没有对象了。但array2这个局部变量还一直引用一个128K的数组，它是存活对象。那么这128K的数组以及还有那500多K的未知对象，此时都去哪里了？


 


首先在Eden区里的3个2M的数组和1个128K的数组，肯定会被回收掉的。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/b6e2c7c4c7a3468386bbc84a89edf7b3~tplv-obj.image?lk3s=ef143cfe&traceid=202501012231235488236301D55701100D&x-expires=2147483647&x-signature=JWiaX%2FzfVYgZ6JPX9yLN4mZMFdE%3D)
然后发现S区中的对象都是存活的，且总大小超过50%以及年龄都是1岁。于是根据动态年龄判定规则：年龄1\+...年龄n的对象总大小超S区50%，年龄n及以上的对象进老年代。由于此时S区里的对象的年龄都是1，所以会全部直接进入老年代了。


 


S区的对象第一次YGC进来时已超50%，但在第二次YGC还存活才升代。所以不是进入S区的时候使用动态年龄去判断，而是扫描S区时才去判断。


 


如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/768dd1cfb4f34792b60731569a165c5d~tplv-obj.image?lk3s=ef143cfe&traceid=202501012231235488236301D55701100D&x-expires=2147483647&x-signature=y%2FDBekOBCa0eWQUpb49g65cQL6o%3D)
这个可以从下面的日志进行确认：



```
concurrent mark-sweep generation total 10240K, used 700K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
```

CMS管理的老年代，此时使用空间刚好是700K。由此证明此时Survivor的对象触发了动态年龄判定规则，虽然没达到15岁，但全部进入老年代了。


 


三.然后array4变量引用的2M的数组，此时就会分配到Eden区中，如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/3fe4b3e7c310476292f6417a333ee083~tplv-obj.image?lk3s=ef143cfe&traceid=202501012231235488236301D55701100D&x-expires=2147483647&x-signature=X%2Bw0eGLJFiKYmicQBh%2F6tm4HMn8%3D)
看如下日志：



```
eden space 8192K,  27% used [0x00000000fec00000, 0x00000000fee290e0, 0x00000000ff400000)
```

这就说明Eden区当前就是有一个2M的数组，然后再看下面的日志：



```
from space 1024K,   0% used [0x00000000ff400000, 0x00000000ff400000, 0x00000000ff500000)
to   space 1024K,   0% used [0x00000000ff500000, 0x00000000ff500000, 0x00000000ff600000)
```

表示两个Survivor区域都是空的，因为之前存活的700K对象都进入老年代了，所以现在Survivor区都空了。


 


**(7\)总结**


这里分析了对象是如何通过动态年龄判定规则进入老年代的。如果每次YGC过后存活的对象太多，特别是超过了S区50%的空间。那么下次YGC时就会触发动态年龄判定规则让一些对象进入老年代中。


 


注意：不是进入S区的时候就用动态年龄去判断，而是扫描S区时才判断。


 


**4\.代码模拟S区放不下部分进入老年代**


**(1\)示例代码**


**(2\)GC日志**


**(3\)分析GC日志**


**(4\)总结**


 


**(1\)示例代码**



```
public class Demo {
    public static void main(String[] args) {
        byte[] array1 = new byte[2 * 1024 * 1024];
        array1 = new byte[2 * 1024 * 1024];
        array1 = new byte[2 * 1024 * 1024];
        
        byte[] array2 = new byte[128 * 1024];
        array2 = null;
        
        byte[] array3 = new byte[2 * 1024 * 1024];
    }
}
```

**(2\)GC日志**


使用之前的JVM参数来跑一下上面的程序，可以看到下面的GC日志：



```
0.421: [GC (Allocation Failure) 0.421: [ParNew: 7260K->573K(9216K), 0.0024098 secs] 7260K->2623K(19456K), 0.0026802 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
Heap
 par new generation   total 9216K, used 2703K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  eden space 8192K,  26% used [0x00000000fec00000, 0x00000000fee14930, 0x00000000ff400000)
  from space 1024K,  55% used [0x00000000ff500000, 0x00000000ff58f570, 0x00000000ff600000)
  to   space 1024K,   0% used [0x00000000ff400000, 0x00000000ff400000, 0x00000000ff500000)
 concurrent mark-sweep generation total 10240K, used 2050K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
Metaspace       used 2782K, capacity 4486K, committed 4864K, reserved 1056768K
 class space    used 300K, capacity 386K, committed 512K, reserved 1048576K 
```

**(3\)分析GC日志**


一.首先看如下几行代码：



```
byte[] array1 = new byte[2 * 1024 * 1024];
array1 = new byte[2 * 1024 * 1024];
array1 = new byte[2 * 1024 * 1024];


byte[] array2 = new byte[128 * 1024];
array2 = null;
```

上面的代码中：首先分配了3个2M的数组，然后让array1变量指向第三个2M数组。接着创建了一个128K的数组，并让array2指向了null。同时我们一直都知道，Eden区里会有500K左右的未知对象，如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/fd9aa01339e34a97a7748659917f2678~tplv-obj.image?lk3s=ef143cfe&traceid=202501012231235488236301D55701100D&x-expires=2147483647&x-signature=bvtadfwHG89mrmJ6hV9Tom3VkQo%3D)
二.接着会执行如下代码：



```
byte[] array3 = new byte[2 * 1024 * 1024];
```

此时想要在Eden区里再创建一个2M的数组，肯定是不行的，所以此时必然会触发一次Young GC。


 


先看如下日志：



```
ParNew: 7260K->573K(9216K), 0.0024098 secs
```

这里清晰说明了，本次GC过后，新生代里就剩下了500多K的对象了。这是为什么呢，此时明明array1变量是引用了一个2M的数组的。


 


因为这次GC时，会回收掉上图中的2个2M的数组和1个128K的数组，然后留下一个2M的数组和1个未知的500K的对象作为存活对象。这时存活下来的2M数组和500K未知对象是不能放入Survivor区的，因为Survivor区只有1M。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/552ffcedb59d4fbca9268270b836ccd0~tplv-obj.image?lk3s=ef143cfe&traceid=202501012231235488236301D55701100D&x-expires=2147483647&x-signature=jyzfRJGe7lQk15GXVQHDMvY4Bjc%3D)
三.根据对象进入老年代规则，此时是否要把全部存活对象都放入老年代


也不是，因为首先根据如下日志：



```
eden space 8192K,  26% used [0x00000000fec00000, 0x00000000fee14930, 0x00000000ff400000)
```

可知，Eden区内一定放入了一个新的2M的数组，而且这个数组就是刚才最后想要分配的那个数组，由array3变量引用。此时会如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/dfb82514ef2e453dbede767a5eae0bd9~tplv-obj.image?lk3s=ef143cfe&traceid=202501012231235488236301D55701100D&x-expires=2147483647&x-signature=OMJ4zAyJ3V5Z98zcNq8hg8JhuGU%3D)
然后再根据如下的日志：



```
from space 1024K,  55% used [0x00000000ff500000, 0x00000000ff58f570, 0x00000000ff600000)
```

可知，此时Survivor From区中有500K对象，即那500K未知对象。所以新生代GC后会存活2M数组和500K未知对象，放不进Survivor区。但也不会让这2M数组和500K未知对象全部都进入老年代，而是会把500K的未知对象先放入Survivor From区中。


 


所以结合GC日志，可以清晰的看到：在这种情况下，是会把部分对象放入Survivor区的，如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/9e1278d18c354478bd5795ba0ff59b11~tplv-obj.image?lk3s=ef143cfe&traceid=202501012231235488236301D55701100D&x-expires=2147483647&x-signature=G4nKLuUyZ5G02Zq8qDGJlxfonCY%3D)
接着再根据如下日志：



```
concurrent mark-sweep generation total 10240K, used 2050K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
```

可以发现此时老年代里会有2M的数组，因此可以认为：YGC过后，发现存活下来的对象有2M的数组和500K的未知对象。此时会把500K的未知对象放入Survivor区，把2M的数组放入老年代。如下图示：


![](https://p26-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/3923c91d20e14685aff144cbbbe48b54~tplv-obj.image?lk3s=ef143cfe&traceid=202501012231235488236301D55701100D&x-expires=2147483647&x-signature=ke8WkD9G4UuzJxiKgX8GgQGPviU%3D)
**(4\)总结**


这里展示了YGC后存活对象放不下S区，部分对象会进入老年代的例子。这种场景下，会有部分对象留在Survivor中，有部分对象进入老年代中。


 


**5\.JVM的Full GC日志应该怎么看**


**(1\)示例代码**


**(2\)GC日志**


**(3\)分析日志**


**(4\)总结**


 


**(1\)示例代码**



```
public class Demo {
    public static void main(String[] args) {
        byte[] array1 = new byte[4 * 1024 * 1024];
        array1 = null;
        
        byte[] array2 = new byte[2 * 1024 * 1024];
        byte[] array3 = new byte[2 * 1024 * 1024];
        byte[] array4 = new byte[2 * 1024 * 1024];
        byte[] array5 = new byte[128 * 1024];
        byte[] array6 = new byte[2 * 1024 * 1024];
    }
}
```

**(2\)GC日志**


接下来需要采用如下参数来运行上述程序：



```
 -XX:NewSize=10485760 -XX:MaxNewSize=10485760
 -XX:InitialHeapSize=20971520 -XX:MaxHeapSize=20971520
 -XX:SurvivorRatio=8  -XX:MaxTenuringThreshold=15 
 -XX:PretenureSizeThreshold=3145728 
 -XX:+UseParNewGC -XX:+UseConcMarkSweepGC 
 -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -Xloggc:gc.log
```

这里最关键一个参数，就是\-XX:PretenureSizeThreshold\=3145728。该参数设置大对象阈值为3M，即对象大小超过3M就直接进入老年代。


 


运行之后会得到如下GC日志：



```
0.308: [GC (Allocation Failure) 0.308: [ParNew (promotion failed): 7260K->7970K(9216K), 0.0048975 secs]0.314: [CMS: 8194K->6836K(10240K), 0.0049920 secs] 11356K->6836K(19456K), [Metaspace: 2776K->2776K(1056768K)], 0.0106074 secs] [Times: user=0.00 sys=0.00, real=0.01 secs]
Heap
 par new generation   total 9216K, used 2130K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  eden space 8192K,  26% used [0x00000000fec00000, 0x00000000fee14930, 0x00000000ff400000)
  from space 1024K,   0% used [0x00000000ff500000, 0x00000000ff500000, 0x00000000ff600000)
  to   space 1024K,   0% used [0x00000000ff400000, 0x00000000ff400000, 0x00000000ff500000)
 concurrent mark-sweep generation total 10240K, used 6836K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
Metaspace       used 2782K, capacity 4486K, committed 4864K, reserved 1056768K
 class space    used 300K, capacity 386K, committed 512K, reserved 1048576K
```

**(3\)分析日志**


一.首先看如下代码：



```
byte[] array1 = new byte[4 * 1024 * 1024];
array1 = null;
```

这行代码直接分配了一个4M的大对象，此时这个对象会直接进入老年代，接着array1不再引用这个对象。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/74148e6fff4a40de94308fa8d8d2e5c7~tplv-obj.image?lk3s=ef143cfe&traceid=202501012231235488236301D55701100D&x-expires=2147483647&x-signature=4NoRaZg4Fap6fI0CPZQsqIG%2BNxQ%3D)
二.接着看下面的代码：



```
byte[] array2 = new byte[2 * 1024 * 1024];
byte[] array3 = new byte[2 * 1024 * 1024];
byte[] array4 = new byte[2 * 1024 * 1024];
byte[] array5 = new byte[128 * 1024];
```

连续分配了4个数组，其中3个是2M的数组，1个是128K的数组。如下图示，全部都会进入Eden区。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/d3ae919bbb33497b991207a85ad282b2~tplv-obj.image?lk3s=ef143cfe&traceid=202501012231235488236301D55701100D&x-expires=2147483647&x-signature=aneNcA5i1oxIanabbQEwtxG2%2BU4%3D)
三.接着会执行如下代码：



```
byte[] array6 = new byte[2 * 1024 * 1024];
```

此时新生代就放不下2M的对象了，因为Eden区已经不够空间了，所以会触发一次Young GC。可以参考下面的GC日志：



```
ParNew (promotion failed): 7260K->7970K(9216K), 0.0048975 secs
```

这行日志就显示了：Eden区原来是有7260K对象，但是回收后发现一个都回收不掉，因为上述几个数组都还被main()方法栈的局部变量引用。


 


所以此时就会直接把这些对象都放入到老年代里，但是现在老年代里已经有一个4M的数组了，此时老年代已经放不下3个2M的数组和1个128K的数组了。参考如下GC日志：



```
[CMS: 8194K->6836K(10240K), 0.0049920 secs] 11356K->6836K(19456K), [Metaspace: 2776K->2776K(1056768K)], 0.0106074 secs]
```

可以清晰看到，此时执行了CMS垃圾回收器的Full GC。Full GC会对老年代进行GC，同时一般会跟一次新生代GC关联，以及触发一次元数据区(永久代)的GC。


 


由于在CMS的Full GC之前，就已经触发过Young GC了。所以Young GC已经有了，接着就是执行针对老年代的Old GC。也就是如下日志：



```
CMS: 8194K->6836K(10240K), 0.0049920 secs
```

这里可以看到老年代从8M左右的对象占用，变成了6M左右的对象占用。这个过程具体如下：


 


第一：在完成Young GC之后，先把2个2M的数组放入到老年代。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/372aa294264a45bfb39df5fabd8bda34~tplv-obj.image?lk3s=ef143cfe&traceid=202501012231235488236301D55701100D&x-expires=2147483647&x-signature=aqsPb%2BIelyrpwdW8R34Sc%2F3RXvc%3D)
第二：如果继续往老年代放入1个2M数组和1个128K数组，则一定放不下。因此这时就会触发CMS的Full GC，然后就会回收掉老年代中的一个4M的数组，因为它已经没被引用了。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/39cf11251636415d9efa923013d88938~tplv-obj.image?lk3s=ef143cfe&traceid=202501012231235488236301D55701100D&x-expires=2147483647&x-signature=9eYMXqmtnX%2FqUOMutvjDWZudZJ4%3D)
第三：接着往老年代放入1个2M的数组和1个128K的数组。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/a7674252a08d4045a59b9e3f2ac17ec2~tplv-obj.image?lk3s=ef143cfe&traceid=202501012231235488236301D55701100D&x-expires=2147483647&x-signature=zF87qSTWO4SxDBEBS5at%2FkqFqVA%3D)
所以可以看到如下的CMS垃圾回收日志：



```
CMS: 8194K->6836K(10240K), 0.0049920 secs
```

老年代从Full GC回收前的8M变成了6M，就是上图所示。


 


第四：最后当CMS执行Full GC完毕后，新生代的对象都进入了老年代。此时最后一行代码要在新生代分配2M的数组就可以成功了。如下图示：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/dae1a256b5394854a35906a024f54d9c~tplv-obj.image?lk3s=ef143cfe&traceid=202501012231235488236301D55701100D&x-expires=2147483647&x-signature=hWUUKCrBUMjfTImtYk3VLr%2B4WPM%3D)
**(4\)总结**


这里介绍了一个触发老年代GC的案例：就是新生代存活对象太多，老年代都放不下了，就会触发CMS的FGC。


 


**(5\)触发老年代GC的其他场景**


一.执行YGC后存活对象太多，老年代逐个放不下后会触发老年代GC


二.执行YGC前老年代可用空间小于历次YGC升入老年代对象平均大小，于是就会在执行YGC前，提前触发老年代GC


三.老年代使用率已经达到了92%的阈值，也会触发老年代GC


 


**6\.问题汇总**


**问题一：**


**JVM优化思路总结**


阶段一：项目上线初期


一.上线前，根据预期的QPS、平均每个请求或者任务的内存需求大小等。评估出需要使用几台机器来承载，每台机器需要什么样的配置。


二.根据系统的请求或者任务处理速度，评估内存使用情况。然后合理分配Eden区、Survivor区、老年代的内存大小。


 


JVM调优的总体原则就是：


让短命对象在YGC就被回收，不要进入老年代。让长期存活的对象尽早进入老年代，不要在新生代复制来复制去。对系统响应时间敏感且内存需求大的，建议采用G1回收器。


 


如何合理分配各个区域：


一.根据内存增速来评估多久进行YGC


二.根据每次YGC会有多少存活对象来评估S区的大小设置是否合理


三.评估多久会进行一次FGC\+产生的STW是否可接受


 


阶段二：项目运营出色，系统负载增加了100倍


方案1：增加服务器数量


根据系统负载的增比，同比增加机器数量，机器配置和JVM的配置可以保持不变。


 


方案2：使用更高配置的机器


更高的配置，意味着更快速的处理速度和更大的内存。响应时间敏感且内存需求大的使用G1回收器，这时候需要和项目上线初期一样，合理地使用配置和分配内存。


 


**问题二：**


G1存不存在类似ParNew \+ CMS频繁回收导致的系统变慢问题？


答：G1可能会频繁回收，但它每次回收时间可控，所以不会对系统造成太大影响。


 


 本博客参考[MeoMiao 萌喵加速](https://biqumo.org)。转载请注明出处！
