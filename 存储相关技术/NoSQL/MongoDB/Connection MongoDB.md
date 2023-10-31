# Connection MongoDB
使用Driver驱动程序连接 MongoDB
我们使用 mongodb-driver-sync, 它包含新的API.
也可以使用 mongodb-java-driver, 它包含旧的API的同时添加了新的API.

# 连接到独立的 MongoDB 实例.

MongoDB Driver 中的连接抽象是 MongoClient.
新的API中提供了 MongoClients 工具类, 帮助我们创建对象.
(ps: 新API中提供很多工具类都是以 s 结尾的)

```java
// 不用任何参数可以创建一个 连接到 localhost:27017 的连接.
MongoClient client1 = MongoClients.create();

// 通过 connectionString 连接字符串来创建一个连接.
// 格式接近JDBC的url. mongodb://host:port,host:port 
// 因为可以连接到集群,所以host可以填多个, port 可以不写,会用默认的
MongoClient client2 = MongoClients.create("mongodb://host1");

MongoClient client2 = MongoClients.create("mongodb://host1");

// 还有一种就是使用 MongoClientSettings 对象来创建Client.
// MongoClientSettings 对象有一个 builder() 返回一个建造者来帮助我们创建实例.

MongoClient mongoClient2 = MongoClients.create(
        MongoClientSettings.builder()
                .applyToClusterSettings(builder -> builder.hosts(Arrays.asList(new ServerAddress("hostOne", 27018))))
                .build());

```

# 连接到 MongoDB 集群.
还没学习MongoDB 集群.


# 连接选项

