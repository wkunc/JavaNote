# 2.3 High Availability, Zones and Regions (高可用,  区域与地区)

Eureka Server 没有后端存储, 但是注册表中的所有服务实例都必须发送心跳链接以使其注册保持最新.
(因此可以在内存中完成).
client 还具有Eureka注册的内存缓存(因此, 对于每个对服务的请求, 它们都不必进入注册表)

默认情况下, 每个 Eureka Server 也是一个 Eureka Client,
并且需要至少一个 service URL 来定位对等方.
如果不提供该服务(另一个Eureka Server), 则该服务可以运行, 但是它将使您的日志充满无法向对等方注册的噪音.

# Ribbon (客户端负载均衡工具)

处于正在维护状态, Spring 官方都不推荐使用了但是 eureka-server and eureka-client 的start包中依然包含其依赖.
同时也包含Spring Cloud LoadBalancer(官方出品) 推荐切换到这个上面去

# Circuit Breaker: Spring Cloud Circuit Breaker With Hystrix (熔断器)

