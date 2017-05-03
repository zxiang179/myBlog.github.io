---
title: JVM参数配置
date: 2017-05-03
tags: JVM
categories: JAVA
---
### 虚拟机参数
在虚拟机运行的过程中，如果可以跟踪系统的运行状态，那么对于问题的故障排除会有一定的帮助，为此，虚拟机提供了一些跟踪系统状态的参数，使用给定的参数执行java虚拟机，就可以在系统运行是打印相关日志，用于分析实际问题。我们进行虚拟机参数配置，其实主要就是围绕着**堆栈方法区配置。**

#### 堆分配参数
**-XX:+PrintGC** :使用这个参数，虚拟机启动后只要遇到GC就会打印日志。
**-XX:+UseSerialGC** :配置串行回收器。
**-XX:+PrintGCDetails**: 可以查看详细信息，包括各个区的情况。
**-Xms**:设置java程序启动时的初始堆大小。
**-Xmx**:设置java程序能获得最大堆的大小
**-Xmx20m -Xms5m -XX:+PrintCommandLineFlags**:可以将隐式或显示传给虚拟机的参数输出。
**总结：**在实际工作中，我们可以直接将**初始堆的大小和最大堆大小设置相等**，这样的好处是可以减少运行时的垃圾回收次数，从而提高性能。

```
public class Test01 {
	
	public static void main(String[] args) {
		
		//-XX:+PrintGC -Xms5m -Xmx20m -XX:+UseSerialGC -XX:+PrintGCDetails -XX:+PrintCommandLineFlags
		
		//1:
		//-XX:+PrintGC -Xms5m -Xmx20m -XX:+UseSerialGC -XX:+PrintGCDetails
		
		//查看GC信息
		System.out.println("max memory:"+Runtime.getRuntime().maxMemory());
		System.out.println("free memory:"+Runtime.getRuntime().freeMemory());
		System.out.println("total memory:"+Runtime.getRuntime().totalMemory());
		
		byte[] b1 = new byte[1*1024*1024];
		System.out.println("分配了1M");
		System.out.println("max memory:"+Runtime.getRuntime().maxMemory());
		System.out.println("free memory:"+Runtime.getRuntime().freeMemory());
		System.out.println("total memory:"+Runtime.getRuntime().totalMemory());
		
		byte[] b2 = new byte[4*1024*1024];
		System.out.println("分配了4M");
		System.out.println("max memory:"+Runtime.getRuntime().maxMemory());
		System.out.println("free memory:"+Runtime.getRuntime().freeMemory());
		System.out.println("total memory:"+Runtime.getRuntime().totalMemory());
		
		int a = 0x00000000fee10000;
		int b = 0x00000000fec00000;
		System.out.println("结果为1920k，但实际结果："+(a-b)/1024);
		
	}

}

```

#### 堆分配参数（二）
**-Xmn**:可以设置新生代的大小，设置一个比较大的新生代会减少老年代的大小，这个参数对系统性能以及GC行为有很大的影响，新生代大小一般会设置整个堆空间的1/3到1/4左右。
**-XX:SurvivorRatio**:用来设置新生代中eden空间和from/to空间的比例。含义：-XX:SurvivorRatio=eden/from=eden/to
总结：不同的堆分布情况，对系统的执行会产生一定的影响，在实际工作中，应该根据系统的特点做出合理的配置，基本策略：尽可能的将对象预留在新生代，减少老年代的GC次数。
除了可以设置新生代的绝对大小(-Xmn),还可以使用(-XX:NewRation)设置新生代和老年代的比例：-XX:newRatio=老年代/新生代

```
public class Test02 {

	public static void main(String[] args) {
		//第一次配置                                         eden 2 =from 1+ to 1
		//-Xms20m -Xmx20m -Xmn1m -XX:SurvivorRatio=2 -XX:+PrintGCDetails -XX:+UseSerialGC
		
		//第二次配置
		//-Xms20m -Xmx20m -Xmn7m -XX:SurvivorRatio=2 -XX:+PrintGCDetails -XX:+UseSerialGC
		
		//第三次配置
		//-XX:NewRatio=老年代/新生代
		//-Xms20m -Xmx20m -XX:NewRatio=2 -XX:+PrintGCDetails -XX:+UseSerialGC
	    
		byte[] b=null;
		//连续向系统申请10m空间
		for(int i=0;i<10;i++){
			b=new byte[1*1024*1024];
		}
	}
	
}

```

#### 堆溢出处理
在java程序运行的过程中，如果堆空间不足，则会抛出内存溢出的错误(Out Of Memory),一旦这类问题发生在生产环境，可能引起严重的业务中断，java虚拟机提供了-XX:+HeapDumpOnOutOfMemoryError,使用该参数可以在内存溢出时导出整个堆信息，与之配合的还有参数，-XX:HeapDumpPath,可以设置导出堆的路径。
内存分析工具：Memory Analyzer 1.5.0

```
public class Test3 {
	
	public static void main(String[] args) {
		//-Xms2m -Xmx2m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=e:/Test03.dump
		//堆内存溢出
		Vector vector = new Vector();
		for(int i=0;i<5;i++){
			vector.add(new Byte[1*1024*1024]);
		}
	}

}

```

### 栈配置
Java虚拟机提供了参数-Xss来指定线程的最大栈空间，整个参数也直接决定了函数可调用的最大深度。

```
public class Test4 {
	
	//-Xss1m
	//-Xss5m
	
	//栈调用深度
	private static int count;
	
	public static void recursion(){
		count++;
		recursion();
	}
	
	public static void main(String[] args) {
		try {
			recursion();
		} catch (Throwable e) {
            System.out.println("调用最大深处："+count);
			e.printStackTrace();
		}
	}
}
```
### 方法区
和java堆一样，方法区是一块所有线程共享的内存区域，它用于保存系统的类信息，方法区(永久区)可以保存多少信息可以对其进行配置，在默认情况下，-XX:MaxPermSize为64MB，如果系统运行时产生大量的类，就需要设置一个相对合适的方法区，以免出现永久区内存溢出的问题。
-XX:PermSize=64M -XX:MaxPermSize=64M

### 直接内存配置
直接内存也是java程序中非常重要的组成部分，特别是广泛用在NIO中，直接内存跳过了java堆，使java程序可以直接访问原生堆空间，因此在一定的程度上加快了内存空间的访问速度。但是说直接内存一定就可以提高内存访问速度也不见得，具体问题具体分析。
相关配置参数：-XX:MaxDirectMemorySize,如果不设置默认值为最大空间，即-Xmx。直接内存使用达到上限时，就会触发垃圾回收，如果不能有效的释放空间，也会引起系统的OOM。

### Client和Server虚拟机的工作模式
java虚拟机支持Client和Server模式，使用-client可以指定使用Client模式，使用-server可以使用Server模式。（jdk1.7及以后不需要考虑）
区别：client模式比server模式启动要快，但长期运行server模式性能远远好于Client模式。

JVM博客：http://www.cnblogs.com/redcreen/tag/jvm/

源码：https://github.com/zxiang179/JVM