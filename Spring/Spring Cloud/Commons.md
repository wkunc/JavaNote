# Spring Cloud Context: Application Context Services

## The Bootstrap Application Context

Spring Cloud 应用程序通过创建 "bootstrap" context 来运行.
该 context 是应用程序的父上下文(ps: 惊了, Spring Context 的父子关系在这里又用上了)

这两个上下文共享一个`Environment`, 该环境是任何Spring应用程序的外部属性来源.

默认情况下引导属性具有较高优先级, 因此不能被本地配置覆盖.

bootstrap Context 使用不同于主 applicationContext 的约定来定位外部配置
bootstrap.yml(bootstrap.properties)

# ServiceRegistry

```java
/*
* 服务注册表
*
*/
public interface ServiceRegistry<R extends Registration> {

    void register(R registration);

    void deregister(R registration);

    void close();

    void setStatus(R registration, String status);

    <T> T getStatus(R registration);

}


public interface Registration extends ServiceInstance {

}

/*
* 代表一个服务实例在 discovery system 中
*/
public interface ServiceInstance {

    // 返回服务实例的唯一id
    default String getInstanceId() {
        return null;
    }

    // 返回 service id
    String getServiceId();

    // 返回服务的 hostname
    String getHost();

    // 返回服务实例的端口号
    int getPort();

    // 返回服务实例是否使用 HTTPS
    boolean isSecure();

    // 返回 service URI address
    URI getUri();

    // 返回 key / value pair 形式的服务实例关联元信息
    Map<String, String> getMetadata();

    // 返回服务实例的 scheme
    default String getScheme() {
        return null;
    }

}
```

EurekaServiceRegistry

```java
public class EurekaServiceRegistry implements ServiceRegistry<EurekaRegistration> {




}
```
