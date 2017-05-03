---
title: 垃圾回收/TomcatGC参数配置
date: 2017-05-03
tags: JVM
categories: JAVA
---

# 垃圾回收
垃圾回收有很多种算法，如引用计数法，标记压缩法、复制算法、分带、分区的思想。

### 引用计数法
这是一个比较古老而经典的垃圾回收算法，其核心是在对象被其他所引用时计数器加1，而当引用失效时则减一，但这种方法有非常严重的问题，无法处理循环引用的问题，还有就是每次进行加减操作比较浪费系统性能。

### 标记清除法
就是分别标记和清除两个阶段进行处理内存中的对象，当然这种方式也有很大的弊端，就是空间碎片问题，垃圾回收后的空间不是连续的，不连续的内存空间的工作效率要低于连续的内存空间。

### 复制算法（新生代）
其核心思想将内存空间分为两块，每次只使用其中一块，在垃圾回收时将正在使用的内存中的残留对象复制到未被使用的内存块中，之后去清除之前正在使用的内存块中所有的对象，反复去交换两个内存的角色，完成垃圾回收。（java中新生代的from和to空间就是使用了这个算法）

### 标记压缩法（老年代）
标记压缩法在标记清除法基础上做了优化，把存活的对象压缩到内存的一端，而后进行垃圾清理。

### 为什么新生代和老年代使用不同的算法？
新生代对象被回收的几率要大于老年代中对象被回收的概率，老年代中的对象都较为稳定。采用不同的算法是为了性能优化考虑。

### 分代算法
根据**对象的特点**即将内存分为N块，而后根据内存的特点使用不同的算法。
对于新生代和老生代来说，新生代回收频率较高，但是每次回收耗时都很短，而老年代回收频率较低，但是耗时会相对较长，所以应该尽量减少老年代的GC。

### 分区算法
分区算法：其主要就是将整个内存分为N多个独立空间，每个小空间都可以独立使用，这样细粒度的控制一次回收多个小空间而不是针对整个空间进行GC，从而提升性能，并减少GC的停顿时间。

#### 垃圾回收时的停顿现象
垃圾回收的任务是识别和回收垃圾对象进行内存清理，为了让垃圾回收器可以更高效的执行，大部分情况下，会要求系统进如一个停顿的状态。停顿的目的是为了终止所有的应用线程，只有这样的系统才不会有新垃圾的产生。同时停顿保证了系统状态在某一个瞬间的一致性，也有利于更好的标记垃圾对象。因此在垃圾回收时，都会产生应用程序的停顿。

### 对象如何进入老年代
一般而言对象首次创建会被放置在新生代的eden区，如果没有GC介入，则对象不会离开eden区，那么eden区的对象如何进入老年代那呢？一般来说只要对象的年龄达到一定的大小，就会自动离开年轻代进入老年代，对象年龄是由对象经历数次GC决定的，在新生代每次GC后，如果没有被回收则年龄加1.虚拟机提供一个参数来控制新生代对象的最大年龄，当超过这个年龄范围就会晋升老年代
-XX:MaxTenuringThreshold

```
public class Test5 {

	public static void main(String[] args) {
		//初始对象在eden区
		//-Xmx64m -Xms64m -XX:+PrintGCDetails
		/*for(int i=0;i<5;i++){
			byte[] b = new byte[1024*1024];
		}*/
		
		//测试进入老年代的对象
		//-Xmx1024m -Xms1024m -XX:+UseSerialGC -XX:MaxTenuringThreshold=15 -XX:+PrintGCDetails
		//-XX:+PrintHeapATGC
		for(int k=0;k<20;k++){
			for(int j=0;j<300;j++){
				byte[] b=new byte[1024*1024];
			}
		}
		
	}
}

```
**总结**：根据设置MaxTenuringThreshold参数可以指定新生代对象经过多少次回收后进入老年代。另外，大对象(新生代eden区无法装入时，也会直接进入老年代)。JVM有个参数可以设置对象的大小超过指定的大小之后，直接晋升老年代。
-XX:PretenureSizeThreshold

### TLAB
TLAB全称是Thread Local Allocation Buffer即本地分配缓存，从名字上看是一个线程专用的内存分配区域，是为了加速对象分配而生的。每一个线程会产生一个TLAB，该线程独享的一个工作区域，java虚拟机使用这种TLAB来避免多线程冲突问题。提高了对象分配的效率。TLAB空间一般不会太大，当大对象无法在TLAB分配时，则会直接分配在堆上。
-XX:+UseTLAB 使用TLAB
-XX:+TLABSize设置TLAB大小
-XX:TLABRefillWasterFraction 设置维护进入TLAB空间的单个对象大小，它是一个比例值，默认64，即如果对象大于整个空间的1/64,则在堆创建对象。
-XX:+PrintTLAB查看TLAB信息
-XX:ResizeTLAB自调整TLABRefillWasteFraction阈值。
**每个线程单独的一块工作内存(volatile)**

```
public class Test7 {
	
	public static void alloc(){
		byte[] b = new byte[2];
	}
	
	public static void main(String[] args) {
		//TLAB分配
		//-XX:+UseTLAB -XX:+PrintTLAB -XX:TLABSize=102400 -XX:TLABRefillWasteFraction=100 -XX:-DoEscapeAnalysis
		long start = System.currentTimeMillis();
		for(int i=0;i<10000000;i++){
			alloc();
		}
		long end = System.currentTimeMillis();
		System.out.println(end-start);
	}

}

```

### 对象的创建流程图
![](http://i1.piimg.com/567571/3947acc73907f3f5.png)

### 垃圾收集器
在java虚拟机中，垃圾回收器不仅仅只有一种，什么情况下该使用哪种，对性能又有什么影响，这些都是我们需要了解的。

-  串行垃圾收集器
串行回收器是指使用单线程进行垃圾回收的回收器。每次回收时，串行回收器只有一个工作线程，对于并行能力较弱的计算机来说，串行回收器的专注性和独占性往往有更好的性能表现。串行回收器可以在新生代和老年代使用。根据作用于不同的对空间分为新生代串行回收器和老年代串行回收器。
-XX：+UseSerialGC参数可以设置使用新生代串行回收器和老年代串行回收器。

- 并行垃圾收集器
并行的垃圾回收器在串行回收器的基础上做了改进，他可以使用多个线程同时进行垃圾回收，对于计算能力强的计算机而言，可以有效的缩短垃圾回收所需的实际时间。

ParNew回收器是一个工作在**新生代**的垃圾回收器，他只是简单的将串行回收器多线程化，它的回收策略和算法和串行回收器一样。
使用-XX:+UseParNewGC 新生代使用ParNew回收器，老年代则使用串行回收器。
ParNew回收器工作时的线程数量可以使用-XX：ParallelGCThreads参数指定，一般最好和计算机的CPU相当，避免过多的线程影响性能。

新生代ParallelGC回收器，使用了复制算法的回收器，也是多线程独占形式的回收器，但ParallelGC回收器有一个很重要的特点，就是它非常关注吞吐量。
提供了两个参数控制系统的吞吐量
-XX: MaxGCPauseMillis:设置最大垃圾收集停顿时间，可用于把虚拟机在GC停顿的时间控制在MaxGCPauseMillis范围内，如果希望减少GC停顿时间可以将MaxGCPauseMillis设置的很小，但是会导致GC频繁，从而增加GC的总时间，降低了吞吐量。所以要根据实际情况设置该值。
-XX: GCTimeRatio:设置吞吐量的大小，它是一个0到100之间的整数，默认情况下它的取值是99，那么系统将花费不超过1/(1+n)的时间用于垃圾回收，也就是1/(1+99)=1%的时间。
另外还可以指定-XX：+UseAdaptiveSizePolicy打开自适应模式，在这种模式下，新生代的大小、eden、from/to的比例，以及晋升老年代的对象的年龄参数将被自动调整，以达到在堆大小、吞吐量和停顿时间之间的平衡点。

老年代ParallelOldGC回收器也是一种多线程的回收器，和新生代的ParallelGC回收器一样，也是一种关注吞吐量的回收器，它使用标记压缩算法实现。
-XX:+UseParallelOldGC进行设置
-XX:+ParallelGCThreads也可以设置垃圾收集时的线程数量。

- CMS回收器

CMS全称为：Current Mark Sweep意为并发标记清除，他使用的是标记清除法，主要关注系统的停顿时间。
使用-XX:+UseConcMarkSweepGC进行设置
使用-XX:+ConcGCThreads设置并发线程数量

CMS并不是独占的回收器，也就是说CMS回收的过程中，应用程序仍在不断的运行，又会有新的垃圾不断的产生，所以在使用CMS的过程中应该确保应用程序的内存足够用。CMS不会等到应用程序饱和的时候才去回收垃圾，而是在某一阈值的时候开始回收，回收阈值可以用指定的参数进行配置，-XX:CMSInitiatingOccupancyFraction来指定，默认值为68，也就是说当老年代的空间使用率达到68%的时候，会执行CMS回收。如果内存使用率增长很快，在CMS执行的过程中，已经出现内存不足的情况，此时CMS回收就会失败，虚拟机将启动老年代串行回收器进行垃圾回收，这会使应用程序中断，直到垃圾回收完成后才会正常工作。这个过程GC停顿的时间可能比较长，所以-XX:CMSInitiatingOccupancyFraction的设置要根据实际情况。

之前我们在学习算法的时候说过，标记清除法有个缺点就是内存碎片的问题，那么CMS有个参数设置-XX:+UseCMSCompactAtFullCollection可以使CMS回收完成之后进行一次碎片整理，-XX:CMSFullGCsBeforeCompaction参数可以设置进行多少次CMS回收之后对内存进行一次压缩。

- G1回收器
G1回收器(Garbage-First)是在jdk1.7中提出的垃圾回收器，从长期目标来看是为了取代CMS回收器，G1回收器拥有独特的垃圾回收策略，G1属于分代垃圾回收器，区分新生代和老年代，依然有eden和from/to区,并不要求整个eden区或新生代、老年代空间都连续，它使用的是分区算法。
**并行性：** G1回收期间可多线程同时工作。
**并发性：** G1拥有与应用程序交替执行的能力，部分工作可与应用程序同时执行，在整个GC期间不会完全阻塞应用程序。
**分代GC：** G1依然是一个分代收集器，但是它是兼顾新生代和老年代一起工作的，之前的垃圾收集器或者在新生代工作或者在老年代工作，因此这是一个很大的不同。
**空间整理：** G1在回收过程中，不会像CMS那样在若干次GC后进行碎片整理，G1采用有效复制对象的方式减少空间碎片。
可预见性：由于分区的原因，G1可以只选取部分区域进行回收，缩小了回收的范围，提高了性能。
使用-XX:+UseG1GC应用G1收集器
使用-XX:MaxGCPauseMillis指定最大的停顿时间
使用-XX:ParallelGCThreads设置并行回收的线程数量

**配置Tomcat参数（一）**
使用串行垃圾回收器
```
-XX:+PrintGCDetails -Xmx32m -Xms32m
-XX:+HeapDumpOnOutOfMemoryError
-XX:+UseSerialGC
-XX:PermSize=32m
```
测试结果显示吞吐量为:871.8/sec 87KB/sec

**配置Tomcat参数（二）**
扩大内存提高系统性能
```
-XX:+PrintGCDetails -Xmx512m -Xms32m
-XX:+HeapDumpOnOutOfMemoryError
-XX:+UseSerialGC
-XX:PermSize=32m
-Xloggc:d:/gc.log
```
测试结果显示吞吐量为:1383.31/sec 138KB/sec

**配置Tomcat参数（三）**
调整初始堆大小
```
-XX:+PrintGCDetails -Xmx512m -Xms128m
-XX:+HeapDumpOnOutOfMemoryError
-XX:+UseSerialGC
-XX:PermSize=32m
-Xloggc:d:/gc.log
```
测试结果显示吞吐量为:1501.3/sec 149.8KB/sec

**配置Tomcat参数（四）**
测试ParNew回收器的表现
```
-XX:+PrintGCDetails -Xmx512m -Xms128m
-XX:+HeapDumpOnOutOfMemoryError
-XX:+UseParNewGC
-XX:PermSize=32m
-Xloggc:d:/gc.log
```
测试结果显示吞吐量为:2130/sec 212KB/sec

**配置Tomcat参数（五）**
测试ParallelOldGC回收器的表现
```
-XX:+PrintGCDetails -Xmx512m -Xms128m
-XX:+HeapDumpOnOutOfMemoryError
-XX:+UseParallelGC
-XX:+UseParallelOldGC
-XX:ParallelGCThreads=8
-XX:PermSize=32M
-Xloggc:d:/gc.log
```
测试结果显示吞吐量为:2236/sec 223KB/sec

**配置Tomcat参数（五）**
测试CMS回收器的表现
```
-XX:+PrintGCDetails -Xmx512m -Xms128m
-XX:+HeapDumpOnOutOfMemoryError
-XX:+UseConcMarkSweepGC
-XX:ConcGCThreads=8
-XX:PermSize=32M
-Xloggc:d:/gc.log
```
测试结果显示吞吐量为:1813/sec 181KB/sec

源码：https://github.com/zxiang179/JVM