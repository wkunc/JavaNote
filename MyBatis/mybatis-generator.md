# table

\<table>元素用于选择数据库中的表以进行内省。选定的表将导致为每个表生成以下对象：

MyBatis / iBATIS格式的SQL Map文件

一组构成表格“模型”的类，包括：

用于匹配表的主键的类（如果表具有主键）。

用于匹配表中不在主键和非BLOB字段中的字段的类。如果有主键，则此类将扩展主键。

用于保存表中任何BLOB字段的类（如果有）。此类将根据表的配置扩展前两个类中的一个。

一个类，用于在不同的“by example”方法中生成动态where子句（selectByExample，deleteByExample）。
（可选）DAO接口和类

必须至少指定一个\<table>元素作为\<context>元素的必需子元素。您可以指定无限制的表元素。
>### (Required Attributes)必要参数
>|Attribute|Description
>|---------|-----------
>|tableName|	The name of the database table(数据库表名)

>### Optional Attributes(可选参数)
>***
>|Attribute|Description
>|---------|-----------
>|schema(模式)| not required(不是必须的,如果你的数据库不存在模式)
>|catalog(目录)| not required(不是必须的,如果你的数据库不使用 catalog,或者有默认 catalog, mysql不支持)
>|alias(别名)|
>|domainObjectName(领域对象)|The name will be used to compute generated domain class names and DAO class names.如:foo.Bar会在foo包下生成Bar实体
>|mapperName|这个名字用来命名自动生成的mapping.xml文件和mapper类,没有指定就是 domainObjectName+Mapper
>|sqlProviderName|与mybatis的注解方式相关
>|enableInsert|Signifies whether an insert statement should be generated.The default is true.(是否生成 insert 语句)
>|modelType(模型类型)|用来覆盖全局的modelType

>### Supported Properties(支持属性值)
> ***
> 如\<Property name="" value="">
> |Property Name|Property Values
> |-------------|---------------
> |constructorBased|default=false

## Child Elements
> \<property> (0..N)
> 
> \<generatedKey> (0 or 1)
> 
> \<domainObjectRenamingRule> (0 or 1)
> 
> \<columnRenamingRule> (0 or 1)
> 
> \<columnOverride> (0..N)
> 
> \<ignoreColumn> (0..N)

## domainObjectRenamingRule
和 columnRenamingRule 相似,
mybatis默认用表名命名 POJO ,而表名会有前缀等如sys_user, 这个元素可以去除匹配的前缀生成想要的 POJO 对象名
> ### Required Attributes(必须属性)
> |Attribute|Description
> |---------|-----------
> |searchString|This is a regular expression that defines the substring to be replaced.
> ### Optional Attributes(可选属性)
> |Attribute|Description
> |---------|-----------
> |replaceString|This is a string to be substituted for every occurrence of the search string. If not specified, the empty string is used.
> 列如: \<domainObjectRenamingRule searchString="^Sys" replaceString="" />

## columnOverride
MyBatis Generator (MBG) uses the \<columnOverride> element to change certain attributes(某些属性) of an introspected(反思,内省的) database column from the values that would be calculated by default. This element is an optional child element of the \<table> element.
> ### Required Attributes(必须属性)
> |Attribute|Description
> |---------|-----------
> |column   |The column name of the generated column.
> |sqlStatement|
> ### Optional Attributes(可选属性)
> |Attribute|Description
> |---------|-----------
> |identity|The default is false.
> |type|

## generatedKey
帮你生成在 \<insert> 标签中生成一个 \<selectKey>主要用不能自动生成主键的情况
> ### Required Attributes(必须属性))
> |Attribute|Description
> |---------|-----------
> |column   |The column name of the generated column.
> |sqlStatement|
> ### Optional Attributes(可选属性)
> |Attribute|Description
> |---------|-----------
> |identity|The default is false.
> |type|
