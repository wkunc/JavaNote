# 映射继承关系
* 每个具体类使用一张表并且使用默认的运行时多态行为
* 每个具体类使用一张表但完全舍弃SQL架构的多态和继承关系, 将SQL UNION 查询用于运行时多态行为
> 此时数据库中有一张 hibernate\_sequence 表用来存下标
* 每个类层次结构使用一个表: 通过反规范化SQL架构来启用多态并且依赖基于行的区别来判定超类还是子类
* 每个子类使用一个表: 将 is a(继承) 关系表示为 has a(外键)关系, 并使用 SQL JOIN 操作

# MapperSuperclass

每个具体子类一张独立的表, 父类中的字段会映射到每个子类表中.
可以在子类上使用 @AttributeOverride 注解来覆盖父类中的定义.

其实也可以在父类中声明对象标识符, 使用一个共享的列名称和生成策略应用于所有子类.
这样就不必重复它了. 不管这是可选的. 所以我在示例中没有这样做.

```java
@MappedSuperclass
public abstract class BillingDetails {

    protected String owner;

    public String getOwner() {
        return owner;
    }

    public void setOwner(String owner) {
        this.owner = owner;
    }
}

@Entity
public class BankAccount extends BillingDetails {

    @Id
    @GeneratedValue
    private Long id;

    String account;
    String bankname;
    String swift;

    // getter setter
}

@Entity
@AttributeOverride(
    name = "owner",
    column = @Column(name ="CC_OWNER", nullable = false))
public class CreditCard extends BillingDetails {

    @Id
    @GeneratedValue
    private Long id;

    String cardNumber;
    String expMonth;
    String expYear;

    // getter setter
}
```

## Table per class 
表看起来和上面是一样的.
JPA标准指定了 TABLE\_PER\_CLASS 是可选的, 所以并非所有的JPA实现都支持这个策略.

```java
@Entity
@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)
public abstract class BillingDetails {

    @Id
    @GeneratedValue
    private Long id;

    protected String owner;

    public String getOwner() {
        return owner;
    }

    public void setOwner(String owner) {
        this.owner = owner;
    }
}

@Entity
public class BankAccount extends BillingDetails {

    String account;
    String bankname;
    String swift;

    // getter setter
}

@Entity
public class CreditCard extends BillingDetails {

    String cardNumber;
    String expMonth;
    String expYear;

    // getter setter
}
```

## SINGLE_TABLE

```java
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "BD_TYPE")
public abstract class BillingDetails {

    @Id
    @GeneratedValue
    private Long id;

    protected String owner;

    public String getOwner() {
        return owner;
    }

    public void setOwner(String owner) {
        this.owner = owner;
    }
}

@Entity
public class BankAccount extends BillingDetails {

    String account;
    String bankname;
    String swift;

    // getter setter
}

@Entity
public class CreditCard extends BillingDetails {

    String cardNumber;
    String expMonth;
    String expYear;

    // getter setter
}
```

## JOINED

```java
@Entity
@Inheritance(strategy = InheritanceType.JOINED)
public abstract class BillingDetails {

    @Id
    @GeneratedValue
    private Long id;

    protected String owner;

    public String getOwner() {
        return owner;
    }

    public void setOwner(String owner) {
        this.owner = owner;
    }
}

@Entity
public class BankAccount extends BillingDetails {

    String account;
    String bankname;
    String swift;

    // getter setter
}

@Entity
public class CreditCard extends BillingDetails {

    String cardNumber;
    String expMonth;
    String expYear;

    // getter setter
}
```

## 混合继承策略 



# 多态关联

# 映射集合和实体关联
重点是**@ElementCollecction** 注解
这个注解用于**值类型**元素的集合
映射 Set
@CollectionTable 和 @Column 用来重命名
```java
@Entity
public class Item {
    @ElementCollection
    @CollectionTable(
        name = "IMAGE",
        joinColumns = @JoinColumn(name = "ITEM_ID"))
    @Column(name = "filename")
    protected Set<String> images = new HashSet<>();
}
```


```java
@Entity
public class Item {
    @ElementCollection
    @CollectionTable( name = "IMAGE")
    @Column(name = "filename")
    @org.hibernate.annotations.CollectionId(
        Columns = @Column(name = "IMAGE_ID"),
        type = @org.hibernate.annotation.Type(type = "long"),
        generator = Constants.ID_GENERATOR //有问题
        )
    protected Collection<String> images = new ArrayList<>();
}
```


```java
@Entity
public class Item {
    @ElementCollection
    @CollectionTable( name = "IMAGE")
    @Column(name = "filename")
    @OrderColunm
    protected Collection<String> images = new ArrayList<>();
}
```


映射一个Map
```java
@Entity
public class Item {
    @ElementCollection
    @CollectionTable( name = "IMAGE")
    @MapKeyColumn(name = "FILENAME")
    @Column(name = "IMAGENAME")
    protected Map<String, String> images = new HashMap<>();
}
```


