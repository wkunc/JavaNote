# 反射工具箱

Reflector ReflectorFactory

Reflector 是 MyBatis 反射的基础, 每个Reflector对象都对应一个类,
在Reflector中缓存了反射操作需要使用的类的元信息.
Reflector中的字段含义如下:

```java
    //对应的 Class 类型
    private final Class<?> type;

    //可读属性的名称集合, 就是有getter方法的属性
    private final String[] readablePropertyNames;

    //可写属性的名称集合, 就是有setter方法的属性
    private final String[] writeablePropertyNames;

    //属性对应的 setter 方法, key是属性名, value 是 Invoker 对象(ps:它是 Method 对象的封装)
    private final Map<String, Invoker> setMethods = new HashMap<String, Invoker>();

    //属性对应的 getter 方法, key是属性名, value 是 Invoker 对象
    private final Map<String, Invoker> getMethods = new HashMap<String, Invoker>();

    //属性对应的 setter 方法的参数类型, key属性名, value 是参数类型
    private final Map<String, Class<?>> setTypes = new HashMap<String, Class<?>>();

    //属性对应的 getter 方法的参数类型, key属性名, value 是返回值类型
    private final Map<String, Class<?>> getTypes = new HashMap<String, Class<?>>();

    // 默认构造器
    private Constructor<?> defaultConstructor;

    //所有属性名称的集合
    private Map<String, String> caseInsensitivePropertyMap = new HashMap<String, String>();
```

Reflector 的构造器要求指定解析的 Class 对象, 并填充上述集合

```java
    public Reflector(Class<?> clazz) {
        type = clazz;
        //用反射机制查找默认的无参构造器, 代码比较简单
        addDefaultConstructor(clazz);

        //处理 clazz 中的 getter 方法, 填充 getMethods 集合和 getTypes 集合.
        addGetMethods(clazz);
        //处理 clazz 中的 setter 方法, 填充 setMethods 集合和 setTypes 集合.
        addSetMethods(clazz);
        //处理没有 getter setter 方法的字段
        addFields(clazz);

        // 用上面初始化完毕的 getType,setType 集合进行填充
        readablePropertyNames = getMethods.keySet().toArray(new String[getMethods.keySet().size()]);
        writeablePropertyNames = setMethods.keySet().toArray(new String[setMethods.keySet().size()]);
        for (String propName : readablePropertyNames) {
          caseInsensitivePropertyMap.put(propName.toUpperCase(Locale.ENGLISH), propName);
        }
        for (String propName : writeablePropertyNames) {
          caseInsensitivePropertyMap.put(propName.toUpperCase(Locale.ENGLISH), propName);
        }
    }
```

addGetMethods() 方法主要负责解析类中定义的 getter 方法,
addSetMethods() 方法主要负责解析类中定义的 setter 方法.
两者比较类似, 下面以 addGetMethods() 方法为例.

```java
    private void addGetMethods(Class<?> cls) {
        Map<String, List<Method>> conflictingGetters = new HashMap<String, List<Method>>();
        /*
        *主要步骤1:调用 Reflector.getClassMethods() 方法获取当前类及其父类中定义的
        *所有方法的唯一签名以及 Method 对象
        */
        Method[] methods = getClassMethods(cls);

        /*
        *主要步骤2:遍历上一步返回的 Method 数组,
        *从中选择出 getter 方法,将其记录到 conflictingGetters 集合中
        *conflictingGetters 集合(HashMap<String, List<Method>>类型),
        *key为属性名, value是属性对应的 getter 方法集合
        */
        for (Method method : methods) {
            if (method.getParameterTypes().length > 0) {
                continue;
            }
            //获取方法名
            String name = method.getName();
            //判断是否为 getter 方法
            if ((name.startsWith("get") && name.length() > 3)
              || (name.startsWith("is") && name.length() > 2)) {
                name = PropertyNamer.methodToProperty(name);
                //添加到 HashMap<String, List<Method>>中
                addMethodConflict(conflictingGetters, name, method);
            }
        }

        /*
        *主要步骤3: 当子类覆盖了父类的 getter 方法且返回值发生变化时
        *(Java中子类覆盖父类方法时返回值可以是父类返回值的子类),
        *步骤1中就会产生两个签名不同的方法. 主要是返回值不同
        *步骤3中调用 Reflector.resolveGetterConflicts()方法对这种情况进行处理(ps:Conflict 矛盾)
        *
        */
        resolveGetterConflicts(conflictingGetters);
    }
```

主要步骤一:

```java
    private Method[] getClassMethods(Class<?> cls) {
        Map<String, Method> uniqueMethods = new HashMap<String, Method>();
        Class<?> currentClass = cls;
        // 这里使用了一个循环, 每次循环都会把 currentClass 变量变成之前的父类Class对象
        // 这样就可以将所有方法都遍历到了
        while (currentClass != null && currentClass != Object.class) {
            //在 Refector.adUniqueMethods() 方法中会为每个方法生成唯一签名并记录到集合中
            addUniqueMethods(uniqueMethods, currentClass.getDeclaredMethods());

            // we also need to look for interface methods -
            // because the class may be abstract
            Class<?>[] interfaces = currentClass.getInterfaces();
            for (Class<?> anInterface : interfaces) {
                addUniqueMethods(uniqueMethods, anInterface.getMethods());
            }

            currentClass = currentClass.getSuperclass();
        }

        Collection<Method> methods = uniqueMethods.values();

        return methods.toArray(new Method[methods.size()]);
    }

    //
    private void addUniqueMethods(Map<String, Method> uniqueMethods, Method[] methods) {
        for (Method currentMethod : methods) {
            /*
            *判断方法是否为桥接方法(桥接方法和Java泛型实现有关)
            *不是桥接方法才是我们要的方法
            */
            if (!currentMethod.isBridge()) {
                /*
                *生成唯一签名,格式是: 返回值类型#方法名:参数类型列表
                *比如 Reflector.getSignature(Method)方法的签名是:
                *java.lang.String#getSignature:java.lang.reflect.Method
                *生成的方法签名是唯一的
                */
                String signature = getSignature(currentMethod);

                // check to see if the method is already known
                // if it is known, then an extended class must have
                // overridden a method
                if (!uniqueMethods.containsKey(signature)) {
                    if (canAccessPrivateMethods()) {
                        try {
                          currentMethod.setAccessible(true);
                        } catch (Exception e) {
                          // Ignored. This is only a final precaution, nothing we can do.
                        }
                    }

                    uniqueMethods.put(signature, currentMethod);
                }
            }
        }
    }
```

步骤3:

```java
  private void resolveGetterConflicts(Map<String, List<Method>> conflictingGetters) {
    for (Entry<String, List<Method>> entry : conflictingGetters.entrySet()) {
      Method winner = null;
      String propName = entry.getKey();
      for (Method candidate : entry.getValue()) {
        if (winner == null) {
          winner = candidate;
          continue;
        }
        Class<?> winnerType = winner.getReturnType();
        Class<?> candidateType = candidate.getReturnType();
        if (candidateType.equals(winnerType)) {
          if (!boolean.class.equals(candidateType)) {
            throw new ReflectionException(
                "Illegal overloaded getter method with ambiguous type for property "
                    + propName + " in class " + winner.getDeclaringClass()
                    + ". This breaks the JavaBeans specification and can cause unpredictable results.");
          } else if (candidate.getName().startsWith("is")) {
            winner = candidate;
          }
        } else if (candidateType.isAssignableFrom(winnerType)) {
          // OK getter type is descendant
        } else if (winnerType.isAssignableFrom(candidateType)) {
          winner = candidate;
        } else {
          throw new ReflectionException(
              "Illegal overloaded getter method with ambiguous type for property "
                  + propName + " in class " + winner.getDeclaringClass()
                  + ". This breaks the JavaBeans specification and can cause unpredictable results.");
        }
      }
      //
      addGetMethod(propName, winner);
    }
  }

  /*
  *
  */
  private void addGetMethod(String name, Method method) {
    if (isValidPropertyName(name)) {
      getMethods.put(name, new MethodInvoker(method));
      // 这里用到了 TypeParameterResolver
      Type returnType = TypeParameterResolver.resolveReturnType(method, type);
      getTypes.put(name, typeToClass(returnType));
    }
  }
```

接下来看看 addFields() 方法的执行过程

```java
    private void addFields(Class<?> clazz) {
        // 获得 class 声明的所有字段
        Field[] fields = clazz.getDeclaredFields();
        for(Field field : fields) {
            if (canAccessPrivateMethods()) {
                try {
                    field.setAccessible(true);
                } catch (Exception e) { }
            }
            // 这个判断是不是一定为 true, 除非上面抛异常. 我现在对 Java 的安全模型还不够了解
            if (field.isAccessible()) {
                // 如果之前初始化完成的 setMethods 没有这个属性, 说明这个字段没有 setter 方法
                if (!setMethods.containsKey(field.getName())) {
                    int modifiers = field.getModifiers();
                    // 这里获得当前 field 的修饰符, 排除 static 和 final 的常量
                    if (!(Modifier.isFinal(modifiers) && Modifier.isStatic(modifiers))) {
                        addSetField(field);
                    }
                }
                if (!getMethods.containsKey(field.getName())) {
                    addGetField(field);
                }
            }
        }
        if (clazz.getSuperclass() !=null) {
            addFields(clazz.getSuperclass());
        }
    }
```

ReflectorFactory 接口主要实现了对 Reflector 对象的创建和缓存.
Mybatis 只提供了一个DefaultReflectorFactory实现类, 这个类相当简单就不仔细分析了
我们也可以自己实现一个然后再配置文件中声明, 从而实现功能扩展

```java
    public interface ReflectorFactory {
        boolean isClassCacheEnabled();
        void setClassCacheEnabled(boolean classCacheEnabled);
        Reflector findForClass(Class<?> type);
    }
```

这个默认的 ReflectiorFactory 实现非常的简单,
MyBatis也允许我们自己实现这个类从而实现特殊功能

```java
public class DefaultReflectorFactory implements ReflectorFactory {
    private boolean classCacheEnabled = true;
    private final ConcurrentMap<Class<?>, Reflector> reflectorMap = new ConcurrentHashMap<Class<?>, Reflector>();

    public DefaultReflectorFactory() {
    }

    @Override
    public boolean isClassCacheEnabled() {
        return classCacheEnabled;
    }

    @Override
    public void setClassCacheEnabled(boolean classCacheEnabled) {
        this.classCacheEnabled = classCacheEnabled;
    }

    @Override
    public Reflector findForClass(Class<?> type) {
        if (classCacheEnabled) {
            // synchronized (type) removed see issue #461
            Reflector cached = reflectorMap.get(type);
            if (cached == null) {
                cached = new Reflector(type);
                reflectorMap.put(type, cached);
            }
            return cached;
        } else {
            return new Reflector(type);
        }
    }
}
```
