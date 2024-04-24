# BeanNameGenerator

一个用于给指定Bean生成Name的策略接口

```java
public interface BeanNameGenerator {
    String generateBeanName(BeanDefinition definition, BeanDefinitionRegistry registry);
}
```

这个接口有两个实现

```java
// 委托给 BeanDefinitionReaderUtils.generateBeanName() 方法实现
/**
* 总结生成规则:1
* 1. 如果是普通的 BeanDefinition. 那么就是对应的 java class 的全类名
* 2. 如果是 FactoryBean 创建的Bean, 那么就是对应的 FactoryBean 在IOC中的名字后面拼接 $created
* 3. 如果是通过继承<bean extend = ""/>声明的, 那么也拿不到直接的class, 就是获取父定义的class 后面拼接 $chlid
* 4. 最后, 如果同一个 class 在IOC中注册了多个, 那么按照顺序在后面拼接 #count, 计数从0开始
*/
public class DefaultBeanNameGenerator implements BeanNameGenerator {
    public String generateBeanName(BeanDefinition definition, BeanDefinitionRegistry registry){
        return BeanDefinitionReaderUtils.generateBeanName(definition, registry);
    }
}

public static String generateBeanName(BeanDefinition beanDefinition, BeanDefinitionRegistry registry)
        throws BeanDefinitionStoreException {
    return generateBeanName(beanDefinition, registry, false);
}

public static String generateBeanName(
        BeanDefinition definition, BeanDefinitionRegistry registry, boolean isInnerBean)
        throws BeanDefinitionStoreException {

    // 获取声明的类型名, 当这个<bean/>声明是通过继承的, 或者FactoryBean方式就不能直接获取到
    String generatedBeanName = definition.getBeanClassName();
    if (generatedBeanName == null) {
        // 获取继承的父亲的名字,如果有就在后面拼接 $child
        if (definition.getParentName() != null) {
            generatedBeanName = definition.getParentName() + "$child";
        }
        else if (definition.getFactoryBeanName() != null) {
            // 获取对应的 FactoryBean 的名字, 在后面拼接 $created
            generatedBeanName = definition.getFactoryBeanName() + "$created";
        }
    }
    // 如果上面的方式都拿不到就抛异常, 毕竟是定义出错了
    if (!StringUtils.hasText(generatedBeanName)) {
        throw new BeanDefinitionStoreException("Unnamed bean definition specifies neither " +
                "'class' nor 'parent' nor 'factory-bean' - can't generate bean name");
    }

    String id = generatedBeanName;
    // 如果是 InnerBean 的话, 最后的Name 会拼接 "#" + hash串
    if (isInnerBean) {
        // Inner bean: generate identity hashcode suffix.
        id = generatedBeanName + GENERATED_BEAN_NAME_SEPARATOR + ObjectUtils.getIdentityHexString(definition);
    }
    else {
        // Top-level bean: use plain class name.
        // Increase counter until the id is unique.
        int counter = -1;
        // 判断registry中是否已经有这个名字有的话count++, 最后的名字会拼上 #2 之类的
        while (counter == -1 || registry.containsBeanDefinition(id)) {
            counter++;
            id = generatedBeanName + GENERATED_BEAN_NAME_SEPARATOR + counter;
        }
    }
    return id;
}
```

这个就是为了的基于注解配置自动beanName生成器.

```java
public class AnnotationBeanNameGenerator implements BeanNameGenerator {
    private static final String COMPONENT_ANNOTATION_CLASSNAME = "org.springframework.stereotype.Component";

    @Override
    public String generateBeanName(BeanDefinition definition, BeanDefinitionRegistry registry){
        // 要求 bean definition 必须是 AnnoteatedBeanDefinition
        if (definition instanceof AnnotatedBeanDefinition) {
            String beanName = determineBEanNameFromAnnotation((AnnotatedBeanDefinition) definition);
            if (StringUtils.hasText(beanName)) {
                return beanName;
            }
        }
        return buildDefaultBeanName(definition, registry);
    }

    protected String determineBeanNameFromAnnotation(AnnotatedBeanDefinition annotatedDef) {
        AnnotationMetadata amd = annotatedDef.getMetadata();
        // 获取注解类型
        Set<String> type = amd.getAnnotationTypes();
        String beanName = null;
        for (String type : types) {
            // 获取每个注解的属性值集合
            AnnotationAttributes attributes = AnnotationConfigUtils.attributesFor(amd, type);
            // 利用方法方法判断这个注解是不是 有Name属性的 stereotype
            if (isStereotypeWithNameValue(type, amd.getMetaAnnotationTypes(type), attributes) {
                Object valule = attributes.get("value");
                if (value instanceof String) {
                    String strVal = (String) value;
                    if (StringUtils.hasLength(strVal)) {
                        if (beanName != null && !strVal.equals(beanName)) {
                            throw new IllegalStateException("Stereotype annotations suggest inconsistent " +
                                    "component names: '" + beanName + "' vsersus '" + strVal + "'");
                        }
                        beanName = strVal;
                    }
                }
            }
        }
    }

    // 判断注解类型
    protected boolean isStereotypeWithNameValue(String annotationType,
            Set<String> metaAnnotationTypes, Map<String, Object> attributes) {
        boolean isStereotype = annotationType.equals(COMPONENT_ANNOTATION_CLASSNAME) ||
                (metaAnnotationTypes != null && metaAnnotationTypes.contains(COMPONENT_ANNOTATION_CLASSNAME)) ||
                annotationType.equals("javax.annotation.ManagedBean") ||
                annotationType.equals("java.inject.Named");

        return (isStereotype && attributes != nulli && attributes.containsKey("value"));
    }

	protected String buildDefaultBeanName(BeanDefinition definition, BeanDefinitionRegistry registry) {
		return buildDefaultBeanName(definition);
	}

	protected String buildDefaultBeanName(BeanDefinition definition) {
		String shortClassName = ClassUtils.getShortName(definition.getBeanClassName());
		return Introspector.decapitalize(shortClassName);
	}
}
```
