# binding 模块
这个模块的功能就是实现神奇的将sql语句和mapper接口连接起来的关键
MyBatis会自己产生对应Mapper接口的代理对象, 注册到 MapperRegistry中

# MapperRegistry&MapperProxyFactory
```java
public class MapperRegistry {
    // Configuration 是 MyBatis 配置文件在Java中的表示
    private final Configuration config;
    // 一个Map记录了Mapper接口和其对应的代理工厂
    private final Map<Class<?>, MapperProxyFactory<?>> knownMappers = new HashMap<>();

    public MapperRegistry(Configuration config){
        this.config = config;
    }

    /*
    * 这个方法会根据传入的 Mapper.class 类型
    * 查找 knownMappers 中是否有对应的记录,
    * 如果有就调用对应的 Mapper代理工厂创建出一个指定 Mapper接口的代理对象
    */
    public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
        final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
        if (mapperProxyFactory == null) {
            throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
        }
        try {
            return mapperProxyFactoty.newInstance(sqlSession);
        } catch (Exception e) {
            throw new BindingException("Error getting mapper instance. Cause: " + e, e);
        }
    }

    // 指定的Mapper接口是否已知
    public <T> boolean hasMapper(Class<T> type) {
        return knownMappers.containsKey(type);
    }

    /*
    * 这是一个注册方法(ps: 虽然它不叫register)
    * 注册的本质就是向 knownMappers 中添加
    */
    public <T> void addMapper(Class<T> type) {
        // 这里就强制要求传入的 class 一定是个接口
        if (type.isInterface()) {
            // 如果传入的接口已经注册就会报错
            if (hasMapper(type)) {
                throw new BindingException("Type " + type + " is alreader known to the MapperRegistry.");
            }
            boolean loadCompleted = false;
            try {
                //向 MapperRegistry 注册
                knownMappers.put(type, new MapperProxyFactory<T>(type));
                // 这一步
                MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
                parser.parse();
                loadCompleted = true;
            } finally {
                // 如果前面抛出了异常就代表加载失败了
                if (!loadCompleted) {
                    knownMappers.remove(type);
                }
            }
        }
    }

    pubilc Collection<Class<?>> getMappers(){
        return Collections.unmodifiableConllection(knownMappers.keySet());
    }

    /*
    * 这个方法是 3.2.2添加的, 好像和 MyBatis 的包扫描功能有关
    */
    public void addMappers(String packageName, Class<?> superType) {
        ResolverUtil<Class<?>> resolverUtil = new ResolverUtil<>();
        resolverUtil.find(new ResolverUtil.IsA(superType), pacakageName);
        Set<Class<? extends Class<?>>> mapperSet = resolverUtil.getClasses();
        for (Class<?> mapperClass : mapperSet) {
            addMapper(mapperClass);
        }
    }
}
```
----
MapperProxyFactory
很显然这个类是MyBatis的Mapper代理类生成工厂
```java
public class MapperProxyFactory<T> {
    // 要生成的代理接口
    private final Class<T> mapperInterface;
    // key 是 Mapper接口中方法, value 是 MapperMethod(方法代理)
    private final Map<Method, MapperMethod> methodCache = new ConcurrentHashMap<>();

    pubilc MapperProxyFactory(Class<T> mapperInterface) {
        this.mapperInterface = mapperInterface;
    }
    public Class<T> getMapperInterface(){
        return mapperInterface;
    }

    public Map<Method, MapperMethod> getMethodCache() {
        return methodCache;
    }
    
    /*
    * 这里使用的是 JDK 动态代理
    * Proxy.newInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h);
    * 所以很显然 MapperProxy 是一个基于 SqlSession 实现了 InvocationHandler 接口的类
    * 它会将对应的接口方法调用转接到 SqlSession 上
    */
    protected T newInstance(MapperProxy<T> mapperProxy) {
        return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[]{mapperInterface}, mapperProxy);
    }
    public T newInstance(SqlSession sqlSession) {
        final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
        return newInstance(mapperProxy);
    }
}
```


