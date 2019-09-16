类的层次结构hibernate表示
# Table Per Hierarchy
![](img/Hi.PNG)
一个类层次(一个继承树)一个表的情况下, hibernate 添加了一个鉴别列,
用于记录指定对应的类型.使用 <discriminator/> 子元素指定此项
```xml
<hibernate-mapping>
    <class name="com.wkunc.entity.Employee" table="employee" discriminator-value="emp">
        <id name="id" column="id">
            <generator class="native"/>
        </id>
        <discriminator column="type" type="java.lang.String"/>
        <property name="name" column="name"/>

        <subclass name="com.wkunc.entity.ContractEmployee" discriminator-value="com_emp">
            <property name="contractDuration"/>
            <property name="payPerHour"/>
        </subclass>
        <subclass name="com.wkunc.entity.RegularEmployee" discriminator-value="reg_emp">
            <property name="salary"/>
            <property name="bonus"/>
        </subclass>
    </class>
</hibernate-mapping>
```
使用<subclass/>指定子类
![](img/table.PNG)
在这种情况下, hibernate 会将所有属性整合成一张表中的字段.
并且有一个 type 字段指定每一条记录是什么类型的.
但是由于不同子类中的字段不同,会导致表中有许多null值(ps:没法加上非空约束).

---
类层次结构中的每一个类都对应一张表.
id的生成策略不能使用native. 因为这三个类都是员工, 所以id需要连续,
mysql自增主键很显然达不到要求. 所以将id交给hibernate来维护
```xml
<hibernate-mapping>
    <class name="com.wkunc.employee.Employee" table="employee2">
        <id name="id" column="id">
            <generator class="increment"/>
        </id>
        <property name="name" column="name"/>

        <union-subclass name="com.wkunc.employee.ContractEmployee" table="conemp2">
            <property name="contractDuration"/>
            <property name="payPerHour"/>
        </union-subclass>
        <union-subclass name="com.wkunc.employee.RegularEmployee" table="regemp2">
            <property name="salary"/>
            <property name="bonus"/>
        </union-subclass>
    </class>
</hibernate-mapping>
```

---
还是一个类一张表但是,
```xml
<hibernate-mapping>
    <class name="com.wkunc.employee.Employee" table="employee3">
        <id name="id" column="id">
            <generator class="increment"/>
        </id>
        <property name="name" column="name"/>

        <joined-subclass name="com.wkunc.employee.ContractEmployee" table="contemp3">
            <key column="eid"/>
            <property name="payPerHour"/>
            <property name="contractDuration"/>
        </joined-subclass>
        <joined-subclass name="com.wkunc.employee.RegularEmployee" table="regemp3">
            <key column="eid"/>
            <property name="salary"/>
            <property name="bonus"/>
        </joined-subclass>
    </class>
</hibernate-mapping>
```
