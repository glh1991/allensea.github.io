# Disruptor

首先我们来看看JDK提供的线程安全的内存队列有哪些?

| 队列        | 有界性    |  锁  | 数据结构 |
| --------   | -----:   | :----: | :----: |
| ArrayBlockingQueue | 有界 | 加锁 | ArrayList |
| LinkedBlockingQueue | 可选有界 | 加锁 | LinkedList |
| ConcurrentLinkedQueue | 无界 | 无锁 | LinkedList |
| LinkedTransferQueue | 无界 | 无锁 | LinkedList |
| PriorityBlockingQueue | 无界 | 加锁 | 堆 |
| DelayQueue | 无界 | 加锁 | 堆 |

底层的数据结构主要分为三种: ArrayList, LinkedList, 堆. PriorityBlockingQueue, DelayQueue主要是用来实现优先级的队列, 这里我们暂时不考虑.其次我们来看看无界和有界的差别, 无界队列在生产者生产速率过快的时候, 可能导致OOM的问题, 这在生产环境中显然不适用. 那么就剩下ArrayBlockingQueue, 这个还不错的选择, 但是ArrayBlockingQueue是通过加锁实现线程安全. 通过加锁实现的线程安全相对CAS方式实现的线程安全在性能上有8倍的差距.这是ArrayBlockingQueue的一个弱势.

### ArrayBlockingQueue

ArrayBlockingQueue会在实际的使用过程中,由于加锁和伪共享的问题导致性能问题.

#### 加锁

通过加锁方式实现的线程安全性能问题是显而易见的, 线程竞争锁失败导致挂起, 等待锁被释放, 线程从Blocked到Runnable这个过程也存在不小的开销. 此外还可能存在优先级反转的问题, 比如说, 一个阻塞的线程优先级更高, 而获得锁的线程优先级没那么高, 就会导致优先级反转的情况. ArrayBlockingQueue的put源码:

```
public void put(E e) throws InterruptedException {
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == items.length)
            notFull.await();
        enqueue(e);
    } finally {
        lock.unlock();
    }
}
```

#### 伪共享

伪共享的本质是多个没有关联的变量被CPU加载到同一个缓存行中, 在多线程的环境下被不同的CPU执行, 这样就会导致缓存行频繁失效导致Cache命中率下降. 以ArrayBlockingQueue为例, ArrayBlockingQueue中有3个成员变量: 

```
/** 需要被取走的元素下标 */
int takeIndex;

/** 可被元素插入的位置的下标 */
int putIndex;

/** 队列中元素的数量 */
int count;
```

这三个变量很容易被放到一个缓存行中,但是由于之间没有太多的关联, 其中一个变量发生修改, 就会导致缓存行失效, 需要到主存中获取数据, 降低了性能, 无法充分使用缓存行的特性, 这种现象就被称为伪共享.

### Disruptor的设计
* Event: 消息载体

* 环形数组(RingBuffer)
  
  用于存储Event,Disruptor规定RingBuffer大小必须为2的整数次幂, 否则会报错. 使用数组的数据结构能够减少GC, 同时数组相对于CPU的缓存机制也更友好.通过位运算算出index的大小, 与HashMap计算index的方式一致, 这种方式相对与取模的效率要高. 

* 元素位置定位(Sequence)
  
  这个对于Disruptor来说是核心所在, Sequence通过顺序递增的序号来编号管理事件，对事件的处理过程总是沿着序号逐个递增处理. 一个Sequence用于跟踪标识某个特定的事件处理者(Producer/Consumer)的处理进度.注意: Producer和Consumer是分别用两个 Sequence来管理的. Sequence是一个long类型的数值占8bit, 一个CPU的缓存行大小64bit, 为了避免伪共享的问题(Producer和Consumer的Sequence落在同一条缓存行上) Sequence会使用一个长度为8的long[],最后一位保存value, 其他位置用0填充缓存行的方式,来实现避免伪共享. 同时通过CAS操作来维护value, 做序号的递增.
  
* Sequencer

  Sequencer有两个实现类, SingleProducerSequencer, MultiProducerSequencer, 它们定义在生产者和消费者之间快速、正确地传递数据的并发算法. 
  我们先来看SingleProducerSequencer的实现.
  
  ```
  @Override  
  private boolean hasAvailableCapacity(int requiredCapacity, boolean doStore)
    {
      // 下一个序列值
        long nextValue = this.nextValue;

    // 申请序列点, 生产者序列值不能比消费者序列值大一圈
        long wrapPoint = (nextValue + requiredCapacity) - bufferSize;
        // 消费者上次申请的消费序列
        long cachedGatingSequence = this.cachedValue;

     
        if (wrapPoint > cachedGatingSequence || cachedGatingSequence > nextValue)
        {
            if (doStore)
            {
                cursor.setVolatile(nextValue);  // StoreLoad fence
            }
      // gatingSequences 是多个consumer的消费序列.
            long minSequence = Util.getMinimumSequence(gatingSequences, nextValue);
            this.cachedValue = minSequence;

            if (wrapPoint > minSequence)
            {
                return false;
            }
        }

        return true;
    }
  ```
  我们来做一个输入一个case. 假设buffsize=32, nextValue = 60, requiredValue=3, cachedValue=30, 那么wrapPoint=63-32=31. 也就是说, 这个时候provider去申请buffer的时候发现比consumer上次申请的序列要超一圈了, 这个时候就需要判定是否能够支持provider支持申请ringbuffer.这个时候重新去获取consumer的消费结果, 获得最小的序列值并再次做判定, 如果最小的序列值还是小于wrapPoint, 则说明生产者没有办法申请ringbuffer.
  
  从上面的源码我们可以看出SingleProducerSequencer除了要维护nextValue以外还需要追踪cachedValue, 也就是消费者的最小消费序列值.
  
  我们再来看一看发布事件的代码, 发布一个序列以后, 将会唤起事件consumer
  
  ```
  @Override
    public void publish(long sequence)
    {
        cursor.set(sequence);
        waitStrategy.signalAllWhenBlocking();
    }
  ```
  
  接着, 我们再来看看MultiProducerSequencer的实现, 相对于SingleProducerSequencer不同的是, MultiProducerSequencer添加了一个availableBuffer.是一个int型的数组.size大小和RingBuffer的Size一样大, 用来追踪Ringbuffer的每个槽的状态.初始化为-1, 每次赋的值是当前的圈数.
* 无锁设计

  采用CAS的方式, 相对于使用加锁方式性能更高效.


### 参考资料
* https://tech.meituan.com/disruptor.html
* http://blog.decaywood.me/2016/01/22/disruptor-guide/ 
* http://brokendreams.iteye.com/blog/2255703