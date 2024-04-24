# JsonFactory

这是 jackson-core 包中的, 这个包重要提供了 Event Stream 流API.

这几个机制和XML解析时是同一个思路, 通过事件流来处理XML文档.

# Reading from Stream-of-Events

事件流是一种抽象, 不是具体的事物, 首先要决定如何暴露它或者说表示它.
这有许多可能性, 这里列举常见的3种方案.

1. 作为可迭代的事件对象流, 这是Stax Event API的做法, 优点是访问的简单性和对象封装, 允许在处理期间保留Event对象.
2. 作为在事件发生时表示事件回调, 将所有数据作为回调参数传递.这是 SAX API的做法(这个比较熟悉)
3. 作为允许一次访问关于一个事件的逻辑游标, 这是 Stax Cursor API 的做法.

JackJosn 使用了第三种方法来实现它的Event Stream API.
"JsonParser" 对象就是它暴露出的 cursor.

为了迭代事件流, 应用程序应该调用 JsonParser.nextToken().
来推进游标(Jackson 更喜欢术语"令牌"而不是"事件").
并访问令牌游标指向的数据和属性.

readJsonToJavaObject

```java
JsonFactory factory = new JsonFactory();
factory.enable(JsonParser.Feature.ALLOW_COMMENTS);

JsonParser jp = factory.createParser(new File("..."));

if(jp.nextToken() != JsonToken.START_OBJECT) {
    throw new IOException("Excepted data to start with an Object");
}

TwitterEntry result = new TwitterEntry();

while (jp.nextToken() != JsonToken.END_OBJECT) {
    String fieldName = jp.getCurrentName();
    jp.nextToken();

    if (fieldName.equals("id")) {
        result.setid(jp.getLongValue());
    } else if (fieldName.equals("text")) {
        result.setText(jp.getText());
    } else if (fieldName.equals("fromUserId")) {
        result.setFromUserId(jp.getIntValue());
    } else if (fieldName.equals("toUserId")) {
        result.setToUserId(jp.getIntValue());
    } else if (fieldName.equals("languageCode")) {
        result.setLanguageCode(jp.getText());
    } else {
        throw new IOException("Unrecognized field '" + fieldName +"'");
    }
}
```
