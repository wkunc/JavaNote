# Network

java.net 提供以下与寻址相关的类:
1. InetAddress
2. Inet4Address
3. Inet6Address
4. SocketAddress
5. InetSocketAddress

对于IP地址, 提供了三大类: InetAddress, Inet4Address ,Inet6Address.
InetAddress 代表IP地址, 该地址是IP使用的32位或128位无符号数字,
主要使用方式: InetAddress 类上提供的6个静态方法.

对于套接字寻址, 提供了两个类: SocketAddress 和 InetSocketAddress.

TCP 连接
ServerSocket
Socket

UDP
DatagramPacket
DatagramSocket

定位/识别网络资源:
1. URI
2. URL
3. URLClassLoader
5. URLStreamHandler
4. URLConnection
6. HttpURLConnection
7. JarURLConnection

# URL
URL类代表一个统一资源定位符, 它是指向Internet上 'resource' 的指针.
资源可以是简单的文件或目录, 也可以是对更复杂对象的引用, 例如对数据库或搜索引擎的查询.


NOET: URL类本身不会根据RFC2396种定义的转义机制对任何URL组件进行编码或解码.
调用方有责任对调用URL之前需要转义的所有字符进行编码,
并对从URL返回的所有字符进行解码.

TIP: URI类在某些情况下会对字符进行转义, 管理编码和URL解码的推荐方法是使用URI,
并使用toURI() 和 URI.toURL()在它们之间进行转换.
URLEncoder 和 URLDecoder 类也可以用来编码解码, 但是只为HTML形式编码, 和RFC2396定义的不同.

URL

getAuthority()
getDefaultport()
getFile()
getHost()
getPath()
getPart()
getProtocol()
getQuery()
getRef()
getUserInfo()
sameFile()

特殊方法:
openConnection()
openConnection(Proxy)
getContent()
getContent(Class[] classes)
openStream()


# HTTP Client

1. URLConnection Caching API
2. Cookie Management
3. Http authentication
4. Persistent Connections (http持久连接 keep-alive)

