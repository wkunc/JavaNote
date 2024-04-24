MongoDB 把 Document 存储到 collection 中, collection 包含在 database.

# Access a Database

我们需要一个连接到部署的MongoDB的 MongoClient 实例.
使用这个实例的 getDatabase() 方法来访问一个 database.

指明要访问的 database 的 name, 如果没有这个name的 database.
MongoDB 会创建这个数据库在你第一次存储数据到这个数据库中.

```java
MongoDatabase database = mongoClient.getDatabase("test");
```

> 注意:
> MongoDatabase 实例是一个 不可变的 对象.和String一样.

# Access a Collection

我们有一个 MongoDatabase 实例, 使用它的 getCollection()
方法来访问一个 Collection.

需要指明访问的Collection的name.

```java
MongoCollection<Document> collection = database.getCollection("test");
```

> 注意:
> MongoCollection 实例是一个 不可变的 对象.和String一样.

如果 Collecion 不存在, MongoDB 会在你第一次存储该集合的数据时创建.

我们还可以使用各种选项的方法显式创建集合, 例如设置最大大小或文档验证规则.

## 明确的创建 Collection

使用 MongoDatabase 提供的 createCollection() 方法我们可以显式的创建集合.
显式创建集合的时候, 可以使用 CreateCollectionOptions 类指定各种*集合选项*.
例如: 最大大小或文档验证规则.
如果没有指定这些规则, 则无需显式创建集合.
因为MongoDB会在第一次存储集合数据时创建集合.

```java
// 创建一个名字时 cappedCollection 的集合, 并且单个文档上限是 1MB
database.createCollection("cappedCollection", new CreateCollectionOptions().capped(true).sizeInBytes(0x100000));

// 创建一个民族是 contacts 的集合, 并且其中的文档必须有email或phone字段
ValidationOptions collOptions = new ValidationOptions().validator(Filters.or(Filters.exists("email"), Filters.exists("phone")));
database.createCollection("contacts", new CreateCollectionOptions().validationOptions(collOptions));
```

# 获取Collection的List(获得多个Collection)

```java
for (String name : database.listCollectionNames()) {
    System.out.println(name);
}
```

# 删除一个Collection

调用对应Colleciton的 MongoCollection实例上的 drop() 方就可以删除对应Collection

```java
MongoCollection<Document> collection = database.getCollection("contacts");
collection.drop();
```


