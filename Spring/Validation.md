# Validation
将验证视为业务逻辑有利有弊, 而Spring提供了一种不排除其中任何一种的验证(和数据绑定)设计. 
具体来说, 验证不应该与Web层绑定, 并且应该易于本地化,
并且应该可以插入任何可用的验证器. 考虑到这些问题, Spring提出了一个Validator接口

# Validation by Using Spring's Validator Interface
```java
public interface Validator {
    boolean supports(Class<?> clazz);
    void validate(Object target, Errors errors);
}
```

# Resolving Codes to Error Messages


# Bean Manipulation and the BeanWrapper
```java
public interface BeanWrapper extends ConfigurablePropertyAccessor {
    void setAutoGrowCollectionLimit(int autoGrowCollectionLimit);
    int getAutoGrowCollectionLimit();
    Object getWrappedInstance();
    Class<?> getWrappedClass();
    PropertyDescriptor[] getPropertyDescriptiors();
    PropertyDescriptor getPropertyDescriptor(String propertyName);
}
