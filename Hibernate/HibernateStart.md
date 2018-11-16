# The Hibernate configuration file
hibernate.cfg.xml 文件定义了 Hibernate 配置信息

connection.driver\_class, connection.url, connetction.username, connection.password
定义了 JDBC 连接的信息.

connection.pool\_size 配置 Hibernate 内置连接池的大小

dialect(方言) 属性指定 SQL 方言 Hibernate 将转换

# The mapping file
```xml
<class name="Event" table="EVENTS">
	...
</class>
```
