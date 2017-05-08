---
title: Disruptor基础
date: 2017-05-08
tags: 并发编程
categories: JAVA并发编程
---

# Disruptor

>Martin Fowler在自己网站上写了一篇LMAX架构的文章，在文章中他介绍了LMAX是一种新型零售金融交易平台，它能够以很低的延迟产生大量交易。这个系统是建立在JVM平台上，其核心是一个业务逻辑处理器，**它能够在一个线程里每秒处理6百万订单。**业务逻辑处理器完全是运行在内存中，使用事件源驱动方式。业务逻辑处理器的核心是Disruptor。

### 在Disruptor中，我们想实现hello world 需要如下几步骤：
- 第一：建立一个Event类
- 第二：建立一个工厂Event类，用于创建Event类实例对象
- 第三：需要有一个监听事件类，用于处理数据（Event类）
- 第四：我们需要进行测试代码编写。实例化Disruptor实例，配置一系列参数。然后我们对Disruptor实例绑定监听事件类，接受并处理数据。
- 第五：在Disruptor中，真正存储数据的核心叫做RingBuffer，我们通过Disruptor实例拿到它，然后把数据生产出来，把数据加入到RingBuffer的实例对象中即可。

**Event类**
```
public class LongEvent {

	private long value;
	public long getValue() {
		return value;
	}
	public void setValue(long value) {
		this.value = value;
	}
}
```
**EventFactory类**
```
//需要让disruptor为我们创建事件，我们同时还声明了一个EventFactory来实例化Event对象
public class LongEventFactory implements EventFactory{

	@Override
	public Object newInstance() {
		return new LongEvent();
	}
}
```
**EventHandler类**
```
public class LongEventHandler implements EventHandler<LongEvent> {

	@Override
	public void onEvent(LongEvent longEvent, long l, boolean b)
			throws Exception {
		System.out.println(longEvent.getValue());
	}

}
```
**LongEventProducer类**
```
/**
 * 很明显 当用一个简单队列来发布事件的时候会牵涉更多的细节，这是因为事件对象还需要预先创建。
 * 发布事件最少需要两步：获取下一个事件槽并发布事件（发布事件的时候要使用try/finally保证事件一定被发布）
 * 如果我们使用RingBuffer.next()获取一个事件槽，那么一定要发布对应的事件。
 * 如果不能发布事件，那么会引起Disruptor状态的混乱。
 * 尤其是在多个事件生产者的情况下会导致事件消费者失速，从而不得不重启应用才能恢复。
 * @author Carl_Hugo
 *
 */
public class LongEventProducer {
	
	private final RingBuffer<LongEvent> ringBuffer;
	
	public LongEventProducer(RingBuffer<LongEvent> ringBuffer){
		this.ringBuffer=ringBuffer;
}
	
	/**
	 * onData用来发布事件，每调用一次就发布一次事件
	 * 它的参数会通过事件传递给消费者
	 * @param bb
	 */
	public void onData(ByteBuffer bb){
		//1 可以把ringBuffer看作一个事件队列那么next就是得到下一个事件槽
		long sequence = ringBuffer.next();
		try {
			//2 用上面的一个索引取出一个空的事件用于填充(获取该序号对应的事件对象)
			LongEvent longEvent = ringBuffer.get(sequence);
			//3 获取要通过事件传递的业务数据
			longEvent.setValue(bb.getLong(0));
		} finally{
			//4 发布事件
			//注意，最后的ringBuffer.publish方法必须包含在finally中以确保必须得到调用，如果某个请求的sequence未被提交
			ringBuffer.publish(sequence);	
		}
	}

}
```
**Main类**
```
public class LongEventMain {
	
	public static void main(String[] args) {
		//创建缓冲区
		ExecutorService executor = Executors.newCachedThreadPool();
		//创建工厂
		LongEventFactory factory = new LongEventFactory();
		//创建bufferSize，也就是RingBuffer的大小，必须是2的N次方
		int ringBufferSize = 1024*1024;
		
		/*//BlockingWaitStrategy是最低效的策略，但其对CPU的消耗最小并且在各种不同的环境中能够提供更加一致的性能表现
		WaitStrategy BLOCKING_WAIT = new BlockingWaitStrategy();
		//SleepingWaitStrategy的性能表现和BlockingWaitStrategy差不多，对CPU消耗也类似，但其对生产者线程的影响最小，适合用于异步日志类的场景。
		WaitStrategy SLEEPING_WAIT = new SleepingWaitStrategy();
		//YieldingWaitStrategy的性能最好，适用于低延迟的系统。在要求性能极高且时间处理线程数小于CPU逻辑线程数的场景中，推荐使用此策略：例如CPU开启超线程策略
		WaitStrategy YIELD_WAIT = new YieldingWaitStrategy();*/
		
		//创建Disruptor
		//1 第一个参数为工厂类对象，用于创建一个个的LongEvent，LongEvent是实际的消费数据
		//2 第二个参数是缓冲区大小
		//3 第三个参数是线程池，进行Disruptor内部的数据接受处理调度
		//4 第四个参数ProducerType.SINGLE和ProducerType.MULTI
		//5 第五个参数是一种策略:WaitStrategy
		Disruptor<LongEvent> disruptor = 
				new Disruptor<LongEvent>(factory, ringBufferSize, executor,ProducerType.SINGLE,new YieldingWaitStrategy());
		//连接消费事件方法
		disruptor.handleEventsWith(new LongEventHandler());
		//启动
		disruptor.start();
		
		//Disruptor的事件发布有两个阶段提交的过程
		//使用该方法获得具体存放数据的容器ringBuffer(环形结构)
		RingBuffer<LongEvent> ringBuffer = disruptor.getRingBuffer();
		
		LongEventProducer producer = new LongEventProducer(ringBuffer);
//		LongEventProducerWithTranslator producer = new LongEventProducerWithTranslator(ringBuffer);
		ByteBuffer byteBuffer = ByteBuffer.allocate(8);
		for(long a=0;a<100;a++){
			byteBuffer.putLong(0,a);
			producer.onData(byteBuffer);
		}
		
		disruptor.shutdown();//关闭Disruptor，方法会堵塞，知道所有的事件都得到处理
		executor.shutdown();//关闭Disruptor使用的线程池：如果需要的话，必须手动关闭，Disruptor在shutdown时不会自动关闭。
	}

}
```

**LongEventProducerWithTranslator类**
```
public class LongEventProducerWithTranslator {
	
	//一个translator可以看作一个事件的初始化器，publicEvent方法会调用它
	//填充Event
	public static final EventTranslatorOneArg<LongEvent, ByteBuffer> TRANSLATOR =
			new EventTranslatorOneArg<LongEvent, ByteBuffer>() {
				@Override
				public void translateTo(LongEvent event, long sequence,ByteBuffer buffer) {
					event.setValue(buffer.getLong(0));
				}
			};
			
	private final RingBuffer<LongEvent> ringBuffer;
	
	public LongEventProducerWithTranslator(RingBuffer<LongEvent> ringBuffer){
		this.ringBuffer=ringBuffer;
	}
	
	public void onData(ByteBuffer buffer){
		ringBuffer.publishEvent(TRANSLATOR,buffer);
	}
}
```


**RingBuffer:** 被看作Disruptor最主要的组件，然而从3.0开始RingBuffer仅仅负责存储和更新在Disruptor中流通的数据。对一些特殊的使用场景能够被用户(使用其他数据结构)完全替代。

>**ringbuffer到底是什么？**
答：嗯，正如名字所说的一样，它是一个环（首尾相接的环），你可以把它用做在不同上下文（线程）间传递数据的buffer。

基本来说，ringbuffer拥有一个序号，这个序号指向数组中下一个可用元素
![](http://i.imgur.com/nqzyhjh.png)

随着你不停地填充这个buffer（可能也会有相应的读取），这个序号会一直增长，直到绕过这个环。

要找到数组中当前序号指向的元素，可以通过mod操作：sequence mod array length = array index（取模操作）以上面的ringbuffer为例（java的mod语法）：12 % 10 = 2。很简单吧。
事实上，上图中的ringbuffer只有10个槽完全是个意外。如果槽的个数是2的N次方更有利于基于二进制的计算机进行计算。

>**ringbuffer的优点？**
- 因为它是数组，所以要比链表快，而且有一个容易预测的访问模式。
- 这是对CPU缓存友好的，也就是说在硬件级别，数组中的元素是会被预加载的，因此在ringbuffer当中，cpu无需时不时去主存加载数组中的下一个元素。
- 其次，你可以为数组预先分配内存，使得数组对象一直存在（除非程序终止）。这就意味着不需要花大量的时间用于垃圾回收。此外，不像链表那样，需要为每一个添加到其上面的对象创造节点对象—对应的，当删除节点时，需要执行相应的内存清理操作。


**Sequence:** Disruptor使用Sequence来表示一个特殊组件处理的序号。和Disruptor一样，每个消费者(EventProcessor)都维持着一个Sequence。大部分的并发代码依赖这些Sequence值的运转，因此Sequence支持多种当前为AtomicLong类的特性。
**Sequencer:** 这是Disruptor真正的核心。实现了这个接口的两种生产者（单生产者和多生产者）均实现了所有的并发算法，为了在生产者和消费者之间进行准确快速的数据传递。
**SequenceBarrier: **由Sequencer生成，并且包含了已经发布的Sequence的引用，这些的Sequence源于Sequencer和一些独立的消费者的Sequence。它包含了决定是否有供消费者来消费的Event的逻辑。

**WaitStrategy：**决定一个消费者将如何等待生产者将Event置入Disruptor。
**EventProcessor：**主要事件循环，处理Disruptor中的Event，并且拥有消费者的Sequence。它有一个实现类是BatchEventProcessor，包含了event loop有效的实现，并且将回调到一个EventHandler接口的实现对象。
**EventHandler：**由用户实现并且代表了Disruptor中的一个消费者的接口。
**Producer：**由用户实现，它调用RingBuffer来插入事件(Event)，在Disruptor中没有相应的实现代码，由用户实现。
**WorkProcessor：**确保每个sequence只被一个processor消费，在同一个WorkPool中的处理多个WorkProcessor不会消费同样的sequence。
**WorkerPool：**一个WorkProcessor池，其中WorkProcessor将消费Sequence，所以任务可以在实现WorkHandler接口的worker吃间移交。
**LifecycleAware：**当BatchEventProcessor启动和停止时，于实现这个接口用于接收通知。

### 初看Disruptor，给人的印象就是RingBuffer是其核心，生产者向RingBuffer中写入元素，消费者从RingBuffer中消费元素，如下图：

![](http://i.imgur.com/nx2DiPG.png)

[源码 GitHub][1]
[1]: https://github.com/zxiang179/Disruptor

