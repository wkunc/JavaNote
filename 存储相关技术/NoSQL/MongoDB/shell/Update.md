# Update Operations
MongoDB 提供了以下方法来进行 update. 其实还有很多方法也可以用来更新. https://docs.mongodb.com/manual/reference/update-methods/
db.collection.updateOne();
db.collection.updateMany();
db.collection.replaceOne(); // 替换
方法参数应该是, 一个条件, 一个update

为了更新一个文档, MongDB 提供了 update operators, 如 $set. 去改变字段值.
更新操作符用在更新方法中.

```
updateOne(
    {
        <update operator1> : {<field1> : <value1>, ...},
        <update operator2> : {<field2> : <value2>, ...},
        ...
    }
);
```

# Update Behavior
1. 无法改变 \_id 字段, 无论是update还是replace
2. 所有的写操作都是原子性的. 所以update操作也是原子性的.
3. MongoDB在写操作之后保留文档字段的顺序. 除了更新字段名的update操作.和2.6版本以下的MongoDB.
3. Write Concern.
5. Upsert, 如果update方法中带有 upsert : true 参数. 那么在没有对应文档可更新时会插入当前文档.

# Update Operators

Fields
|Name| Description
|----|------------|
|$currentDate| 为一个字段的值设置为 current date as a Date or Timestamp.
|$inc| 按指定的次数 increments (增长) 字段的值.
|$min| 仅当指定值小于字段值时才更新字段.
|$max| 仅当指定值大于字段值时才更新字段.
|$mul| 将字段的值乘以指定的值.
|$rename| 重命名字段
|$set| 设置指定字段的值
|$setOnInsert| 
|$unset| 从一个document中删除指定字段.

Array
Operators
|Name| Description
|----|------------|
|$   |
|$[] |
|$[<identifier>]|
|$addToSet| 添加一个元素到数组, 当且仅当set中不存在这个元素.
|$pop| 移除数组的第一项或最后一项
|$pull| 移除所有匹配指定query的数组元素.
|$push| 添加一个元素到数组.
|$pullAll| 移除所有值为指定值的数组元素.(和上面的区别是这个是指定值, 精准匹配)

Modifiers(修饰符, 只用在某些 operator 符上)
|Name| Description
|----|------------|
|$each| 用在 $push, $addToSet
|$position|用在 $push
|$slice|用在 $push
|$sort|用在 $push

Bitwise
|Name| Description
|----|------------|
|$bit|
