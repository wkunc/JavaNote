# databind
jackson-databind 包依赖于另外两个包jackson-core 和 jackson-annotations.

在Java对象和Json内容之间绑定(也称为映射)数据.

也就是说给定Json内容的某种形式(比如说 Stream-of-Event)
库可以构造等效的Java对象. 
相反它可以在给定Java对象的情况下编写Json内容.

如果没有针对低级读写操作的要求, 请选择这种方式.
这通常时Java程序员最自然的方法, 因为它是以对象为中心的.

# Reading Object from Json
相比较低级的Stream-of-event API, 对象绑定代码更加的简单.

下面是几个example:
```java
ObjectMapper mapper = new ObjectMapper();
TwitterEntry entry = mapper.readValue(new File("..."));

Boolean yesOrNo = mapper.readValue("true"); // return Boolean.TRUE
int[] ids = mapper.readValue("1, 3, 98"); // new int[] {1, 3, 98}
// 由于泛型擦除, 所以无奈的使用了一个特殊的手段, 通过TypeReference 子类化来传递泛型信息. 所以这个 TypeReference 中没有任何abstract方法, 它就是这么用的.
Map<String, List<String>> dict = mapper.readValue("{ \"word\" : [ \"synonym1\", \"synonym2\" ] }", new TypeReference<Man<String, List<String>>>() {});

// 会生成一个 List<Object> 来转换这个 Json 数组
Object misc = mapper.readVlaue("[1, true, null]", Object.class);

// 将一个Json对象映射到Map上, 因为Json中没有类型信息, 所以ObjectMapper尽力将Json内容映射到特定的Java类型.
// 并且通常, 对象可以被视为中 "静态Map". 因此将Json对象映射到 Map 中完全没有问题.
Map<String, Object> entryAsMap = mapper.readValue(new File("input.json"), new TypeReference<Map<String, Object>>(){});
// Jackson 对指定Object.class 做了一些特殊处理, 它表示 mapper 将使用于所解析Json内容最匹配的对象, 对于 Json对象意味着 Map, 对于 Json数组意味着 List.
// 这种情况下它的工作方式类似于指定接过为 Map 类型
Object entryAsMap = (Map<String,Object>)mapper.readValue(new File("input.json"), Object.class);
```

# Writing Objects as Json


