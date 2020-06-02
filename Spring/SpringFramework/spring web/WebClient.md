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

首先学习两个简单的函数式接口. 分别表示对request, response的处理.
```java
/**
 * 请求时回调接口, 允许操作 request Header, 或者 request Body.
 *
 * 在 RestTemplate 内部使用, 也可以用于应用程序代码.
 * 提供的工厂方法
 * 1. RestTemplate.acceptHeaderRequestCallback(Class)
 * 2. RestTemplate.httpEntityCallback(Object)
 * 3. RestTemplate.httpEntityCallback(Object, Type)
 */
public interface RequestCallback { 
 
	void doWithRequest(ClientHttpRequest request) throws IOException;

}

public interface ResponseExtractor {

	@Nullable
	T extractData(ClientHttpResponse response) throws IOException;

}
```

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

