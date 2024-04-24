# 一对多

数据库中的一对多关系, 最常见也是最重要的关系.

通常数据库用外键例的方式表现:

## 外键列

1.one方的主键作为many方的外键列.
比如: 一个UserInfo拥有多个Account.

```sql
create table userInfo (
    id bigint primary key,
    ....
);
create table account (
    id bigint primary key,
    userInfo_id bigint foreign key references account(id),
    ....
);
```

从数据库结构看, userInfo 并不知到 Account 的信息.
只有Account持有 userInfo 的id, 所以很自然我们映射成为如下Java类

```java
class UserInfo {
    Long id;
    ...
}
class Account {
    Long id;
    UserInfo userInfo;
}
```

我们希望在查找Account时还需要将其对应的UserInfo查找出来(方便数据展示).
为了达到这个目的我们有两种方式:

1. 执行两句select
2. 执行一句select

首先将第一种方式
根据account找出对应的userInfo.
首先查找到Account, 然后根据外键列值查找UserInfo(执行两句SQL select).

```sql
select * from account where id = ?;
select * from userInfo where id = ?;
```

第二种方式, 使用join on进行联表查询.
首先将 account 表和 userInfo 通过外键列联接成为一张表, 然后执行查找.

```sql
select *
from account A
left outer join userInfo U on A.userInfo\_id = U.id
where A.id = U.id;
```

无论采用哪种方式我们都可以在查找一个Account信息时将和其对应的UserInfo给查找出来.

## 多对一

多对一在数据库理论中并没有这个说法, 它是我们在学习Hibernate或ORM时接触到的.
它其实是我们Java类的一个需求, 有时候根据one方查找出和它对应的所以many方是很有用的.
下面 Account 和 UserInfo 的例子并不太恰当
(可以等价考虑员工和部门的关系)

```java
class UserInfo {
    Long id;
    Collection<Account> accounts;
    ...
}
class Account {
    Long id;
}
```

一个UserInfo拥有多个Account, 我们希望在查询UserInfo时同时找出其拥有的账号.
(一个部门拥有多个员工, 通过部门获取其所有员工是非常自然的.方便我们在展示部门时可以一起展示员工)

要加载这样的UserInfo, 我们需要执行两个select

```sql
# 根据主键查找出唯一的 UserInfo
select * from userInfo where id = ?;
# 根据外键关联查找出所有的 Account
select * from Account where userInfo_id = ?;
```

所谓的多对一其实是一对多的反面,
也就是说如果不存在一对多也就不存在多对一.

# 一对一

数据库中的一对一关系

# 多对多

数据库中的多对多关系
