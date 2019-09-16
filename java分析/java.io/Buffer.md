# Buffer
如何获得*缓冲器*. ByteBuffer 提供了两种方式, 四个静态方法.
1. 通过分配获得一个空的Buffer
2. 通过包装已有的byte[]获得Buffer
```java
public static ByteBuffer allocate(int capacity) {
    if (capacity < ) {
        throw new IlleagealArgumentException();
    }
    return new HeapByteBuffer(capacity, capacity);
}
public static ByteBuffer wrap(byte[] array, int offset, length) {
    try {
        return new HeapByteBuffer(array, offset, length);
    } catch (IllegalArgumentException x) {
        throw new IndexOutOfBoundsException();
    }
}
```

# HeapByteBuffer
多数情况下会new HeapByteBuffer().
HeapByteBuffer 是ByteBuffer的子类.

```java
HeapByteBuffer(int cap, int lim) {
    super(-1, 0, lim, cap, new byte[cap], 0);
}
HeapByteBuffer(byte[] buf, int off, int len) {
    super(-1, off, off+len, buf.length, buf, 0);
}
```

```java
ByteBuffer(int mark, int pos, int lim, int cap, byte[] hb, int offset) {
    super(mark, pos, lim, cap);
    this.hb = hb;
    this.offset = offset;
}
```

```java
Buffer(int mark, int pos, int lim, int cap) {
    if (cap < 0)
        throw new 
    this.capacity = cap;
    limit(lim);
    position(pos);
    if (mark >= 0) {
        if (mark > pos)
            throw new 
        this.mark = mark;
    }
}
```

Buffer 中重要的四个值.
pos(位置), limit(界限), capacity(容量), mark(标记)

pos: 当前位置, 也就是下次读写发生的位置, get()/put() 方法会读出一个byte,并将pos向后+1

capacity: 缓冲器的整体容量, 就是底层byte[]的长度. 被包装的byte[]不应该改变所以cap通常是不会变的

limit: 读写的界限, 因为缓冲器不一定是满的, 可能只有一半是有数据的.
在没有数据的位置进行读写操作是毫无意义的.
所以需要一个标记来表示读写的界限, get()/put() 只能到limit.

mark: 标记, 是用来调整pos位置的记号. 因为get()/put()都会使pos向后移动,
如果我们想在当前pos之前进行读写除了绝对的get()方法和put()方法外, 我们可以设一个标记,
在调用某些方法后向前移动我们的pos(). 让我们可以反复读写.

## ByteBuffer的使用
get() 系列方法.
除了get(int index) 这个决定的get方法.
其他get方法均会移动 pos.
```java
// 这两个方法没有实现交给 ByteBuffer 子类实现
public abstract byte get();
public abstract byte get(int index);

// 获取多个值填充到给定的 byte 数组中.
public ByteBuffer get(byte[]) {
    return get(dst, 0, dst.length);
}
// offset : 开始填充的位置, 复制的长度
public ByteBuffer get(byte[] dst, int offset, int length) {
    checekBounds(offset, length, dst.length);
    if (length > remaining())
        throw new 
    // 确定复制完成后数组的下标. 执行length个循环.调用get()方法
    int end = offset + length;
    for (int i = offset; i < end; i++)
        dst[i] = get();
    return this;
}
```

```java
public abstract ByteBuffer put(byte b);
public abstract ByteBuffer put(int index, byte b);

public final ByteBuffer put(byte[] src) {
    return put(src, 0, src.length);
}
public final ByteBuffer put(byte[] src, int offset, int length) {
    checkBounds(offset, length, src.length);
    if (length > remaining())
        throw new 
    for (int i = offset; i < end; i++)
        this.put(src[i]);
    return this;
}
public ByteBuffer put(ByteBuffer src) {
    if (src == this) {
        throw new 
    }
    if (isReadOnly())
        throw new 
    if (n > remaining())
        throw new
    for (int i = 0; i < n; i++)
        put(src.get());
    return this;
}
```

----
Buffer 中定义的方法

```java
filp()
clear()
rewind()

mark()
reset()

postition()
postition(int)
limit()
limit(int)
```

```java
// 在put()方法或channel.read()方法后调用, 让这个Buffer准备好被写
public final Buffer filp() {
    limit = position;
    position = 0;
    mark = -1;
    return this;
}
// 在write() 之后调用, 让这个Buffer准备好被读
public final Buffer rewind() {
    position = 0;
    mark = -1;
    return this;
}
// 这个方法将Buffer中的标志变成初始化时的样子.
// 在重新使用Buffer前调用这个方法.
// 这样新写入的byte就会覆盖之前的数据
public fianl Buffer clear() {
    position = 0;
    limit = capacity;
    mark = -1;
    return this;
}

mark()
reset()

postition()
postition(int)
limit()
limit(int)
```
