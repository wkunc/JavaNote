# RestTemplate
提供了11种不同的方法, 发送http请求.
10种有3种重载, 1种有8个重载, 一共38个.

1. delete 3
2. exchange 8
2. execute 3
3. getForEntity 3
4. getForObject 3
5. headForHeaders 3
6. optionsForAllow 3
7. patchForObject 3
8. postForEntity 3
9. postForLocation 3
10. postForObject 3
11. put 3

这些方法最后都是调用 execute(),
execute() 最后又都是调用一个内部方法 doExecute()
```java
public <T> T execute(String url, HttpMethod method, @Nullable RequestCallback requestCallback,
        @Nullable ResponseExtractor<T> responseExtractor, Object... uriVariables) throws RestClientException {

    URI expanded = getUriTemplateHandler().expand(url, uriVariables);
    return doExecute(expanded, method, requestCallback, responseExtractor);
}

public <T> T execute(String url, HttpMethod method, @Nullable RequestCallback requestCallback,
        @Nullable ResponseExtractor<T> responseExtractor, Map<String, ?> uriVariables)
        throws RestClientException {

    URI expanded = getUriTemplateHandler().expand(url, uriVariables);
    return doExecute(expanded, method, requestCallback, responseExtractor);
}

public <T> T execute(URI url, HttpMethod method, @Nullable RequestCallback requestCallback,
        @Nullable ResponseExtractor<T> responseExtractor) throws RestClientException {

    return doExecute(url, method, requestCallback, responseExtractor);
}
```

```java
protected <T> T doExecute(URI url, @Nullable HttpMethod method, @Nullable RequestCallback requestCallback,
        @Nullable ResponseExtractor<T> responseExtractor) throws RestClientException {

    Assert.notNull(url, "URI is required");
    Assert.notNull(method, "HttpMethod is required");
    ClientHttpResponse response = null;
    try {
        // 1. 创建请求
        ClientHttpRequest request = createRequest(url, method);
        // 2. 应用callback
        if (requestCallback != null) {
            requestCallback.doWithRequest(request);
        }
        // 3. 发送请求
        response = request.execute();
        // 4. 处理response, 主要是处理异常状态码4xx,5xx
        handleResponse(url, method, response);
        // 5. 调用 extractor 转换成目标类型.
        return (responseExtractor != null ? responseExtractor.extractData(response) : null);
    }
    catch (IOException ex) {
        String resource = url.toString();
        String query = url.getRawQuery();
        resource = (query != null ? resource.substring(0, resource.indexOf('?')) : resource);
        throw new ResourceAccessException("I/O error on " + method.name() +
                " request for \"" + resource + "\": " + ex.getMessage(), ex);
    }
    finally {
        if (response != null) {
            response.close();
        }
    }
}
```
# RestTemplate 具体实现

## HttpAccessor

```java
public abstract class HttpAccessor {

    private ClientHttpRequestFactory requestFactory = new SimpleClientHttpRequestFactory();

    // 5.2 添加的新接口, 与 ClientHttpRequestInterceptor 相比不需要将 request body 加载到内存.
    private final List<ClientHttpRequestInitializer> clientHttpRequestInitializers = new ArrayList<>();

	public void setRequestFactory(ClientHttpRequestFactory requestFactory) {
		Assert.notNull(requestFactory, "ClientHttpRequestFactory must not be null");
		this.requestFactory = requestFactory;
	}

	public ClientHttpRequestFactory getRequestFactory() {
		return this.requestFactory;
	}

	public void setClientHttpRequestInitializers(
			List<ClientHttpRequestInitializer> clientHttpRequestInitializers) {

		if (this.clientHttpRequestInitializers != clientHttpRequestInitializers) {
			this.clientHttpRequestInitializers.clear();
			this.clientHttpRequestInitializers.addAll(clientHttpRequestInitializers);
			AnnotationAwareOrderComparator.sort(this.clientHttpRequestInitializers);
		}
	}


	public List<ClientHttpRequestInitializer> getClientHttpRequestInitializers() {
		return this.clientHttpRequestInitializers;
	}

    // 主要逻辑, 调用RequestFactory创建 ClientHttpRequest 后使用初始化器进行初始化
    // 比较特殊的点是 通过 getRequestFactory() 方法获取 ClientFactory, 而不是直接访问字段
    // 
	protected ClientHttpRequest createRequest(URI url, HttpMethod method) throws IOException {
		ClientHttpRequest request = getRequestFactory().createRequest(url, method);
		initialize(request);
		if (logger.isDebugEnabled()) {
			logger.debug("HTTP " + method.name() + " " + url);
		}
		return request;
	}

	private void initialize(ClientHttpRequest request) {
		this.clientHttpRequestInitializers.forEach(initializer -> initializer.initialize(request));
	}
}

public interface ClientHttpRequestInitializer {
    void initialize(ClientHttpRequest request);
}
```

添加了拦截器机制的 Http 访问器.
并且可以看到这里重写了 getRequestFactory() 方法.
这也是为什么 HttpAccessor 要通过 getRequestFactory() 方法进行访问而不是直接通过字段.
```java
public abstract class InterceptingHttpAccessor extends HttpAccessor {

	private final List<ClientHttpRequestInterceptor> interceptors = new ArrayList<>();

	@Nullable
	private volatile ClientHttpRequestFactory interceptingRequestFactory;


	public void setInterceptors(List<ClientHttpRequestInterceptor> interceptors) {
		// Take getInterceptors() List as-is when passed in here
		if (this.interceptors != interceptors) {
			this.interceptors.clear();
			this.interceptors.addAll(interceptors);
			AnnotationAwareOrderComparator.sort(this.interceptors);
		}
	}

	public List<ClientHttpRequestInterceptor> getInterceptors() {
		return this.interceptors;
	}

	@Override
	public void setRequestFactory(ClientHttpRequestFactory requestFactory) {
		super.setRequestFactory(requestFactory);
		this.interceptingRequestFactory = null;
	}

	@Override
	public ClientHttpRequestFactory getRequestFactory() {
		List<ClientHttpRequestInterceptor> interceptors = getInterceptors();
		if (!CollectionUtils.isEmpty(interceptors)) {
			ClientHttpRequestFactory factory = this.interceptingRequestFactory;
			if (factory == null) {
				factory = new InterceptingClientHttpRequestFactory(super.getRequestFactory(), interceptors);
				this.interceptingRequestFactory = factory;
			}
			return factory;
		}
		else {
			return super.getRequestFactory();
		}
	}

}
```

## ClientHttpRequestFactory
上面 HttpAccessor 到 InterceptingHttpAccessor, 都用到了 ClientHttpRequestFactory 抽象.


很直白的抽象, 通过 createRequest() 方法创建一个 ClientHttpRequest 的工厂.
```java
public interface ClientHttpRequestFactory {

    ClientHttpRequest createRequest(URI uri, HttpMethod httpMethod) throws IOException;

}
```

[](image/ChttpHttpRequestFactory.png)

通过继承实现结构. 很明显可以看出用了一个`装饰器模式`

`AbstractClientHttpRequestFactoryWrapper` 是装饰器的基类.
提供了拦截器机制的装饰器, 以及可重复读的装饰器.

和Java IO 流的设计一样, 出了装饰器, 还提供了基于不同的底层实现的实现类.

1. SimpleClientHttpRequestFactory -> Java 原生 net 包下通过 HttpURLConnection 实现
2. HttpComponentsClientHttpRequestFactory -> 基于 Apache HttpComponents 库实现
3. OkHttp3ClientHttpRequestFactory -> 基于 OkHttp 实现
4. RibbonClientHttpRequestFactory -> 基于 Ribbon 实现(通过源码, 可以读出使用 Ribbon 时 URI 中的host应该写 serviceId)
5. Netty4ClientHttpRequestFactory -> 基于Netty4实现, 不过已经在5中废弃了, 推荐使用WebClient. 也就是Reactive那一套

## ClientHttpRequest and ClientHttpResponse

和ClientHttpRequsetFactory一样, 既然每个请求机制有对应的ClientHttpRequestFactory实现.
那么每个机制的 ClientHttpRequest 以及 ClientHttpRespose 的实现都是独立.

```java
public interface ClientHttpRequest extends HttpRequest, HttpOutputMessage {

    ClientHttpResponse execute() throws IOException;

}

public interface ClientHttpResponse extends HttpInputMessage, Closeable {

    HttpStatus getStatusCode() throws IOException;

    int getRawStatusCode() throws IOException;

    String getStatusText() throws IOException;

    void close();

}
```

从这个接口上, 我们可以看到熟悉的 HttpOutputMessage, HttInputMessage, 以及HttpRequest

HttpOutputMessage, HttInputMessage 是在 HttMessageConverter 机制中学习过.
这里用到呢也是出于同样的目的, 利用 HttMessageConverter 机制进行Request,Response的转换.
这一点会在 RestTemplate 中看到.

