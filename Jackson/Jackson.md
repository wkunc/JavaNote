# Creating JSON from JAVA

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

对象中的数组或者List<?>都会被转化成 json [] 数组
对象中的Map或者其他的对象类型都会被转换成 json {} 对象

## 处理空值

当Jackson发现需要序列化的对象中包含为Null的引用比如定义但是没有初始化的数组
private String[] home; 它会将其序列化为 null
如果它发现一个为Null的引用,但是它是一个空的数组或集合,
private List\<String\> list = new ArrayList<>();
Jackson会将它序列化为 空数组对象 [],

可以设置mapper属性来忽略空值
这样就可以让Jackson自动忽略**null引用**和**空数组(集合)**
``java
mapper.setSerializationInclusion(JsonInclude.Include.NON_EMPTY);

```

# Tree Model
1. Create a JsonNodeFactory to create the nodes
2. Create a JsonGenerator from a JsonFactory and specify the output method. In this method we print to console.
3. Create an ObjectMapper that will use the jsonGenerator and the root node to create the JSON.

默认情况下 ObjectMapper 不会命名根节点

# Stream API
Parse JSON to Java using Streaming Jackson Parser


# Annotation and Serialization

* List Serialization :如何序列化时带上类型信息, 这样反序列化 List 的时候就可以根据类型信息创建对应的 Java 对象
* Annotation and Dynamic beans :将json中的多于的信息放到一个 Map 中
* Annotation Filters :自定义想要序列化哪些字段(即哪些字段应该被 ignore 不应该写入生成的 json)
* Mix-in :
* Polymorphic Behaviour

## Annotation

1. @JsonProperty
2. @JsonCreator
3. @JsonAnyGetter and @JsonAnySetter

1. @JsonIgnoreProperties
2. @JsonIgnore
3. Custom filter(自定义过滤器)
