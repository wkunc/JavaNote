# Loading YAML
Spring 提供了两个用来解析YAML文档的方便类.
* YamlPropertiesFactoryBean (ps: 将YAML加载为 Properties)
* YamlMapFactoryBean        (ps: 将YAML加载为 Map)

YamlPropertySourceLoader 类可以用于Spring环境中将YAML公开为PropertySource
这样做可以用 @Value 来访问YAML属性(PS: 因为@PropertySource注解无法加载YAML文件所以才需要这个类)
