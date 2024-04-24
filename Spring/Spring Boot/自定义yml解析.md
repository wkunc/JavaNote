# 如何自定义yml解析流程

需求来源于开发时, 配置里为了方便需要获取全部的类型

1. 方案1, 手动维护, 在 all-type-code 下维护上面所有的类型
2. 方案2, all-type-code, 不作为属性没有对应的字段, 在类上实现一个 getAllTypeCode() 方法, 每次获取其他属性拼接起来
3. 方案3, 查找`SpringBoot`yml支持, 看看是否存在 `${key}` 之类的语法来支持这个需求

```yml
  aca:
    face-type-code: &face
      - "30-57.20.25.10"
    door-type-code: &door
      - "1"
    lock-type-code: &lock
      - "1"
    elevator-type-code: &elevator
      - 30-19.10.00.00
      - 11-11.12.11.00
    all-type-code: !flatten
      - *face
      - *door
      - *lock
      - *elevator
```

经过查找, 发现yml原始语法中就有merge key相关语法, 以及引用标签语法

```yml
  aca:
    face-type-code: &face
      - "30-57.20.25.10"
    door-type-code: &door
      - "1"
    lock-type-code: &lock
      - "1"
    elevator-type-code: &elevator
      - 30-19.10.00.00
      - 11-11.12.11.00
    all-type-code: !flatten
      <<: *face
      <<: *door
      <<: *lock
      <<: *elevator
```

但是merge key语法是设计用于合并两个 map 类型
上述写法 all-type-code 就得是 `List<List<String>>`
然后我们就需要让 yaml 解析时可以将其展开

## 翻阅SpringBoot使用的yaml解析库, 找到扩展点

通过yaml的tag语法, 可以实现自定义的标签解析
所以我们定于

```java
    private class OriginTrackingConstructor extends Constructor {

        public OriginTrackingConstructor() {
            this.yamlConstructors.put(new Tag("!flatten"), new FlattenConstruct());
        }

        @Override
        protected Object constructObject(Node node) {
            if (node instanceof ScalarNode) {
                if (!(node instanceof KeyScalarNode)) {
                    return constructTrackedObject(node, super.constructObject(node));
                }
            }
            else if (node instanceof MappingNode) {
                replaceMappingNodeKeys((MappingNode) node);
            }
            return super.constructObject(node);
        }

        private void replaceMappingNodeKeys(MappingNode node) {
            node.setValue(node.getValue().stream().map(KeyScalarNode::get).collect(Collectors.toList()));
        }

        private Object constructTrackedObject(Node node, Object value) {
            Origin origin = getOrigin(node);
            return OriginTrackedValue.of(getValue(value), origin);
        }

        private Object getValue(Object value) {
            return (value != null) ? value : "";
        }

        private Origin getOrigin(Node node) {
            Mark mark = node.getStartMark();
            TextResourceOrigin.Location location = new TextResourceOrigin.Location(mark.getLine(), mark.getColumn());
            return new TextResourceOrigin(MyYamlProcessor.this.resource, location);
        }

        // 写的比较简单粗暴, 没有做到合理的错误提示, 来应各种情况
        private class FlattenConstruct extends AbstractConstruct {

            @Override
            public Object construct(Node node) {
                SequenceNode node1 = (SequenceNode) node;
                Collection<Object> r = new ArrayList<>();
                node1.getValue().forEach(e -> {
                    constructSequenceStep2((SequenceNode) e, r);
                });
                return r;
            }
        }

    }
```

## 如何替换SpringBoot的yml解析

在`spring-boot.jar` 的`spring.factories`中可以找到.
所以我们只要去实现 PropertySourceLoader 接口就好了,
所以我简单粗暴的就去继承原来的实现类, 然后把自定义标签`flatten`的处理逻辑注册进去.
就大功告成

```properties
org.springframework.boot.env.PropertySourceLoader=org.springframework.boot.env.PropertiesPropertySourceLoader,\
  org.springframework.boot.env.YamlPropertySourceLoader
#替换后的内容
org.springframework.boot.env.PropertySourceLoader=org.springframework.boot.env.PropertiesPropertySourceLoader,\
  cec.saas.aca.config.MyYamlLoader
```
