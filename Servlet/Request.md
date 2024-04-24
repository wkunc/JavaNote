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
    //返回给定名字的参数的对应值,没有就返回null,如果你用这个方法访问有多个参数值的话只返回第一个值
    public String getParameter(String name);
    //返回所有参数名
    public Enumeration<String> getParameterNames();
    //返回给定参数名的所有对应值
    public String[] getParameterValues(String name);
    //返回所有的参数和参数值
    public Map<String, String[]> getParameterMap();
```

注意:
> 查询参数在post数据前发送,所以一个参数名多个参数值的情况下,查询参数值在前

这些 API 不会暴露 Get 请求的路径参数, 它们必须从 getRequestURI () 方法或 getPathInfo()
方法返回的字符串中解析

## 文件上传

当数据以 multipart/form-data 的格式发送时, servlet 容器支持文件上传.

如何使 request 中 multipart/form 类型的数据可用, 取决于 servlet 容器是否提供 multipart/form-data
格式数据的处理:
如果 servlet 容器提供这种处理,可通过 HttpServletRequest 中的以下方法得到:

```java
public Collection<Part> getPart() throws IOException, ServletException;
public Part getPart(String name) throws IOException, ServletException;
```

如果 servlet 容器不提供 multipart/from-data 格式的数据处理, 这些数据可以通过
HttpServletRequset.getInputStream 得到

## 属性
