# Request
请求对象封装了客户端请求的所有信息
在 Http 协议中,这些信息是从客户端发送到服务器请求的 Http 头部和消息体
## Http 协议参数
servlet 的请求参数以字符串的形式作为请求的一部分从客户端发送到 servlet 容器
当请求是一个 HttpServletRequest 对象, 且符合"参数可用时" 描述时, 
容器从 URI 查询字符串和 Post 数据中填充参数
参数以一系列的 key-value 对的形式保存
给定的参数名称可能存在多个参数值
```java
public String getParameter(String name);
public Enumeration<String> getParameterNames();
public String[] getParameterValues(String name);
public Map<String, String[]> getParameterMap();
```
## 文件处理

## 属性
