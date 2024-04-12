#Netty
# ChannelOutBoundBuffer
AbstractChannel 使用的内部数据结构来存储其挂起的出站写入请求.
所以其构造器只有一个地方使用, 那就是 `AbstractChannel.AbstractUnsafe`
```java
protected abstract class AbstractUnsafe implements Unsafe {
    private volatile ChannelOutboundBuffer outboundBuffer = new ChannelOutboundBuffer(AbstractChannel.this);
    // ...
}
```

用链表的结构存储添加的`msg`,
有3个指针分别是
1. `flushedEntry` 链表的头, 表示第一个等待 flush(写入) 的消息
2. `unflushedEntry` 链表中第一个未标记需要flush的消息, 在其到头之间的消息都需要flush(写入), 之后的消息都不需要flush
3. `tailEntry` 链表的尾

```java
public final class ChannelOutboundBuffer {

    private final Channel channel;

    // Entry(flushedEntry) --> ... Entry(unflushedEntry) --> ... Entry(tailEntry)
    //
    // The Entry that is the first in the linked-list structure that was flushed
    private Entry flushedEntry;
    // The Entry which is the first unflushed in the linked-list structure
    private Entry unflushedEntry;
    // The Entry which represents the tail of the buffer
    private Entry tailEntry;
    // The number of flushed entries that are not written yet (尚未写入的元素个数)
    private int flushed;

    private int nioBufferCount;
    private long nioBufferSize;

    private boolean inFail;

    private static final AtomicLongFieldUpdater<ChannelOutboundBuffer> TOTAL_PENDING_SIZE_UPDATER =
            AtomicLongFieldUpdater.newUpdater(ChannelOutboundBuffer.class, "totalPendingSize");

    // 等待出站的字节数量
    @SuppressWarnings("UnusedDeclaration")
    private volatile long totalPendingSize;

    private static final AtomicIntegerFieldUpdater<ChannelOutboundBuffer> UNWRITABLE_UPDATER =
            AtomicIntegerFieldUpdater.newUpdater(ChannelOutboundBuffer.class, "unwritable");

    @SuppressWarnings("UnusedDeclaration")
    private volatile int unwritable;

    private volatile Runnable fireChannelWritabilityChangedTask;

}
```

### `addMessage()`
```java
public void addMessage(Object msg, int size, ChannelPromise promise) {
	Entry entry = Entry.newInstance(msg, size, total(msg), promise);
	if (tailEntry == null) {
		flushedEntry = null;
	} else {
        // 在链表尾部插入
		Entry tail = tailEntry;
		tail.next = entry;
	}
	tailEntry = entry;
    // unflushedEntry 为空说明之前所有的msg都已经写入了,本次的msg是第一个未写入的节点
	if (unflushedEntry == null) {
		unflushedEntry = entry;
	}

	// increment pending bytes after adding message to the unflushed arrays.
	// See https://github.com/netty/netty/issues/1619
	incrementPendingOutboundBytes(entry.pendingSize, false);
}
```

### `addFlush()`

移动`unflushedEntry`指针到链表尾, 相当于标记这些消息都需要真正写入channel.
[[outBoundEntry-addFlush.ex]]

```java
public void addFlush() {
    // There is no need to process all entries if there was already a flush before and no new messages
    // where added in the meantime.
    //
    // See https://github.com/netty/netty/issues/2577
    Entry entry = unflushedEntry;
    if (entry != null) {
        if (flushedEntry == null) {
            // there is no flushedEntry yet, so start with the entry
            flushedEntry = entry;
        }
        // 从未被标记未刷新的指针开始遍历, 直到链表的结尾
        // 并且设置这些消息的 promise 为不可取消
        // 如果在调用flush之前,有消息已经取消了,
        // 调用 entry.cancel() 方法清空并 release msg, 替换为一个 Unpooled.EMPTY_BUFFER (PS: why 这里并没有选择直接移除链表中的节点)
        // 并且修改减少待出站的字节数
        do {
            flushed ++;
            if (!entry.promise.setUncancellable()) {
                // Was cancelled so make sure we free up memory and notify about the freed bytes
                int pending = entry.cancel();
                decrementPendingOutboundBytes(pending, false, true);
            }
            entry = entry.next;
        } while (entry != null);

        // All flushed so reset unflushedEntry
        unflushedEntry = null;
    }
}
```

> [!NOTE] 该方法只在 AbstractChannel.flush() 中调用

## `current()`

返回当前第一个待刷新的entry. 也就是 flushedEntry 指针对应的msg.
如果返回null说明所有entry都已经刷新完了, 没有可写的东西.
```java
public Object current() {
    Entry entry = flushedEntry;
    if (entry == null) {
        return null;
    }

    return entry.msg;
}
```

## `remove()`

移除当前消息, 并且将其 ChannelPromise 标记为 success.
如果不存在 flushed message, 返回false表示没有更多消息了

```java
public boolean remove() {
    Entry e = flushedEntry;
    if (e == null) {
        clearNioBuffers();
        return false;
    }
    Object msg = e.msg;

    ChannelPromise promise = e.promise;
    int size = e.pendingSize;

    removeEntry(e);

    if (!e.cancelled) {
        // only release message, notify and decrement if it was not canceled before.
        ReferenceCountUtil.safeRelease(msg);
        safeSuccess(promise);
        decrementPendingOutboundBytes(size, false, true);
    }

    // recycle the entry
    e.recycle();

    return true;
}
```

``` java
public boolean remove(Throwable cause) {
    return remove0(cause, true);
}

private boolean remove0(Throwable cause, boolean notifyWritability) {
    Entry e = flushedEntry;
    if (e == null) {
        clearNioBuffers();
        return false;
    }
    Object msg = e.msg;

    ChannelPromise promise = e.promise;
    int size = e.pendingSize;

    removeEntry(e);

    if (!e.cancelled) {
        // only release message, fail and decrement if it was not canceled before.
        ReferenceCountUtil.safeRelease(msg);

        safeFail(promise, cause);
        decrementPendingOutboundBytes(size, false, notifyWritability);
    }

    // recycle the entry
    e.recycle();

    return true;
}

```

```java
private void removeEntry(Entry e) {
    if (-- flushed == 0) {
        // processed everything
        flushedEntry = null;
        if (e == tailEntry) {
            tailEntry = null;
            unflushedEntry = null;
        }
    } else {
        flushedEntry = e.next;
    }
}
```
