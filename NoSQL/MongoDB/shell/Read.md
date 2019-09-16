# Read Operations

## Select All Document in a Colelction
db.inventory.find( {} );
select * from inventory;

## Specify Equality Condition(指定条件)
使用 <field>:<value> 表达式在 query filter document:
```
db.inventory.find( {status: "D"} );
select * from inventory where status = "D";
```
## Specify Conditions Using Query Operatiors
一个 query filter document 能使用 query operators 来指定条件,如下所示:
```
{ <field>: { <operator>: <value>},... }

db.inventory.find( {status: {$in: ["A", "D"]} } );


select * from inventory where status in ("A", "D");
```

# 投影操作
上面讲的都是相当于 select * from table.
会查出所有字段, 如果我们只需要一部分字段就必须自己显式的指定. 在find方法参数中.

我们都知道在find()方法中第一个参数是一个 query filter document.
第二个参数就是我们需要指定的. 他是一个 projection document. (投影文档)
只需要在投影文档中将字段的值设为1, 它就会被查询出来.

\_id 是默认被查询出来的, 如果不想查询它, 需要显式的关闭,如 :
```
// 第一对{}中代表 query filter document, 第二对{}才是投影文档
db.inventory.find({status : "A"},  {item: 1, status: 1});

select _id, item, status from inventory where status = "A"
```

\_id 是默认被查询出来的, 如果不想查询它, 需要显式的关闭, 如下 :

```
db.inventory.find({status : "A"},  {item: 1, status: 1, _id: 0});

select _id, item, status from inventory where status = "A"

```


# Query and Projection Operators

## Query Selector
Comparison
|name|description|
|----|----------|
|$eq (equal)| 等于
|$gt (greater)| 大于
|$gte (greater equal)| 大于等于
|$in| 在包含在什么里面
|$lt (less than)| 小于
|$lte (less than equal)|小于等于
|$ne (not equal)| 不等于
|$nin (not in)| 

logical

