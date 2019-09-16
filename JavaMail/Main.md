# JavaMail 的基本使用

# 发送邮件
发送邮件可以分为三步:
1. 获得 java.mail.Session 对象
2. 构建 java.mail.Message 对象
3. 使用 Transport 发送 Message 对象.

Session 类表示邮件会话, 它收集JavaMailAPI使用的属性和默认值.

Session 类提供对实现Store, Transport 和相关的协议提供程序的访问.

Session 类提供了两组获取Session对象的静态方法.
```java
public static Session getInstance(Properties props, Authenticator authenticator);
public static Session getInstance(Properties props);
public static synchorized Session getDefaultInstance(Properties props, Authenticator authenticator);
public static synchorized Session getDefaultInstance(Properties props);
```
----

Message abstract 类,通常使用其子类 MimeMessage.
Message 是电子邮件的抽象, 有两种获取方式
1. 使用构造器创建新的对象(ps: 通常用来创建需要发送的邮件)
2. 从 Folder 中获得(ps: 文件夹指邮件服务器中的文件夹, 这种方式用在读取邮件中)

## 构造Message对象


---
Transport 类是发送信息的类, 它隐藏了底层的细节, 它也是一个抽象类.
它提供了两个静态方法发送邮件.
```java
public static void send(Message msg) throws MessagingException {
}
public static void send(Message msg, Address[] addresses) throws MesException {
}
```

