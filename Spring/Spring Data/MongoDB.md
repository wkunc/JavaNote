# MongoDB

[MongoDB support](https://docs.spring.io/spring-data/mongodb/docs/2.1.9.RELEASE/reference/html/#mongo.core)

[MongoDB Repositories](https://docs.spring.io/spring-data/mongodb/docs/2.1.9.RELEASE/reference/html/#mongo.repositories)

# MongoDB support

* Spring配置支持基于Java的@Configuration类或Mongo驱动程序实例和副本集的XML命名空间.

* MongoTemplate助手类, 可在执行常见的Mongo操作时提高工作效率.包括文档和POJO之间的集成对象映射。

* 异常转换为Spring的可移植数据访问异常层次结构.

* 与Spring的转换服务集成的功能丰富的对象映射.

* 基于注释的映射元数据, 可扩展以支持其他元数据格式.

* 持久性和映射生命周期事件.

* 基于Java的查询, 标准和更新DSL.

* 自动实现Repository接口, 包括支持自定义finder方法.

* QueryDSL集成以支持类型安全查询.

* 对JPA实体的跨存储持久性支持,使用MongoDB透明地保留和检索字段(不建议使用 - 无需替换即可删除)

* GeoSpatial 集成.

## Connecting to MongoDB with Spring

使用MongoDB with Spring时的首要任务之一是
*使用IOC容器创建com.mongodb.MongoClient或com.mongodb.client.MongoClient对象*.

有两种主要方法可以实现此目的.

1. 使用原生的创建方式
2. 使用Spring提供的FactoryBean

```java
@Configuratoin
public class AppConfig {
    // 老式API
    @Bean
    public com.mongodb.MongoClient mongoClient1() {
        return new MongoClient("localhost");
    }
    // Sync API
    @Bean
    public com.mongodb.client.MongoClient mongoClient2() {
        return MongoClients.create("localhost");
    }

    // 旧API的FactoryBean
    @Bean
    public MongoClientFactoryBean mongo() {
        MongoClientFactoryBean mongo = new MongoClientFactoryBean();
        mongo.setHost("localhost");
        return mongo;
    }
}
```

使用FactoryBean的方式, Spring会提供MongDb异常到 SpringDataAccessException的转换.

## The MongoDbFactory Interface

虽然com.mongodb.MongoClient是MongoDB driver API的入口点,
但是连接到特定MongoDB数据库实例需要其他信息, 例如数据库名称和可选的用户名和密码.
使用该信息, 您可以获取com.mongodb.client.MongoDatabase 对象并访问数据库实例的所有功能.

说了这么多, 总结起来就是 MongoClient 是连接到整个MongoDB Server的实例.
我们通常需要从MongoClient上获取具体的 MongoDatabase 对象.
而Spring提供了 MongoDBFactroy 接口, 用于引导到数据库的连接:

```java
public interface MongoDbFactory {

  MongoDatabase getDb() throws DataAccessException;

  MongoDatabase getDb(String dbName) throws DataAccessException;
}
```

主要有两个实现类对应新老API: SimpleMongoDbFactory(老), SimpleMongoClientDbFactory(新)

## MongoTemplate

MongoTemplate 在org.springframework.data.mongodb.core 包中,
是Spring对 MongDB 支持的 central 类. 提供了与数据库交互的丰富的功能集.

这个 template 提供了便捷的操作来 create, update, delete, select
MongoDB Document (文档)对象, 并提供 domain 对象和 Document 之间的映射.

> 注意:
> MongoTemplate 是线程安全的类, 可以在多个类中共享实例.

Document 和 domain 对象之间的映射功能是委托给 MongoConverter 接口的实现类.
Spring 提供了 MappingMongoConverter 做为默认实现, 但是你可以配置自己的
Converter 来覆盖. 更多信息参阅[Overriding Default Mapping with Custom Converters](#)

MongoTemplate 类实现了 MongoOperations 接口.
MongoOperations 接口中定义的方法命名尽可能的接近 MongoDriver
中的 Collection 的方法名. 方便熟悉DriverAPI的人学习使用.

和Collection上方法的区别就是 MongoOperations 中的方法允许传入 domain 对象.
而不是 Document, 此外, MongoOperations 具有更棒的查询API.

## Methods for Saving and Inserting Documents

MongoTemplate 有几种方便的方法来保存和插入对象.
要对象转换进行更细粒度的控制, 可以使用MappingMongoConverter注册Spring Converter.
如: Converter<Person, Document>, Converter<Document, Person>

使用 save 操作最简单的方式是 save 一个 POJO.
在这种情况下MongoDB Collection的名称由类的Simple Name确定.
也可以使用特定集合名称调用 save 操作. 可以使用映射元数据覆盖.

insert 或 save 保存时, 如果没有设置Id属性, 则假设其值将数据库自动生成.
因此, 类中的Id字段或属性的类型必须是 String, ObjectId, BigInteger.
(因为MongoDB生成的Id一定是 ObjectId类型, 而且默认只有String 或 BigInteger可以和其转换)

```java
Person p = new Person("Bob", 33);
// insert(object, collectionName);
mongoOperations.insert(p);
mongoOperations.insert(p, "person");
mongoOperations.save(p);
mongoOperations.save(p, "person");
```

除了了在调用insert, save方法传入collectionName可以显式指定,
通过@Document注解也可以指定目标Collection.

```java
// 设置这个实体类映射的collection.
@Document("pers")
public class Person() {
    ...
}
```

当然最高优先级是调用MongoTemplate.insert()方法时传入的值.

save 和 insert 的API是一致的.
区别在于如果要保存的 object 在数据库中已经存在同一Id的文档.
insert 会报错, 而 save 会覆盖那个文档.

## Updateing Documents in a Collection

MongoDB 驱动提供了update第一个匹配项的方法和更新所有匹配项的方法,
所以 MongoOperation 也提供了一样的逻辑和方法:
updateFirst(Query, Update, ...), updateMulti(Query, Update, ...)

```java
// updateMult(Query, Update, Class), 重点是 Query对象和 Update对象的构建.
// 后面会详细说.
mongoOperations.updateMult(
new Query(where("accounts.accountType").is(Account.Type.SAVINGS)),
new Update().inc("accounts.$.balance", 50.00),
Account.class);
```

### Methods in the Update Class

我们可以在 Update 类中使用一些"语法糖", 因为它的方法是链式的即返回的值是this.

* Update addToSet(String key)
* Update currentDate(String key)
* Update currentTimestamp(String key)
* Update inc(String key, Number inc)
* Update max(String key, Object max)
* Update min(String key, Object min)
* Update multiply(String key, Number nultiplier)
* Update pop(String key, Update.Position pos)
* Update pull(String key, Object value)
* Update pullAll(String key, Object[] valules)
* Update push(String key, Object value)
* Update pushAll(String key, Object[] values)
* Update rename(String oldName, String newName)
* Update set(String key, Ojbect value)
* Update setOnInsert(String key, Object value)
* Update unset(String key)

## Querying Documents in a Collection

我们可以使用Query和Criteria类来表达我们的查询表达式,
它们具有MongoDB运算符的镜像方法名称.例如 lt, lte, is等等.

## Query Distinct Values

## GeoSpatial Queries(地理空间查询)

MongoDB 通过使用 $near, $within, $geoWithin $nearSphere 等运算符支持GeoSpatial查询.
Criteria 类提供了特定于地理空间查询的方法,
还有些形状类(Box, Point, Cricle)与地理空间相关的Criteria方法结合使用.

## Query by Example

## Map-Reduce Operations

# MongDB Repositories


