TestClass, FrameworkMethod, FrameworkField.

# TestClass
TestClass 是我们测试类的包装, 持有那个类中的Field和Method的包装类对象.
```java
public TestClass(Class<?> clazz) {
    this.clazz = clazz;
    if (clazz != null && clazz.getConstructors().length > ) {
        thorw new IllegalArgumentException("Test class 
                can only have one constructor");
    }

    // key 为 注解的class对象, value 是该注解标记的方法集合
    Map<Class<? extends Annotation>, List<FrameworkMethod>> methodsForAnnotations = new LinkedHashMap<>();
    // key 为 注解的class对象, value 是该注解标记的方法集合
    Map<Class<? extends Annotation>, List<FrameworkMethod>> methodsForAnnotations = new LinkedHashMap<>();

    scanAnnotatedMembers(methodsForAnnotations, fieldsForAnnotations);

    this.methodsForAnnotations = makDeeplyUnmodifiable(methodsForAnnotations);
    this.fieldsForAnnotations = makeDeeplyUnmodifiable(fieldsForAnnotations);
}

protected void scanAnnotatedMembers(Map<Class<? extends Annotation>, List<FrameworkMethod>> methodsForAnnotations, Map<Class<? extends Annotation>, List<FrameworkField>> fieldsForAnnotations) {
    // 传入就是两个空的linkedhashmap
    // 遍历测试Test.class, 目的是为了完整获得其中申明的method和field包括父类中
    for (Class<?> eachClass : getSuperClasses(clazz)) {
        for (Method eachMethod : MethodSorter.getDeclaredMethods(eachClass)) {
            addToAnnotationLists(new FrameworkMethod(eachMethod), methodsForAnnotations);
        }
        // ensuring fields are sorted to make sure that entries are inserted
        // and read from fieldForAnnotations in a deterministic order
        for (Field eachField : getSortedDeclaredFields(eachClass)) {
            addToAnnotationLists(new FrameworkField(eachField), fieldsForAnnotations);
        }
    }
}

protected static <T extends FrameworkMember<T>> void addToAnnotationLists(T member, 
        Map<Class<? extends Annotation>, List<T>> map) {
    // 
    for (Annotation each : member.getAnnotations()) {
        Class<? extends Annotation> type = each.annotationType();
        List<T> members = getAnnotatedMembers(map, type, true);
        // 
        if (member.isShadowedBy(members)) {
            return;
        }
        if (runsTopToBottom(type)) {
            members.add(0, member);
        }else {
            members.add(member);
        }
    }
}
private static <T> List<T> getAnnotatedMembers(Map<Class<? extends Annotation>, List<T>> map,
        boolean fillIfAbsent) {
    // 如果map中没有这个key, 当然也不会有对应的value, fillIfAbsent来控制是否创建list
    if (!map.containsKey(type) && fillIfAbsent) {
        map.put(type, new ArrayList<T>();
    }
    List<T> members = map.get(type);
    return members == null ? Collections.<T>emptyList() : members;
}
```

TestClass会在什么时候创建呢?
ParentRunner的构造器中调用createTestClass()方法创建了TestClass对象.
并且ParentRunner只有一个构造器所以其子类也必须执行这段逻辑,
即在初始化的时候创建TestClass对象.
```java
protected ParentRunner(Class<?> testClass) throws InitializationError {
    this.testClass = createTestClass(testClass);
    validate();
}

protected TestClass createTestClass(Class<?> testClass) {
    return new TestClass(testClass);
}
```
