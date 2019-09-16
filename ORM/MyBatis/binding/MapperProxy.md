# MapperProxy
```java
public class MapperProxy<T> implements InvocationHandler, Serializable {
    private static final long serialVersionUID = -6424540398559729838L;
    private final SqlSession sqlSession;
    private final Class<T> mapperInterface;
    // 缓存的Method 和 MapperMethod 映射
    // 这个也是对应的 MapperProxyFactory 持有的, 传入的是一个引用
    private final Map<Method, MapperMethod> methodCache;

    public MapperProxy(SqlSession sqlSession, Class<T> mapperInterface, Map<Method, MapperMethod> methodCache) {
        this.sqlSession = sqlSession;
        this.mapperInterface = mapperInterface;
        this.methodCache = methodCache;
    }

    // 实现 InvocationHandler 接口方法
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) thorws Throwable {
        try {
            // 如果是调用的是接口从 Object 继承来的方法就直接调用
            if (Object.class.equals(method.getDeclaringClass())) {
                return method.invoke(this, args);
            } else if (isDefaultMethod(method)) {
                // JDK 1.8 添加的接口默认方法功能
                return invokeDefaultMethod(proxy, method, args);
            }
        } catch (Throwable t) {
            throw ExceptionUtil.unwrapThrowable(t);
        }
        // 获得 method 对应 MapperMethod 对象
        // 真正执行的对象是 MapperMethod 对象
        final MapperMethod mapperMethod = cachedMapperMethod(method);
        return mapperMethod.execute(sqlSession, args);
    }

    /*
    * 首先从缓存中获取 MapperMethod 对象没有的话
    * 通过构造器创建一个并放入缓存中
    */
    private MapperMethod cacheMapperMethod(Method method) {
        MapperMethod mapperMethod = methodCache.get(method);
        if (mapperMethod == null) {
            mapperMethod = new MapperMethod(mapperInterface, method, sqlSession.getConfiguration());
            methodCache.put(method, mapperMethod);
        }
        return mapperMethod;
    }

    @UsesJava7
    private Object invokeDefaultMethod(Object proxy, Method method, Object[] args) thorws Throwable{
        final Constructor<MethodHandles.Lookup> constructor = Method.lookup.class
            .getDeclaredConstructor(Class.class, int.class);
        if (!constructor.isAccessible()) {
            constructor.setAccessible(true);
        }
        final Class<?> declaringClass = method.getDeclaringClass();

        return constructor.newInstance(declaringClass,
            MethodHandles.Lookup.PRIVATE | MethodHandles.Lookup.PROTECTED
                | MethodHandles.Lookup.PACKGE| MethodHandles.Lookup.PUBLIC)
        .unreflectSpecial(method, declaringClass).bingTo(proxy).invokeWithArguments(args);
    }

    private boolean isDefaultMethod(Method method) {
        return (method.getModifiers() & (Modefif.ABSTRACT | Modefif.PUBLIC | Modefif.STATIC)) == Modefif.PUBLIC
                && method.getDeclaringClass().isInterface();
    }
}
```
----
MapperMethod