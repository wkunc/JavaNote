# SingletionBeanRegistry
为共享 Bean 实例定义注册表接口
```java
void registerSingletion(String beanName, Object singletonObject);
Object getSingletion(String beanName);
boolean containsSingletion(String beanName);
String[] getSingletionNames();
int getSingletOnCount();
Object getSingletionMutex();
```

# AliasRegistry
SimpleAliasRegistry 是它的简单实现

```java
void registerAlias(String name, String alias);
void removeAlias(String alias);
boolean isAlias(String name);
String[] getAliases(String name);
```

# BeanDefinitionRegistry
```java
void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
void removeBeanDefinition(String beanName);
BeanDefinition getBeanDefinition(String beanName);
boolean containsBeanDefinition(String beanName);
String[] getBeanDefinitionNames();
int getBeanDefinitionCount();
boolean isBeanNameInUse(String beanName);
```
