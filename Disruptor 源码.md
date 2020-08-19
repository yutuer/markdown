### Disruptor 源码

1.  Disruptor

   1. ```java
      该类其实是个门面，用于帮助用户组织消费者。
      ```

   2. ```java
      * 2.{@link #handleEventsWith(EventHandler[])}
      *   {@link #handleEventsWith(EventProcessor...)}
      *   {@link #handleEventsWith(EventProcessorFactory[])}
      *   所有的 {@code handleEventsWith} 都是添加消费者，每一个{@link EventHandler}、
      *   {@link EventProcessor}、{@link EventProcessorFactory}都会被包装为一个独立的消费者{@link BatchEventProcessor}
      *   数组长度就代表了添加的消费者个数。
      ```

   3. ```java
      * 3.{@link #handleEventsWithWorkerPool(WorkHandler[])} 方法为添加一个多线程的消费者，这些handler共同构成一个消费者.
      *   （WorkHandler[]会被包装为 {@link WorkerPool}，一个WorkPool是一个消费者，WorkPool中的Handler们
      *   协作共同完成消费。一个事件只会被WokerPool中的某一个WorkHandler消费。
      *   数组的长度决定了线程池内的线程数量。
      ```

   4. 