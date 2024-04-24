# 2.Spring Cloud Commons:Common Abstractions

Service discovery, load balancing and circuit breakers 等模式的通用抽象层
可以由所有Spring Cloud client 使用, 而与实现无关

## The @EnableDiscoveryClient Annotation

Spring Cloud Commons 提供了`@EnableDiscoveryClient` Annotation.
这将查找具有META-INF/spring.factories 的DiscoveryClient和ReactiveDiscoveryClient接口实现.

DiscoveryClient 实现的包括 Spring Cloud Netfix Eureka, Spring Cloud Consul Discovery
Spring Cloud Zookeeper Discover.

### Health Indicator (健康指示符)

Commons create a Spring Boot HealthIndicator that DiscoveryClient implementations can
participate in by implementing DiscoverHelthIndicator.

### Ordering DiscoverClient instances

### SimpleDiscoveryClient

如果classpath下没有支持service registry的 DiscoveryClient 实现.
则将使用 properties 获取有关服务和实例信息的 SimpleDiscoveryClient 实现.

可用实例应通过以下格式的属性传递:

spring.cloud.discovery.client.simple.instances.service1[0]=http://s11:8080
其中 spring.cloud.discovery.client.simple.instances. 是通用前缀.
servcie1 代表服务ID, [0] 表示实例的index(从零开始).

## ServiceRegistry

Commons 现在提供一个`ServiceRegistry`接口, 该借口提供诸如
registry(Registration) 和 deregister(Registration) 之类的方法.
Registration是一个标记借口.

```java
@Configuration
@EnableDiscoveryClient(autoRegister=false)
public class MyConfiguration {

    private ServiceRegistry registry;

    public MyConfiguration(ServiceRegistry registry) {
        this.registry = registry
    }

    // called throu
    public void register() {
        Registration registration = constructiRegistation();
        this.registry.register(registration);
    }

}
```

```java
public interface ServiceRegistry<R extends Registration> {

    void register(R registration);

    void deregister(R registration);

    void close();

    void setStatus(R registration, String status);

    <T> T getStatus(R registration);
}
```

## Spring RestTemplate as a Load Balancer Client

```java
@Configuration
public class MyConfiguration {

    @LoadBalanced
    @Bean
    RestTemplate restTemplate() {
        return new RestTemplate();
    }


}

public class MyClass {
    @Autowired
    private RestTemplate restTemplate;

    public String doOtherStuff() {
        String result = restTemplate.getForObject("http://stores/stores", String.class);
        return result;
    }
}
```

## 2.9 HTTP Client Factories

Spring Cloud Commons 提供了用于创建 Apache Http client(ApacheHttpClientFactory)
和 Ok HTTP 客户端 ( OkHttpClientFactory )的bean.
仅当 OK HTTP jar位于 classpath 上时才创建 OkHttpClientFactory bean.


