# Q&A
开发中碰到过的疑问点

## 为何有时参数名没有正确解析

java8 增加了编译参数 `-parameters` 可以让class中保留参数名, Spring, Mybatis等框架会用这个特性减少参数名映射时的申明工作量

在java8之前需要用注解指定方法中的参数名, 否则在xml中就只能用`#{param1}`这样的参数了
```java
Person listSimpleById(@Param("ids") String id);

// java8开启 -parameters 编译参数之后
Person listSimpleById(String id);
```

异常情况, 单个 `Collection`类型参数的情况下.
运行时会报错, 提示没有ids这个名字的参数
> Parameter 'ids' not found. Available parameters are  \["collection", "list"\]
```java
List<Person> listSimpleByIds(List<String> ids);

```
看起来`-parameters`失效了. 本质原因是Mybatis对单个参数的特殊处理.
单个参数的情况下实际上mybatis是不case这个参数名的.

`org.apache.ibatis.reflection.ParamNameResolver`
```java 
  /**
   * <p>
   * A single non-special parameter is returned without a name.
   * Multiple parameters are named using the naming rule.
   * In addition to the default names, this method also adds the generic names (param1, param2,
   * ...).
   * </p>
   */
  public Object getNamedParams(Object[] args) {
    final int paramCount = names.size();
    if (args == null || paramCount == 0) {
      return null;
    // 如果方法只有单个参数, 且没有用@Param,会直接返回那个参数值
    } else if (!hasParamAnnotation && paramCount == 1) {
      return args[names.firstKey()];
    } else {
      // 其他情况下都是一个参数名和参数值的ParamMap.
      final Map<String, Object> param = new ParamMap<>();
      int i = 0;
      for (Map.Entry<Integer, String> entry : names.entrySet()) {
        param.put(entry.getValue(), args[entry.getKey()]);
        // add generic param names (param1, param2, ...)
        final String genericParamName = GENERIC_NAME_PREFIX + String.valueOf(i + 1);
        // ensure not to overwrite parameter named with @Param
        if (!names.containsValue(genericParamName)) {
          param.put(genericParamName, args[entry.getKey()]);
        }
        i++;
      }
      return param;
    }
  }
```

org.apache.ibatis.session.defaults.DefaultSqlSession
```java
  @Override
  public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
    try {
      MappedStatement ms = configuration.getMappedStatement(statement);
      // select, update, selectCursor, 等方法调用时都会调用wrapCollection, 也就是所有sql执行前, 参数都会经过该方法的处理
      return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
  // 如果参数是集合类型, 替换成 StrictMap 类型. 这里也就是单个集合类型参数的默认名称来源
  private Object wrapCollection(final Object object) {
    if (object instanceof Collection) {
      StrictMap<Object> map = new StrictMap<>();
      map.put("collection", object);
      if (object instanceof List) {
        map.put("list", object);
      }
      return map;
    } else if (object != null && object.getClass().isArray()) {
      StrictMap<Object> map = new StrictMap<>();
      map.put("array", object);
      return map;
    }
    return object;
  }
```

org.apache.ibatis.scripting.defaults.DefaultParameterHandler
```java
  @Override
  public void setParameters(PreparedStatement ps) {
    ErrorContext.instance().activity("setting parameters").object(mappedStatement.getParameterMap().getId());
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    if (parameterMappings != null) {
      for (int i = 0; i < parameterMappings.size(); i++) {
        ParameterMapping parameterMapping = parameterMappings.get(i);
        if (parameterMapping.getMode() != ParameterMode.OUT) {
          Object value;
          String propertyName = parameterMapping.getProperty();
          if (boundSql.hasAdditionalParameter(propertyName)) { // issue #448 ask first for additional params
            value = boundSql.getAdditionalParameter(propertyName);
          } else if (parameterObject == null) {
            value = null;
          // 如果是TypeHandlerRegistry 中可以直接处理的类型就直接用这个该值, 不考虑自定义注解的情况下通常就是基础类型
          } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
            value = parameterObject;
          } else {
            // 这里相当与是ParamMap类型, 或者单个参数时是用户的pojo类型
            // 利用 MetaObject 反射工具获取pojo或者map里对应的属性名的值
            MetaObject metaObject = configuration.newMetaObject(parameterObject);
            value = metaObject.getValue(propertyName);
          }
          TypeHandler typeHandler = parameterMapping.getTypeHandler();
          JdbcType jdbcType = parameterMapping.getJdbcType();
          if (value == null && jdbcType == null) {
            jdbcType = configuration.getJdbcTypeForNull();
          }
          try {
            typeHandler.setParameter(ps, i + 1, value, jdbcType);
          } catch (TypeException | SQLException e) {
            throw new TypeException("Could not set parameters for mapping: " + parameterMapping + ". Cause: " + e, e);
          }
        }
      }
    }
  }
```
