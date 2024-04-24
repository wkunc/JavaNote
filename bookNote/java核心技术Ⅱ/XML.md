# 6 流机制解析器

DOM 解析器会完整的读入 XML 文档, 然后将其转换成一个树形的结构.
对于大多数的应用, DOM 都运行的很好. 但是如果文件很大, 并且处理算法又非常简单,
可以运行时解析节点, 而不必看到完整的树形结构, 那么 DOM 可能就会显得效率低下了.
这种情况下, 我们应该使用 *流机制解析器 (streaming parser)*

在 Java 中有两个流机制解析器: 老而弥坚的 SAX 解析器 和 添加到 SE6 中更现代化的 StAX 解析器

SAX 解析器使用的是事件回调 (event callback), 而 StAX 解析器提供了遍历解析事件的迭代器,
后者通常用起来会更方便一些.

## 使用 SAX 解析器

SAX 解析器在解析 XML 输入数据的各个组成部分时会报告事件, 但不会一任何方式存储文件,
而是由**事件处理器**建立相应的数据结构.
事实上, DOM 解析器是在 SAX 解析器的基础上构建的, 它在收到解析器事件的时候构建 DOM 树.

在使用 SAX 解析器的时, 需要一个处理器来为各种解析器事件定义事件动作

ContentHandler 接口定义了若干个在解析文档时会调用的回调方法

```java
//每当遇到起始标签或终止标签时调用
public void startElement(String uri, String localName, String qName, Attribute atts) throws SAException;
public void endElement(String uri, String localName, String qName) throws SAException;
//遇到字符数据时调用
public void characters(char ch[], int start, int length) throw SAException;
//分别在文档开始和结束时各调用一次
public void startDocument() throws SAException;
public void endDocument() throws SAException;
```

如何获得使用 SAX 解析器

```java
SAXParserFactory factory = SAXParaserFactory.newInstance();
SAXParser parser = factory.newSAXParaser();
//这里的 source 可以是一个文件,一个URL字符串或者一个输入流, handler 属于 DefaultHandler 的一个子类
// DefaultHandler 中实现了 ContentHandler, DTDHandler, EntityResolver, ErrorHandler 接口的空方法
parser.parse(source, handler);
```
