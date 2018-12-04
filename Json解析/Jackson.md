#Creating JSON from JAVA
There are three ways to create JSON from JAVA:

1. From a Java Object (The Same object can also be then used to read the JSON)
2. From a JsonNode Tree
3. Building a JSON Stream

Jackson提供了可用于将Java对象转换为JSON并返回的类.
在这个例子中, 我们看看如何将Java转换为JSON,将JSON转换为Java.
我们将从一个简单的类开始, 逐渐开始为它添加复杂性.
假设我们是一家音乐公司, 我们希望发布一个用户可以查询相册的API.
我们首先使用单个属性标题构建一个Album类.

```java
class Album {
    private String title;
    public Album(String title) {
        this.title = title;
    }
    public String getTitle(){
        return title;
    }
}
```
## ObjecctMapper
默认情况下, Jackson会使用 BeanSerializer 来序列化POJO.
> 注意: Bean 的私有属性应该具有 getter, 或者属性应该是 public 的

# Java 转 Json
默认情况下, jackson 使用 java 字段名作为 json 属性名.
也可以使用 jackson 提供的注解来改变其默认行为.
如果无法直接获得POJO源码或者不希望Bean和 Jackson 注解绑定,
jackson 提供了另一种方法. 使用 mapper 上的 setPropertyNamingStrategy() 改变命名策略


## 处理Null


# Tree Model
1. Create a JsonNodeFactory to create the nodes
2. Create a JsonGenerator from a JsonFactory and specify the output method. In this method we print to console.
3. Create an ObjectMapper that will use the jsonGenerator and the root node to create the JSON.

默认情况下 ObjectMapper 不会命名根节点

# Stream API
Parse JSON to Java using Streaming Jackson Parser


# Annotation and Serialization

* List Serialization
* Annotation and Dynamic beans
* Annotation Filters
* Mix-in
* Polymorphic Behaviour

## Annotation

1. @JsonProperty
2. @JsonCreator
3. @JsonAnyGetter and @JsonAnySetter
