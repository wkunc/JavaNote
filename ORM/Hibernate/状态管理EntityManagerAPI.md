# 状态管理, 持久化上下文

![实体的实例状态](./img/)

1. Transient 瞬时态, 当采用new 的方式构建的实例就处于这个状态, 数据库中没有它对应的行, 持久化上下文中也没有保存它.
2. Persistent 持久态, 当一个瞬时态的实例调用 em.persist() 和 merge() 方法, 那么它将交给持久化上下文管理,
   em会在合适的时间将其保存到数据库中.
3. Removed 移除态, 只有持久态的实例可以变成移除态, 因为我们只能删除数据库中有的值. 当调用 em.remove() 时, em会在合适时间执行
   delete.
4. Detached 游离态, 当一个持久态实例关联的持久化上下文关闭了, 那么它就是游离状态的. 因为em.close()只会关闭上下文,
   无法删除程序中我们持有的引用. 此时我们对应实例进行修改, 并不会反应到数据库中.

# EntityManager

创建一个 EntityManager 就启动了其持久化上下文, 我们可以把它们划等号.

Hibernate 只在必要时才访问数据库(只有在执行SQL时才会从DataSource中获取数据库连接)

## 保存数据(em.persist)

为了在数据库保存Item实例, Hibernate 需要执行 Insert 语句.
当 em 的事务提交时, Hibernate 会刷新该持久化上下文, 这是才会发生Insert.
Hibernate 甚至可以在JDBC级别将 insert 与其他语句一起进行*批处理*(将这些SQL语句放到一起执行).
所以当调用 persist() 方法时并不一定会立刻执行 Insert 语句, 而仅仅会完成ID的分配.
> 注意:
> 但是有特殊情况, 如果说 ID 不是在每次插入前生成的,
> 那么 Insert 语句会立刻执行.以保证可以调用 getId() 方法.
> 比如说我们采用数据库自增的方式, 那么id是在数据库插入后才知道的
> 这时就会立刻执行Insert语句保存对象.
> 如果我们采用 uuid 之类的策略, 由应用程序生成一个不重复的主键值.
> 那么Insert语句将会延后执行.

```java
Item item = new Item("some item");
//
em.persist(item);

// Id 已经分配, Insert 语句不一定执行
Long ITEM_ID = item.getId();
```

## 检索和修改数据(select and update)

通过em从数据库中检索已经持久化的实例.

find() 方法会从一级缓存(EntityManager缓存) 和 二级缓存(真正意义上的缓存, 默认关闭)
中获取, 当缓存中没有才会执行 select 从数据库中获取.

通过find()方法检索到的实例都处于持久态, 所以对这个实例进行修改.
当前事务提交或刷新上下文时, hibernate 就会为这些修改提交 Update 语句

默认情况下 hibernate 会更改所映射的ITEM表的所有列, 以便将为更改的值修改为旧值.
(update * values(?, ?, ?..) from item where id = ?)
因此 Hibernate 可以在启动时就生成这些基本的SQL语句(节省性能);
如果希望SQL语句中只包含修改过的列,
那么在实体类上加Hibernate特有注解
@DynamicInsert,@DynamicUpdate启用动态SQL生成就行了.

```java
Item item = em.find(Item.class, ITEM_ID);
item.setName("job");

// 重复读取, 第一个从数据库中取到, 第二个从二级缓存中取到, 是同一个对象的引用
Item item1 = em.find(Item.class, ITEM_ID);
Item item2 = em.find(Item.class, ITEM_ID);
item1 == item2 // true
```

Hibernate 是如何知道当前的持久态对象是否被修改过呢?
它会保存查询结果的快照, 利用这个快照进行毕竟就可以知道持久态对象是否被修改
以及哪些字段被修改.

### 获得一个引用

如果不希望在加载一个实体实例时访问数据库, 那么可以让 entityManager 尝试获取一个代理.

```java
Item item = em.getReference(Item.class, ITEM_ID);

// Hibernate.initialize(item);

```

getReference() 如果上下文中没有指定id的实例, hibernate不会立刻执行select语句加载对象.
而是返回一个巧妙的代理类.
只有当调用 getName() 之类的方法访问实体属性时,
才会触发select语句的执行. (和lazy加载是同样的原理)
(除了getId()方法这是特殊的, 这个代理对象的id是可以立即获得的. 因为在构建代理时就传入了id, 代理执行select时也需要这个id值,
所以企图获取id不会触发select)

Hibernate提供了一个initialize()静态方法来, 来显式的触发代理对象的加载内部数据

如果说一个代理对象没有*初始化*(即调用getName()或显式加载访问数据库).
并且和它关联的 entityManager 关闭后.
访问这个代理对象企图获取数据时就会报 LazyInitializationException 异常.
因为上下文关闭代理无法执行select语句. 所以记住要在关联的em关闭前完成代理的初始化加载工作.

> 从上面的hibernate代理行为来看, 代理内部应该包含代理实体的id,
> 以及创建代理对象的那个em. 代理的加载行为就可以是通过 em 执行SQL语句
>

## 让数据变成瞬时的(em.remove)

remove() 方法将一个持久态的实体对象从数据库移除, 变成瞬时态.
> 注意remove()要求传入的实体必须是持久态的,即拥有id并且和em有关联.
> 持久态对象无所谓获得的方式, 只要它是持久态的.(如 find,getReference, persist)

被移除的对象就不是持久态, em.contains() 可以验证这一点.
真正的 delete 语句会延迟执行和insert一样, 在当前事务提交时执行.

> em.remove() 方法默认不会移除实体实例的id值(不会修改这个对象).
> 这意味着即使对象对应的数据库行以已经被删除了,
> 但是内存中的这个对象还是保留着原来的所有信息.
> 这些信息是有用的, 如果用户后悔了, 决定撤销这个操作,
> 那么可能希望再次保存已移除的 Item. 就像下面代码里显示的那样,
> 在刷新上下文之前, 调用 persist() 方法以撤销删除操作.
> 如果在配置文件中将 hibernate.use\_dientifier\_rollback 设置为true
> 那么hibernate会在调用 remove() 将实体对象移除后, 将其 id 设置为nul(如果是基本类型就是 0)
> 此时被移除的对象状态和瞬时态一样 (拥有属性值但是没有id),
> 可以重新保存它, 但是原来那行已经被删除了,
> id 也是重新分配的不再是原来那个.

```java
Item item = em.find(Item.class, ITEM_ID);
// Item item = em.getReference(Item.class, ITEM_ID);
em.remove(item);

em.contains(item); // false, 被移除的实体, 不再包含在em内部
```

## 刷新数据

加载实体对象后, 通过某种方式(并不重要)我们知道了有其他人修改了数据库中的数据
调用 refresh() 会引发 Hibernate 执行Select来读取和收集整个结果集,
对应用程序加载的实体对象进行更新信息(如果被修改了的话).
如果对应实体对象代表的行已经被删除,
那么Hibernate会在refresh()方法上抛出EntityNotFoundException异常.

```java
Item item = em.find(Item.class, ITEM_ID);
item.setName("bill");
String oldName = item.getName();

em.refresh(item);
item.getName().equals(oldName) // false
```

## 复制数据

主要讲如何进行数据库之间的复制操作,
比如说读写分离时, 我们需要在一个数据库上进行读, 在另一个数据库上进行写.

省略....

## 在持久上下文中缓存

entityManager是我们的一级缓存.
它拥有持久态的每一个实例的引用.
因此hibernate的缺点就是可能会碰到 OutOfMemoryException 异常
(如果我们在em中加载上千个实体对象,又不打算修改它, 那么很快内存就会溢出).
因为所有的持久态实体对象都保存在内存中, 而且为了能知道哪些需要实体状态被修改了
hibernate还必须保存每个对象的快照(这一点在修改持久态数据中讲过了).

持久化上下文缓存并不会自动清理.
通常, 上下文中会有许多实体实例是偶然出现的(不小心被加载的).
比如: 我需要一个 Account 实体对象, 但是根据 Many-to-One 注解.
和其关联的 UserInfo 也会被加载(在没有设置lazy加载的情况下).

这样的行为会严重影响性能(执行多个Select)并且需要大量内存(保存实例和其快照)
