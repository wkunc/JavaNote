# TypeParametterResolver
## Type 接口
这是Java反射的知识, 这里简单介绍一下.

* Class 比较常见, 它表示的是原始类型. Class 类的对象表示 JVM 中的一个类或接口,
  每个Java类在JVM里都表现为一个Class对象. 数组也被映射为 Class 对象, 
  所有元素类型相同且维数相同的数组共享一个 Class 对象.
* ParameterizedType 表示的参数化类型, 例如 List<String>, Map<Integer, String>这种带泛型的类型

* TypeVariable 表示的是类型变量, 它用来反映在 JVM 编译该泛型前的信息

* GenericArrayType 表示的是数组类型且组成元素是 ParameterizedType 或 TypeVariable.

* WildcardType 表示的是通配符泛型


---
TypeParameterResolver 是一个工具类, 提供了一系列静态方法来解析指定类中的字段, 方法返回值或方法参数的类型

# ObjectFactory
MyBatis 中会有很多模块会使用 ObjectFacotry 接口, 该接口提供了多个 create() 方法的重载,
通过这些 create() 方法可以创建指定类型的对象.
```java
public interface ObjectFactory {
    /**
    * Sets configuration properties.
    */
    void setProperties(Properties properties);

    /**
    * Creates a new object with default constructor. 
    * 使用默认构造器创建对象
    * class.newInstance();
    */
    <T> T create(Class<T> type);

    /**
    * Creates a new object with the specified constructor and params.
    * 使用指定构造器和参数构造对象
    */
    <T> T create(Class<T> type, List<Class<?>> constructorArgTypes, List<Object> constructorArgs);

    /**
    * Returns true if this object can have a set of other objects.
    * It's main purpose is to support non-java.util.Collection objects like Scala collections.
    */
    <T> boolean isCollection(Class<T> type);
}
```

# Property 工具集
PropertyTokenizer, PropertyNamer, PropertyCopier

```java
private String name; // 当前表达式的名称
private final String indexedName; // 当前表达式的索引名
private String index; // 索引下标
private final String children; // 子表达式
```

PropertyNamer 是另一个工具类, 提供了完成方法名到属性名的转换的静态方法, 以及多种检测操作.

# MetaClass

# ObjectWrapper
ObjectWrapper 接口是对 对象 的包装, 抽象了对象的属性信息,
它定义了一系列查询对象属性信息的方法, 以及更新属性的方法.

```java
    public interface ObjectWrapper {
        /*
        * 如果ObjectWrapper 中封装的普通的 Bean 对象, 则调用相应属性的 getter 方法,
        * 如果封装的是集合类, 则获取指定 key 或下标对应的 value 值
        */
        Object get(PropertyTokenizer prop);

        /*
        *±
        */
        void set(PropertyTokenizer prop, Object value);

        /*
        *
        */
        String findProperty(String name, boolean useCamelCaseMapping);

        /*
        *
        */
        String[] getGetterNames();

        /*
        *
        */
        String[] getSetterNames();

        /*
        *
        */
        Class<?> getSetterType(String name);

        Class<?> getGetterType(String name);

        boolean hasSetter(String name);

        boolean hasGetter(String name);

        MetaObject instantiatePropertyValue(String name, PropertyTokenizer prop, ObjectFactory objectFactory);

        boolean isCollection();

        void add(Object element);

        <E> void addAll(List<E> element);
    }
```
