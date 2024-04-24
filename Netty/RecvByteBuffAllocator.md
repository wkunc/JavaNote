#Netty

# RecvByteBufAllocator

和 ChannelOutboundBuffer 相似, 主要负责入站消息对应的内存大小分配.
来保证从底层channel读取消息时分配的 byteBuf 大小足够的大(可以减少从channel读取的次数), 又不至于过大导致内存浪费

```java
public interface RecvByteBufAllocator {
    /**
     * Creates a new handle.  The handle provides the actual operations and keeps the internal information which is
     * required for predicting an optimal buffer capacity.
     */
    Handle newHandle();

    /**
     * @deprecated Use {@link ExtendedHandle}.
     */
    @Deprecated
    interface Handle {
        /**
         * Creates a new receive buffer whose capacity is probably large enough to read all inbound data and small
         * enough not to waste its space.
         */
        ByteBuf allocate(ByteBufAllocator alloc);

        /**
         * Similar to {@link #allocate(ByteBufAllocator)} except that it does not allocate anything but just tells the
         * capacity.
         * 返回下一次 allocate() 方法分配到bytBuf大小
         */
        int guess();

        /**
         * Reset any counters that have accumulated and recommend how many messages/bytes should be read for the next
         * read loop.
         * <p>
         * This may be used by {@link #continueReading()} to determine if the read operation should complete.
         * </p>
         * This is only ever a hint and may be ignored by the implementation.
         * @param config The channel configuration which may impact this object's behavior.
         */
        void reset(ChannelConfig config);

        /**
         * Increment the number of messages that have been read for the current read loop.
         * @param numMessages The amount to increment by.
         */
        void incMessagesRead(int numMessages);

        /**
         * Set the bytes that have been read for the last read operation.
         * This may be used to increment the number of bytes that have been read.
         * @param bytes The number of bytes from the previous read operation. This may be negative if an read error
         * occurs. If a negative value is seen it is expected to be return on the next call to
         * {@link #lastBytesRead()}. A negative value will signal a termination condition enforced externally
         * to this class and is not required to be enforced in {@link #continueReading()}.
         */
        void lastBytesRead(int bytes);

        /**
         * Get the amount of bytes for the previous read operation.
         * @return The amount of bytes for the previous read operation.
         */
        int lastBytesRead();

        /**
         * Set how many bytes the read operation will (or did) attempt to read.
         * @param bytes How many bytes the read operation will (or did) attempt to read.
         */
        void attemptedBytesRead(int bytes);

        /**
         * Get how many bytes the read operation will (or did) attempt to read.
         * @return How many bytes the read operation will (or did) attempt to read.
         */
        int attemptedBytesRead();

        /**
         * Determine if the current read loop should continue.
         * @return {@code true} if the read loop should continue reading. {@code false} if the read loop is complete.
         */
        boolean continueReading();

        /**
         * The read has completed.
         */
        void readComplete();
    }

    @SuppressWarnings("deprecation")
    @UnstableApi
    interface ExtendedHandle extends Handle {
        /**
         * Same as {@link Handle#continueReading()} except "more data" is determined by the supplier parameter.
         * @param maybeMoreDataSupplier A supplier that determines if there maybe more data to read.
         */
        boolean continueReading(UncheckedBooleanSupplier maybeMoreDataSupplier);
    }

}
```

## MaxBytesRecvByteBufAllocator

子接口, 字面意思就是限制了最大读取字节数.
每次循环读取的字节数最大值, 以及一次read事件读取的字节数最大值
默认值一次循环最大读取64kb, 且一次read事件只能读取64kb

  ``` java
  public DefaultMaxBytesRecvByteBufAllocator() {
      this(64 * 1024, 64 * 1024);
  }
  ```

## MaxMessagesRecvByteBufAllocator

限制每次read事件消息最大接受次数. 每次循环都会调用 incMessagesRead(1). 所以相当于限制了循环的次数

### DefaultMaxMessagesRecvByteBufAllocator

  ``` java
  // 是否上次doRead是否写满了整个byteBuf.
  private final UncheckedBooleanSupplier defaultMaybeMoreSupplier = new UncheckedBooleanSupplier() {
      @Override
      public boolean get() {
          return attemptedBytesRead == lastBytesRead;
      }
  };

  @Override
  public boolean continueReading() {
      return continueReading(defaultMaybeMoreSupplier);
  }

  @Override
  public boolean continueReading(UncheckedBooleanSupplier maybeMoreDataSupplier) {
      // 如果在read的while循环中用户代码取消了autoRead, 那么马上停止read
      return config.isAutoRead() &&
             // 如果尊重更多数据的话, 调用supplier判断上次是否读取是否写满了byteBuf(说明可能还有数据)
             (!respectMaybeMoreData || maybeMoreDataSupplier.get()) &&
             // 类名的来源, 是否到达了单次read事件最大读取次数.
             // 是否没有读到任何内容
             totalMessages < maxMessagePerRead && (ignoreBytesRead || totalBytesRead > 0);
  }
  ```

#### AdaptiveRecvByteBufAllocator

具体子类负责实现 guss() 策略. 这个类按照特定规则扩大/缩小下次分配的byteBuf.
连续两次读取都没有写满时缩小, 直到最小值.
如果上次写满了下次就扩大, 直到最大值.
[0~512]以16为步长, [512~1073741824] 是乘2.
默认最小 64B. 最大 65536B(64kB). 初始值 2048B.
单次read事件最多循环读取16次.

