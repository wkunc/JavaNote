ServiceRegistry
StandardServiceRegistry
Service

```java
public StandardServiceRegistryBuilder() {
    this(new BootstrapServiceRegistryBuilder().enable().builde());
}
public StandardServiceRegistryBuilder(BootstrapServiceRegistry bootstrapServiceRegistry) {
    this(bootstrapServiceRegistry, LoadedConfig.baseLine());
}
public StandardServiceRegistryBuilder(
        BootstrapServiceRegistry bootstrapServiceRegistry,
        LoadedConfig loadedConfigBaseline) {
    this.settings = Environment.getProperties();
    this.bootstrapServiceRegistry = bootstrapServiceRegistry;
    this.configLoader = new ConfigLoader(bootstrapServiceRegistry);
    this.aggregatedCfgXml = loadedConfigBaseline;
}
```

