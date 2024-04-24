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

# 解析Mapper节点.

```java
// XNode = <mappers>
  private void mapperElement(XNode parent) throws Exception {
    if (parent != null) {
      for (XNode child : parent.getChildren()) {
        if ("package".equals(child.getName())) {
          String mapperPackage = child.getStringAttribute("name");
          configuration.addMappers(mapperPackage);
        } else {
          String resource = child.getStringAttribute("resource");
          String url = child.getStringAttribute("url");
          String mapperClass = child.getStringAttribute("class");
          if (resource != null && url == null && mapperClass == null) {
            ErrorContext.instance().resource(resource);
            InputStream inputStream = Resources.getResourceAsStream(resource);
            // 对于每个 <mappers> 子标签,
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
            mapperParser.parse();
          } else if (resource == null && url != null && mapperClass == null) {
            ErrorContext.instance().resource(url);
            InputStream inputStream = Resources.getUrlAsStream(url);
            //
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments());
            mapperParser.parse();
          } else if (resource == null && url == null && mapperClass != null) {
            Class<?> mapperInterface = Resources.classForName(mapperClass);
            configuration.addMapper(mapperInterface);
          } else {
            throw new BuilderException("A mapper element may only specify a url, resource or class, but not more than one.");
          }
        }
      }
    }
  }
```

# XMLMapperBuilder

```java
  public void parse() {
    // 避免重复加载.
    if (!configuration.isResourceLoaded(resource)) {
      // 1.
      configurationElement(parser.evalNode("/mapper"));
      configuration.addLoadedResource(resource);
      bindMapperForNamespace();
    }

    parsePendingResultMaps();
    parsePendingCacheRefs();
    parsePendingStatements();
  }

  // 1. 解析 mapper 的子标签, 就是 cache-ref, cache, parameterMap, resultMap, sql, 以及剩下的select, insert等
  private void configurationElement(XNode context) {
    try {
      String namespace = context.getStringAttribute("namespace");
      if (namespace == null || namespace.equals("")) {
        throw new BuilderException("Mapper's namespace cannot be empty");
      }
      builderAssistant.setCurrentNamespace(namespace);
      cacheRefElement(context.evalNode("cache-ref"));
      cacheElement(context.evalNode("cache"));
      // 解析所有的 parameterMap
      parameterMapElement(context.evalNodes("/mapper/parameterMap"));
      // 解析所有的 resultMap
      resultMapElements(context.evalNodes("/mapper/resultMap"));
      sqlElement(context.evalNodes("/mapper/sql"));
      buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing Mapper XML. The XML location is '" + resource + "'. Cause: " + e, e);
    }
  }

  // 1. 解析所有的 parameterMap.
  private void parameterMapElement(List<XNode> list) {
    for (XNode parameterMapNode : list) {
      //
      String id = parameterMapNode.getStringAttribute("id");
      String type = parameterMapNode.getStringAttribute("type");
      Class<?> parameterClass = resolveClass(type);
      List<XNode> parameterNodes = parameterMapNode.evalNodes("parameter");
      List<ParameterMapping> parameterMappings = new ArrayList<>();
      for (XNode parameterNode : parameterNodes) {
        String property = parameterNode.getStringAttribute("property");
        String javaType = parameterNode.getStringAttribute("javaType");
        String jdbcType = parameterNode.getStringAttribute("jdbcType");
        String resultMap = parameterNode.getStringAttribute("resultMap");
        String mode = parameterNode.getStringAttribute("mode");
        String typeHandler = parameterNode.getStringAttribute("typeHandler");
        Integer numericScale = parameterNode.getIntAttribute("numericScale");
        ParameterMode modeEnum = resolveParameterMode(mode);
        Class<?> javaTypeClass = resolveClass(javaType);
        JdbcType jdbcTypeEnum = resolveJdbcType(jdbcType);
        Class<? extends TypeHandler<?>> typeHandlerClass = resolveClass(typeHandler);
        ParameterMapping parameterMapping = builderAssistant.buildParameterMapping(parameterClass, property, javaTypeClass, jdbcTypeEnum, resultMap, modeEnum, typeHandlerClass, numericScale);
        parameterMappings.add(parameterMapping);
      }
      builderAssistant.addParameterMap(id, parameterClass, parameterMappings);
    }
  }
```
