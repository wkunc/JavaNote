# Spring Cloud LoadBalancer

Spring Cloud 提供了自己的客户端负载均衡抽象和实现.
对于负载均衡平衡机制, 已添加ReactiveLoadBalancer 借口, 并且为其提供了基于Round-Robin的实现.


## Spirng Cloud LoadBalancer integrations


```java

public interface LoadBalancerClient extends ServiceInstanceChooser {

    <T> T execute(String serviceId, LoadBalancerRequest<T> requset) throw IOException;

    <T> T execute(String serviceId, ServiceInstance serviceInstance,
            LoadBalancerRequest<T> requset) throw IOException;

    URI reconstructURI(ServiceInstance instance, URI original);

}

public interface ServiceInstanceChooser {

    ServiceInstance choose(String serviceId);

}

```
