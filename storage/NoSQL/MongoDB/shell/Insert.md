# Insert Operations

MongoDB 提供了以下方法创建并插入文档到Collection中.

* db.collection.insertOne({});
* db.collection.insertMany({},{},...);

在MongoDB中, insert operations 目标是单个collection.
所有的 write operations(写操作) 在mongodb中都是单个文档级别的原子性的操作.

# Insert Behavior

上面两个方法都及其简单, 就不演示用法.

1. 当要插入的collection不存在, 插入操作会创建它.
2. 在MongoDB中每个文档都有一个唯一的 \_id 字段作为主键, 如果在插入时没有指定, MongDB driver会自己生成.
3. 所有的写操作都是一个原子性操作, 详情见事务.
4. Write Concern. 检测等级, 不懂? 好像会在集群, 分片, 副本集的时候出现.
