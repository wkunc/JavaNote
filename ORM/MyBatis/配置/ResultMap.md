#ResultMap

## association

关联元素处理"有一个类型"的关系(就是包含一个引用).
比如说 Account 拥有一个 UserInfo.

不同之处在于, 需要告诉MyBatis如何加载关联:
MyBatis 有两种加载方式:

1. 嵌套Select查询: 通过执行另一个SQL映射语句来加载期望的复杂类型.
2. 嵌套结果映射:

### 嵌套Select查询

```xml
<resultMap id="myset" type="Account" autoMapping="false">
    <constructor>
        <idArg column="account_id" javaType="int"/>
    </constructor>

    <result property="username" column="account_username"/>
    <result property="password" column="account_password"/>
    <result property="authorities" column="account_auth"/>

    <result property="enable" column="account_enable"/>
    <result property="credentialsNonExpired" column="account_cne"/>
    <result property="accountNonLocked" column="account_anl"/>
    <result property="accountNonExpired" column="account_ane"/>

    <association property="userInfo" javaType="UserInfo" select="findUserInfoById" column="userInfo_id" fetchType="lazy">
    </association>
</resultMap>

<select id="findAccountById" parameterType="long" resultType="Account">
    select * from account where id = #{id}
</select>

<select id="findUserInfoById" parameterType="long" resultType="UserInfo">
    select * from userInfo where id = #{id}
</select>
```

我们有两个 select 查询语句: 一个用来加载 Account 一个用来加载 UserInfo,
并且 Account 的结果映射描述了应该使用*findUserInfoById*语句加载它的 UserInfo 属性.
其他所有属性会被自动加载, 只要它的属性名和例名一样.

这样的方式很简单, 但在大型数据集或大型数据表上表现不佳.
这个问题被称为 "N+1" 问题(在hibernate中也有):
指的是这样的行为:

1. 执行了一个单独的SQL语句来获取结果的一个列表(1) select * from account where enable = true;
2. 对列表的每条记录, 执行了一个select查询来加载详细信息(N) select * from userInfo where id = ?;

这个问题会导致执行上千条的SQL语句, 这是不希望看到的.
Mybatis 提供了延迟加载, 将大量语句分散开来.
然而如果加载列表后立即遍历以获得嵌套的数据,
就会触发所有的延迟查询, 性能就会变的很糟糕.

所以还存在另一种方法

### 关联的嵌套结果映射

```xml
<resultMap id="accountMap" type="Account" autoMapping="false">
    <constructor>
        <idArg column="account_id" javaType="int"/>
    </constructor>

    <result property="username" column="account_username"/>
    <result property="password" column="account_password"/>
    <result property="authorities" column="account_auth"/>

    <result property="enable" column="account_enable"/>
    <result property="credentialsNonExpired" column="account_cne"/>
    <result property="accountNonLocked" column="account_anl"/>
    <result property="accountNonExpired" column="account_ane"/>

    <association property="userInfo" javaType="UserInfo">
        <id property="id" column="u_id"/>
        <result property="nickname" column="nickname"/>
        <result property="introduction" column="introduction"/>
        <result property="url" column="url"/>
        <result property="create" column="createDate"/>
        <result property="modified" column="modified"/>
    </association>
</resultMap>

<select id="findAccountById" parameterType="long" resultMap="accountMap">
    select
    A.id as id,
    A.username as username,
    A.password as password,
    A.type as type,
    A.authorities as authorities
    A.enabled as enabled,
    A.account_non_expired as accountNonExpired
    A.account_non_locked as accountNonLocked
    A.credentials_non_expired as credentialsNonExpired,

    U.id as u_id,
    U.nickname as nickname,
    U.introduction as introduction,
    U.create_date as createDate,
    U.modified_date as modified,
    from
    account A left outer join userInfo u on A.userInfo_id = U.id
    where A.id = #{id};
</select>
```

就是利用联表查询, 做出一个笛卡尔积表大概会是这样
A.id|A.username|A.password|A.enable|A.userInfo\_Id|...|U.id|U.nickname|U.introduction|U.createDate|

然后每一条的A.userInfo\_Id会和后面的 U.id 相同.
相当于将每个Account对象和其对应的UserInfo放在同一行中.
接下来只需要取出每个值放置到对应位置.

# ResultMap 解析过程

在总的配置加载时, 解析 mappers 节点时会加载对应的 mapper.xml.
并用 XMLMapperBuilder 开始每一个 mapper.xml的解析.
在解析单个映射文件时会解析到 <result/> 节点.

```java
  private ResultMap resultMapElement(XNode resultMapNode) throws Exception {
    return resultMapElement(resultMapNode, Collections.<ResultMapping> emptyList());
  }

  private ResultMap resultMapElement(XNode resultMapNode, List<ResultMapping> additionalResultMappings) throws Exception {
    ErrorContext.instance().activity("processing " + resultMapNode.getValueBasedIdentifier());
    String id = resultMapNode.getStringAttribute("id",
        resultMapNode.getValueBasedIdentifier());
    String type = resultMapNode.getStringAttribute("type",
        resultMapNode.getStringAttribute("ofType",
            resultMapNode.getStringAttribute("resultType",
                resultMapNode.getStringAttribute("javaType"))));
    String extend = resultMapNode.getStringAttribute("extends");
    Boolean autoMapping = resultMapNode.getBooleanAttribute("autoMapping");
    Class<?> typeClass = resolveClass(type);
    Discriminator discriminator = null;
    List<ResultMapping> resultMappings = new ArrayList<ResultMapping>();
    resultMappings.addAll(additionalResultMappings);
    List<XNode> resultChildren = resultMapNode.getChildren();
    for (XNode resultChild : resultChildren) {
      if ("constructor".equals(resultChild.getName())) {
        processConstructorElement(resultChild, typeClass, resultMappings);
      } else if ("discriminator".equals(resultChild.getName())) {
        discriminator = processDiscriminatorElement(resultChild, typeClass, resultMappings);
      } else {
        List<ResultFlag> flags = new ArrayList<ResultFlag>();
        if ("id".equals(resultChild.getName())) {
          flags.add(ResultFlag.ID);
        }
        resultMappings.add(buildResultMappingFromContext(resultChild, typeClass, flags));
      }
    }
    ResultMapResolver resultMapResolver = new ResultMapResolver(builderAssistant, id, typeClass, extend, discriminator, resultMappings, autoMapping);
    try {
      return resultMapResolver.resolve();
    } catch (IncompleteElementException  e) {
      configuration.addIncompleteResultMap(resultMapResolver);
      throw e;
    }
  }
```
