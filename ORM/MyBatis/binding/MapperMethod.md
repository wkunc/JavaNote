# MapperMethod

```java
public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    switch (command.getType()) {
        case INSERT: {
            Object param = method.convertArgsToSqlCommandParam(args);
            result = rowCountResult(sqlSession.insert(command.getName(), param));
            break;
        }
        case UPDATE: {
            Object param = method.convertArgsToSqlCommandParam(args);
            result = rowCountResult(sqlSession.update(command.getName(), param));
            break;
        }
        case DELETE: {
            Object param = method.convertArgsToSqlCommandParam(args);
            result = rowCountResult(sqlSession.delete(command.getName(), param));
            break;
        }
        case SELECT:
            if (method.returnsVoid() && method.hasResultHandler()) {
                executeWithResultHandler(sqlSession, args);
                result = null;
            } else if (method.returnsMany()) {
                result = executeForMany(sqlSession, args);
            } else if (method.returnsMap()) {
                result = executeForMap(sqlSession, args);
            } else if (method.returnsCursor()) {
                result = executeForCursor(sqlSession, args);
            } else {
            Object param = method.convertArgsToSqlCommandParam(args);
            result = sqlSession.selectOne(command.getName(), param);
            }
            break;
        case FLUSH:
            result = sqlSession.flushStatements();
            break;
        default:
            throw new BindingException("Unknown execution method for: " + command.getName());
    }
    if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
    throw new BindingException("Mapper method '" + command.getName() 
        + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
}
return result;
}
```

SqlCommand, 是MapperMethod的内部静态类, 用于标识对应的SQL语句的id.
它在构造器中会解析出Mapper类对应方法对应的xml文件中的SQL语句
```java
public SqlCoommand(Configuration configuration, Class<?> mapperInterface, Method method) {
    final String methodName = method.getName();
    // 申明方法的类(ps: 不一定是mapperInterface, 还可能是其父接口)
    final Class<?> declaringClass = method.getDeclaringClass();
    MappedStatement ms = resolveMappedStatement(mapperInterface, methodName, declaringClass, configuration);
    if (ms == null) {
        if (method.getAnnotation(Fulsh.class) != null) {
            name = null;
            type = SqlCommandType.FLUSH;
        } else {
            name = ms.getId();
            type = ms.getSqlCommandType();
            if (type == SqlCommandType.UNKNOWN) {
                throw new BindingException("Unknow execution method for: " + name);
            }
        }
    }
}

private MappedStatement resolveMappedStatement(Class<?> mapperInterface, String methodName, 
        Class<?> declaringClass, Configuartion configuration) {
    // 这个就是对应的方法应匹配的ID
    String statementId = mapperInterface.getName() + "." + methodName;
    // 通过configuration判断是否有这个语句, 有可能没有的
    if (configuration.hasStatement(statementId)) {
        return configuration.getMappedStatement(statementId);
    } else if (mapperInterface.equals(declaringClass)) {
        // 如果 mapperInterface 和 declaringClass 是同一个,就说明xml中没有对应的SQL语句.
        return null;
    }
    // 在 mapperInterface 和 declaringClass 不同的情况下还要考虑遍历 mapper接口的父接口
    for (Class<?> supperInterface : mapperInterface.getInterfaces()) {
        if (declaringClass.isAssignableFrom(superInterface)) {
            MappedStatement ms = resolveMappedStatement(supperInterface, methodName,
                    declaringClass, configuration);
            if (ms != null) {
                return ms;
            }
        }
    }
    return null;
}
```
ParamNameResolver
```java
public ParamNameResolver (Confgiuration config, Method method) {
    final Class<?>[] paramTypes = method.getParameterTypes();
    final Annotation[][] paramAnnotations = method.getParameterAnnotations();
    final SortedMap<Integer, String> map = new TreeMap<>();
    int paramCount = paramAnnotation.length;
    for (int paramIndex = 0; paramIndex < paramCount; paramIndex++) {
        if (isSpecialParameter(paramTypes[paramIndex])) {
            continue;
        }
        String name = null;
        for (Annotation annotation : paramAnnotations[paramIndex]) {
            if (annotation instanceof Param) {
                hasParamAnnotaion = true;
                name = ((Param) annoation).value();
            }
        }
        if (name == null) {
            if (config.isUseActualParamName()) {
                name = getActualParamName(method, paramIndex);
            }
            if (name = null) {
                name = String.valueOf(map.size());
            }
        }
        map.put(paramIndex, name);
        names = Collections.unmodifiableSortedMap(map);
    }
}

```

MethodSignature, 是对 方法签名的抽象
```java
    private final boolean returnsMany; // 返回值是否为 collection 类型或数组类型
    private final boolean returnsMap; // 返回值是否为 Map
    private final boolean returnsVoid; // 返回值是否为 Void 
    private final boolean returnsCursor; // 返回值是否为 Cursor 类型
    private final Class<?> returnType; // 返回值类型
    private final String mapKey;       // 如果返回值为 Map, 则这个字段记录了为key的列名
    private final Integer resultHandlerIndex; // 用来标记 ResultHandler 类型参数的位置
    private final Integer rowBoundsIndex;     // 用来标记 RowBounds 类型参数的位置
    private final ParamNameResolver paramNameResolver; // 该方法对应的 ParamNameResolver 对象
```

```java
public MethodSignature(Configuration configuration, Class<?> mapperInterface, Method method) {
    Type resolvedReturnType = TypeParameterResovler.resolveReturnType(method, mapperInterface);
    if (resolvedReturnType instanceof Class<?>) {
        this.returnType = (Class<?>) resolvedReturnType;
    } else if (resolvedReturnType instanceof ParameterizedType) {
        this.returnType = (Class<?>) ((ParameterizedType)resolvedReturnType);
    } else {
        this.returnType = mehthod.getReturnType();
    }
    this.returnsVoid = void.class.equals(this.returnType);
    this.returnsMany = configuration.getObjectFactory().isCollection(this.returnType) || this.returnType.isArray();
    this.returnCursor = Cursor.class.equals(this.returnType);
    this.mapKey = getMapKey(method);
    this.returnsMap = this.mapKey != null;
    this.rowBoundsIndex = getUniqueParamIndex(method, RowBounds.class);
    this.resultHandlerIndex = getUniqueParamIndex(method, ResultHandler.class);
    this.paramNameResolver = new ParamNameResolver(configuration, method);
}
```