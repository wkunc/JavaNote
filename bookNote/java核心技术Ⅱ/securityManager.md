# 安全管理器与访问权限

## 权限检测

安全管理器(SecurityManager)是一个负责控制具体操作是否允许执行类.
安全管理器负责检查的操作包括以下内容:

* 创建一个新的类加载器
* 退出虚拟机
* 使用反射访问另一个类的成员
* 访问本地文件
* 打开 socket 连接
* 启动打印作业
* 访问系统剪切板
* 访问 AWT 事件队列
* 打开一个顶层窗口

在运行Java应用程序时, 默认的设置是不安装 SecurityManager 的.
这样所有的操作都是允许的.

# Java 平台安全性

它的*安全策略*建立了*代码来源*和*访问权限集*之间的映射关系

**代码来源(code source)**是由一个*代码位置*和一个*证书集*指定的.
java.security.CodeSource.

```java
// 一个URL 代码位置, 和 Certificate 证书数组.
public CodeSource(URL url, java.security.cert.Certificate[] certs) {
    this.location = url;
    if (url != null) {
        this.locationNoFragString = URLUtil.urlNoFragString(url);
    }

    // Copy the supplied certs
    if (certs != null) {
        this.certs = certs.clone();
    }
}
```

**权限(permission)**是指由安全管理器负责检查的任何属性.
Java 平台支持许多访问权限类, 每个类都封装了特定权限的详细信息.
权限类的父类是 java.security.Permission


---

每个类都有一个*保护域*, 它是一个用于封装类的*代码来源*和*权限集合*的对象.
当 SecurityManager 需要检查某个权限时, 它要查看当前位于调用堆栈上的
所有方法的类， 然后它要获得所有类的*保护域*, 并且询问每个保护域,
其权限集合是否允许执行当前正在被检查的操作. 如果所有的域都同意,
那么检查得以通过. 否则就会抛出一个 SecurityException 异常

保护域的表示类: java.security.ProtectionDomain

```java
//这是 Class 类中的方法可以获取不同类的保护域
public java.security.ProtectionDomain getProtectionDomain(){
    //...
}
```

# 安全策略文件

策略管理要读取相应的*策略文件*, 这些文件包含了将代码来源映射为权限的指令.

默认情况下, 有两个位置可以安装策略文件

* Java平台主目录的 java.policy 文件
* 用户主目录的 .java.policy 文件

在默认情况下, Java 应用程序是不安装安全管理器的. 因此, 在安装安全管理器之前,
看不到策略文件的作用. 当然,可以将下面的代码加入 main() 方法中,或用-Djava.security.manager

```java
System.setSecurityManager(new SecurityManager());
```

## 策略文件的格式

一个策略文件包含一系列的 grant(发放) 项, 每一项都具有以下格式.

```
grant codesource
{
    permission1;
    permission2;
    ...
};
```

codesource 包含一个代码基和值得信赖的用户特征(principal)与证书签名者的名字.

代码基表示具体的来源如下:
如果URL以 "/" 结束, 那么它是一个目录. 否则它将被视为一个JAR文件的名字.

```
codeBase "url"
codeBase "jrt:/java.compiler"
codeBase "www.horstmann.com/classes/"
codeBase "www.horstmann.com/classes/MyApp.jar"
```

权限部分采用以下格式.

```
permission className targetName, actionList;
```

className 对应具体的权限类的全类名.
targetName 与具体权限类有关的值, 如 FilePermission 中的目录或文件名, SocketPermission 中的主机或端口号.
actionList 同样与具体权限有关, 如文件权限中的 read, write, delete, Socket权限中的 read, connect等操作.

有些权限类并不需要 targetName 和 actionList. 这两个和具体权限类有关.

# 用户认证

Java API 提供了一个名为 Java认证和授权服务的框架,
它将平台提供的认证与权限管理集成起来又名 JAAS 框架
Java Authenticaion and Authorization Service.
Authenticaion 认证
Authorization 授权

它包含两部分认证和授权:
认证主要负责确定程序使用者的身份,
而

# JCA/JCE

Java密码学架构 (Java Cryptography Architecture JCA)
Java密码学扩展 (Java Cryptography Extension JCE)
Java安全socket扩展 (Java Secure Sockets Extension JSSE): 基于 SSL 协议的加密功能.
