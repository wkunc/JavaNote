```java
SqlSessionFactoryBuilder builder = new SqlSessionFactoryBuilder();
factory = builder.build(new FileInputStream("...."));
```
很显然, MyBatis的初始化入口是 SqlSessionFactoryBuilder.build() 方法.
具体实现如下:
```java
public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
    try {
        XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
        return build(parser.parse());
    } catch (Exception e) {
        throw ExceptionFactory.wrapException("Error building SqlSession.", e);
    } finally {
        ErrorContext.instance().reset();
        try {
            inputStream.close();
        } catch(IOException e)
                                                
        }
    }
}
```
# BaseBuilder
```java
public abstract class BaseBuilder {
    protected final Configuration configuration;
    protected final TypeAliasRegistry typeAliasRegistry;
    protected final TypeHandlerRegistry typeHandlerRegistry;

    public BaseBuilder(Configuration configuration) {
        this.configuration = configuration;
        this.typeAliasRegistry = this.configuration.getTypeAliasRegisStry();
        this.typeHandlerRegistry = this.configuration.getTypeHandlerRegistry();
    }

}

```

# XMLConfigBuilder
XMLConfigBuilder 是 BaseBuilder 的众多子类之一, 它主要负责解析 mybatis-config.xml
核心字段如下:
```java
// 是否解析过的标志
private boolean parsed;
// 用于解析 mybatis-config.xml 的解析器
private final XPathParser parser;
// 标识<environment>配置名称, 默认读取<environment>标签的default属性
private String environment;
// 负责创建和缓存 Reflector 对象的工厂
private final ReflectoryFactory localReflectorFactory = new DefaultReflectoryFactory();
```

XMLConfigBuilder.parser() 方法是解析 mybatis-config.xml 配置文件的入口
```java
public Configuration parse() {
    if (parsed) {
        throw new BuilderException("Each XMLConfigBuilder can only be used once.");
    }
    parsed = true;
    parseConfiguration(parser.evalNode("/configuration"));
    return configuration;
}
private void parseConfiguration(XNode root) {
    try {
        // 解析<properties>节点
        propertiesElement(root.evalNode("properties"));
        // 解析<settings> 节点
        Properties settings = settingsAsProperties(root.evalNode("settings"));
        // 设置 vfsImpl 字段
        loadCustomVfs(settings);
        // 解析<plugins>节点
        typeAliasesElement(root.evalNode("typeAliases"));
        // 解析<objectFactory>节点
        pluginElement(root.evalNode("plugins"));
        // 解析<objectWrapperFactory节点
        objectFactoryElement(root.evalNode("objcetFactory"));
        // 解析<reflectorFactory>节点
        reflectorFactoryElement(root.evalNode("reflectorFactory"));
        // 将 settings 值设置到 Configuration 中
        settingsElement(settings);
        // 解析<environment>节点
        environmentsElement(root.evalNode("environments"));
        // 解析<databaseIdProvider>节点
        databaseIdProviderElement(root.evalNode("databaseIdProvider"));
        // 解析<typehandlers>节点
        typeHandlerElement(root.evalNode("typeHandlers"));
        // 解析<mappers>节点
        typeHandlerElement(root.evalNode("mappers"));
    } catch (Exception e) {
        throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
}
```

## 解析properties节点
```java
private void propertiesElement(XNode context) throws Exception {
    if (context != null) {
        Properties defaults = context.getChildrenAsProperties();
        String resource = context.getStringAttribut("resource");
        String url = context.getStringAttribut("url");
        if (resource != null && url != null) {
            throw new BuilderException("...");
        }
        if (resource != null) {
            defaults.putAll(Resources.getResourceAsProperties(resource));
        } else if (url != null) {
            defaults.putAll(Resources.getUrlAsRpoperties(url));
        }
        Properties vars = configuration.getVariables();
        if (vars != null) {
            defaults.putAll(vars);
        }
        parser.setVariables(defaults);
        configuration.setVariables(defaults);
    }
}
```