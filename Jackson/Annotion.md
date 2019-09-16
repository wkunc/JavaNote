# Annotation
1. @JsonProperty
2. @JsonCreator
3. @JsonAnyGetter and @JsonAnySetter

1. @JsonIgnoreProperties
2. @JsonIgnore
3. Custom filter(自定义过滤器)

## Serizlization Annotations

### @JsonAnyGetter

@JsonAnyGetter注释允许使用Map字段作为标准属性的灵活性.

这是一个快速示例:
ExtendableBean实体具有name属性和一组以键/值对形式的可扩展属性:
```java
public class ExtendableBean {
    public String name;
    private Map<String, String> properties;
 
    @JsonAnyGetter
    public Map<String, String> getProperties() {
        return properties;
    }
}
```
在没有@JsonAnyGetter的时候,
Jackson 将这个 properties 视为根对象的一个字段字段,
字段类型是Object. 结果如下: 嵌套的json字符串.
```json
{
  "name" : "wkunc",
  "properties" : {
    "school" : "china",
    "age" : "10",
    "controtry" : "china"
  }
}
```
在标记上这个注解后, Jackson 将Map中的值视为根对象的属性. 
忽略了这个map的存在, 它不会表现出来. 
即使这个map是null或是empty也不会有影响.
```json
{
  "name" : "wkunc",
  "school" : "china",
  "age" : "10",
  "controtry" : "china"
}
# 对象map为null或empty的情况.
{
  "name" : "wkunc",
}
```
这个注解有一个boolean类型 enabled 属性, 可以指示注解是否生效.

最后附上这个注解的官方文档
> 标记注释, 可用于将非静态, 无参数方法定义为"any getter".
> 获取一组键/值对的访问器,
> 作为包含POJO(类似于展开)的一部分和它具有的常规属性值一起被序列化, 
> 这通常用作" any setter”mutators的对应物(参见JsonAnySetter).
> *请注意, 带注释的方法的返回类型必须是java.util.Map.*
> **与JsonAnySetter一样, 应该只有一个属性使用此批注进行注释**
> 如果注释了多个方法，则可能抛出异常.

### @JsonGeter
@JsonGeter 是 @JsonProperty 的替代, 用于将方法标记为属性的getter方法
在序列化时会调用方法获取属性值.

```java
public class MyBean {
    public int id;
    private String name;
 
    @JsonGetter("MyName")
    public String getTheName() {
        return name;
    }
}

```
```json
{
  "id" : 10,
  "MyName" : "wkunc"
}
```
相当于只能用在方法上的@JsonProperty. 并且只对序列化产生影响.

> 标记注释,可用于定义非静态,无参数值返回(非空)方法,
> 以用作逻辑属性的"getter".
> 它可以用作更通用的JsonProperty注释的替代(在一般情况下是推荐的选择).
> Getter意味着当序列化具有此方法的类的Object实例(可能从超类继承)时,
> 通过该方法进行调用,并且返回值将被序列化为属性的值.

### @JsonPeropertyOrder
我们可以使用 @JsonPropertyOrder 注解来声明序列化时属性的顺序.

```java
//@JsonPropertyOrder(alphabetic=true), 使用字母顺序排序, 推荐用途
@JsonPropertyOrder({ "name", "id" })
public class MyBean {
    public int id;
    public String name;
}
```
```json
{
    "name":"My bean",
    "id":1
}
```
>可用于定义序列化对象属性时要使用的排序(可能是部分)的注释。
> 注释声明中包含的属性将首先（按定义的顺序）序列化，
> 然后是定义中未包含的任何属性。
> 注释定义将覆盖任何隐式排序
> （例如保证Creator-properties在非创建者属性之前被序列化） 

> 此注释可能会或可能不会对反序列化产生影响：
> 对于基本JSON处理没有任何影响，
> 但对于其他受支持的数据类型（或结构约定）可能存在影响。
> 注意：允许注释属性，从2.4开始，
> 主要是为了支持java.util.Map条目的字母顺序。

### @JsonRawValue

### @JsonValue

### @JsonRootName

### @JsonSerialize

## Deserialization Annotation
反序列化注解

### @JsonCreator

### @JacksonInject
### @JsonAnySetter
### @JsonSetter
### @JsonDeserialize
### @JsonAlias

## Property Inclusion Annotations
指示Jackson是否包含被标记属性的注解.
默认情况下会将对象的所有属性
所以提供了各种用来标识哪些属性是不需要的.

### @JsonIgnoreProperties
### @JsonIgnore
### @JsonIgnoreType
### @JsonInclude
### @JsonAutoDetect

## Polymorphic Type Handling Annotations
多态类型处理注解.
* @JsonTypeInfo
* @JsonSubTypes
* @JsonTypeName

## General Annotations
通用注解

@JsonProperty
@JsonFormat
@JsonUnwrapped
@JsonView
@JsonManagedReference, @JsonBackReference
@JsonIdentityInfo
@JsonFilter


