# Core concepts
介绍Spring Data中的核心概念, 这些概念不管是在JPA, 还是MongoDB等各种地方都是一致的.

Spring Data repository 抽象中的中央接口是 Repository, 
它将域类以及域的ID类型作为类型参数进行管理.
此接口主要用作标记接口, 用于捕获想要使用的类型,
帮助你发现扩展此接口的接口.

CrudRepository为正在管理的实体类提供复杂的CRUD功能.
```java
public interface CrudRepository<T, ID> extends Repository<T, ID> {
    // 插入
    <S extends T> S save(S entity);
    <S extends T> Iterable<S> saveAll(Iterable<S> entities);

    // 查找
    Optional<T> findById(S entity);
    Iterable<T> findAll();
    Iterable<T> findAllById(Iterable<ID> ids);

    // 删除
    void delete(T entity);
    void deleteById(ID id);
    void deleteAll(Iterable<? extends T> entities);
    void deleteAll();

    long count();
    boolean existsById(ID primaryKey);
}
```
在CrudRepository之上又一个 PagingAndSoringRepository抽象, 
它添加了额外的方法来简化对实体分页的访问.

```java
//分页的Repository
public interface PagingAndSortingRepository<T, ID> extends CrudRepository<T, ID> {

	Iterable<T> findAll(Sort sort);

	Page<T> findAll(Pageable pageable);
}
```
# Query Methods
 步骤
1. 定义Repository接口
```java
interface PersonRepository extends Repository<Person, Long> {...}
```
2. 在接口上声明查询方法
```java
interface PersonRepository extends Repository<Person, Long> {
    List<Person> findByLastname(String lastname);
}
```
3. 设置Spring为这些接口创建proxy实例.
```java
@EableJparepositories
class Config {}
```
4. 注入 repository 实例,并使用它.

* [Defining Repository Interfaces](#Definig Repository Interfaces)
* [Defining Query Methods](#DefinigQueryMethods)
* [Creating Repository Instances]()
* [Custom Implementations for Spring Data Repositories]()

# Defining Repository Interfaces
这个接口必须继承spring-data-commons模块下的
org.springframework.data.repository.Repository 接口或子接口,
并指定 domain 类型(实体类型) 和 ID类型(主键类型). 

如果要公开该domain类型的CRUD方法, 请扩展CurdRepository.
```java
// 定义了一个操作实体是 User, 主键是 Long 的 repository
public interface UserRepository extends Repository<User, Long> {
    ...
}
```
## Fine-tuning Repository Definition(微调存储库定义)
通常, 存储库接口扩展了Repository, CrudRepository 或PagingAndSortingRepository.
如果你不想扩展Spring Data接口, 不想显式的继承和Spring提供的接口耦合.
还可以使用@RepostiroyDefinition注解存储库接口.

## Null Handling of Repository Methods(存储库方法的 Null值处理)
从Spring Data2.0以来, 返回单个实例的存储库CRUD方法使用 Java8 的Optional来指示可能缺少值.
除此之外, Spring Data还支持在查询方法上返回以下包装类型:
* com.google.common.base.Optional
* scala.Option
* io.vavr.contral.Option

或者, 查询方法可以选择不使用包装类型. 然后通过返回null来指示缺少查询结果.
包装返回集合, 包装器, Stream的存储库方法永远不会返回null, 而是返回相应的空表示.
### Nullable Annotaions(可空性注解)

## Using Repositories with Multiple Spring Data Modules(多模块Repsoitory)


# Defining Query Methods
存储库代理有两种方法可以从方法名派生出特定于 DB的查询.
1. 通过直接从方法名称派生查询
2. 通过使用手动定义

## Query Lookup Strategies
这种方式有三种(Strategy)策略.
1. create
2. use\_declared\_query
3. create\_if\_not\_found (默认策略, 结合了上面两个)

CREATE: 尝试从方法名称构造特定于存储的查询
USE\_DECLARED\_QUERY: 尝试查找声明的查询, 如果找不到, 则抛出异常.
可以通过注解来声明, 也可以用其他方法来声明查询.看具体实现

## Query Creation
prefixes(前缀) find..By, read..By, query..By, get...By, count..By
这几个前缀都是同一个意思代表select(除了count 它代表count语句).
一般来说find(动词)和By之间会插入entity(主题).
比如 findUserBy 这样表达这个方法是查询User的方法.
但是这里的entity只是为了更加好读, 因为一个 Repository 操作的实体, 已经从定义时的泛型中声明了. 
如: UserRepository extends Repository<User, Long>
所以这里的主题名完全不影响SpringData解析这个方法名.
不管它是什么, 甚至没有也是可以的如: findById.

> 第一个By用作分隔符以指示实际条件(查询条件)的开始.
> 条件之间用 AND或OR 组合在一起.

> 属性表达式中可以使用 Between, LessThan, GreaterThan, Like 之类的运算符.
> 支持的运算符可能因数据存储而异. mysql, mongodb会有不同的运算符

> IgnoreCase 支持为各个属性设置忽略大小写
> findByLastnameIgnoreCase() 查询通过Lastname相等, 并且忽略它的大小写
> findByLandnameAndFirstnameAllIgnoreCase() 查询通过Lastname, Firstname相等, 并且忽略它们的大小写

> 还可以通过 OrderBy 和 Asc/Desc 来静态排序
> findAllOrderByIdAsc()
> 如果要创建动态排序的方法, 请看下面的[特殊参数处理](#特殊参数处理)

## Prperty Expressions (属性表达式)
属性表达式中只能引用被管理entity(实体)的直接属性.
在创建查询时, 确保表达式中只有托管域类的属性.
但是, 也可以通过遍历嵌套属性来定义约束.

考虑下面的方法签名:
```java
List<User> findByAddressZipCode(ZipCode zipCode);
```
User 有一个 Address , Address有一个ZipCode.

解析器会先查找User中是否有个 AddressZipCode 属性.
如果有则用它, 如果没有就从**右侧**按驼峰命名分割.
AddressZip 和 Code 判断是否有AddressZip有一个Code.
如果没有就把分割点移动到下一个.
会变成 Address, ZipCode.

这个算法在大多数情况下管用, 但是有时也可能选择错误的属性.
假设上面的例子中: User 刚好也有一个 AddressZip 属性. 
算法会在第一个点拆分并选择错误的属性.
因为接下来 AddressZip 可能没有Code属性

解决这种歧义最好的方式就是声明分割点(使用\_)
```java
List<User> findByAddress_ZipCode(ZipCode zipCode);
```
这样就不会有问题了

## Special parameter handling 特殊参数处理
SpringData 还可以识别某些特定类型(如 Pageable, Sort).
以动态的对查询应用*分页和排序*.

```java
Page<User> findByLastname(String lastname, Pageable pageable);
Slice<User> findByLastname(String lastname, Pageable pageable);
List<User> findByLastname(String lastname, Sort sort);
List<User> findByLastname(String lastname, Pageable pageable);
```
org.springframework.data.domain.Pageable
org.springframework.data.domain.Sort

## Limiting Query Results
查询方法的结果可以使用 first 或 top 关键字来限制, 它们是同一个意思可以互换使用.
可选的数值可以附加到top或first, 以指定返回的结果集大小.
如果省略该数字, 则假定结果大小为1.
下面示例演示了如何现在查询大小.
```java
User findFirstByOrderByLastnameAsc();

User findTopByOrderByAgeDesc();

Page<User> queryFirst10ByLastname(String lastname, Pageable pageable);

Slice<User> findTop3ByLastname(String lastname, Pageable pageable);

List<User> findFirst10ByLastname(String lastname, Sort sort);

List<User> findTop10ByLastname(String lastname, Pageable pageable);
```

# Creating Repository Instances

## XML
```xml

```

## JavaConfig
在配置类上加一个@Enable${store}Repositories注解就行.
```java
@Configuration
@EnableJpaRepositories("com.acme.repositories")
class ApplicationConfiguration {

  @Bean
  EntityManagerFactory entityManagerFactory() {
    // …
  }
}
```
## 独立使用
我们还可以在Spring容器之外使用, Spring Data提供了一个特定于持久性技术的
RepositoryFactory, 可以按如下方式使用它:
```java
RepositoryFactorySupport factory = ...// Instantiate factory here
UserRepository repository = factory.getRepository(UserRepository.class);
```
这其实也是底层的原理, 用工厂创建repository的代理对象.

# Custom Implementations for Spring Data Repositories(自定义实现Repository)
当方法需要不同的行为或者无法通过查询派生实现时, 则需要提供自定义实现.
(如果我想执行的查询无法用Spring Data 提供的那套基于 MethodName 生成查询的方式实现.
那么, 我们可以自己提供那个方法的实现, 然后和 Repository 接口放在一起)

```java
interface CustomizedUserRepository {
  void someCustomMethod(User user);
}
class CustomizedUserRepositoryImpl implements CustomizedUserRepository {

  public void someCustomMethod(User user) {
    // Your custom implementation
  }
}
```
到这里都与Spring无关, 我们定义了一个接口, 里面放哪些Spring无法自己实现的方法.
然后自己用子类实现了它们.下面我们定义总的域对象存储库.
```java
interface UserRepository extends CrudRepository<User, Long>, CustomizedUserRepository {

  // Declare query methods here
}
```
我们的接口扩展了CrudRepository 和 CustomizedUserRepository 接口.
这还不够, 我们还要告诉Spring, CustomizedUserRepository接口的实现在哪里.

体现约定优于配置的思想, 默认会在存储库扫描包中查找 repository-impl-postfix属性指定的后缀的实现类.
默认后缀是 Impl, 也就是说我们自定义接口的实现类只要是 接口名-Impl 加上一个后缀就可以被识别出来非常方便.

也可以显式的声明....


# Spring Data 扩展

## QSL
## WebSupport

