# 基础支持层

# 解析器模块
mybatis 的配置文件是通过**DOM解析方式**和**XPath**解析的
(ps:xpath 是一种为了查询XML而设计的语言, 相当于SQL对数据库)
Java5 中推出了 javax.xml.xpath 包

MyBatis 提供了 XPathParser 类封装了前面涉及的 XPath, Document, EntityResolver

