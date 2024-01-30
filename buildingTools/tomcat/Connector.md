# Connector
Connector (连接器) 组件是 Tomcat 最核心的组件, 
主要职责是负责接收客户端连接和客户端请求的处理加工.

每个Connector都将指定一个端口进行监听,
分别负责对请求报文解析和对响应报文组装, 
解析过程生成Request对象, 而组装过程则涉及Response对象.

