# Stream 流的定义

java Stream库提供了比集合 (collection,map)更高一层的抽象

在集合中我们思考怎么做,比如如何排序,选择合适的容器;

而在 Stream 中我们思考做什么,不用考虑流程

# 创建流

流是数据结构的高层抽象, 所以 Collection 接口中有一个 strem() 方法来生成一个流

同时我们可以用 Stream 中的 static 方法来创建 空的 stream 和 无限流
