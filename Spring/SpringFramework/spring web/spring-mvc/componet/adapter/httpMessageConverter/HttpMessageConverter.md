# HttpMessageConverter

```java
public interface HttpMessageConverter<T> {

    // 指示转换器是否支持从特定(MediaType)媒体类型的 request 中读取信息成为指定Java类型对象
	boolean canRead(Class<?> clazz, @Nullable MediaType mediaType);

    // 指示转换器是否支持从特定(MediaType)媒体类型的 reponse 中读取信息成为指定Java类型对象
	boolean canWrite(Class<?> clazz, @Nullable MediaType mediaType);

	List<MediaType> getSupportedMediaTypes();

	T read(Class<? extends T> clazz, HttpInputMessage inputMessage)
			throws IOException, HttpMessageNotReadableException;

	void write(T t, @Nullable MediaType contentType, HttpOutputMessage outputMessage)
			throws IOException, HttpMessageNotWritableException;

}
```

# AbstractHttpMessageConverter
为子类实现提供便利, 利用一个内部的List\<MediaType>实现了接口中的 getSupportedMediaTypes() 方法.
并提供这个内部List的初始化逻辑和setter方法.

并且用一个 Charset 字段标记后面需要使用的默认编码集.

然后将接口中的 canRead(Class, MediaType) 和 canWrite(Class,MediaType) 拆分逻辑并实现.

最后提供 read() write()

```java
public abstract class AbstractHttpMessageConverter<T> implements HttpMessageConverter<T> {

    private List<MediaType> supportedMediaTypes = Collections.emptyList();
    private Charset defaultCharset;


    // 实现HttpMessageConverter接口中的方法, 将逻辑分为, supports(Class) , canRead(mediaType)
    // supports() 方法是一个抽象方法, 由子类实现, 因为是否支持指定的Java类型是子类的事.
    // 为 canRead(MediaType)提供默认实现, 利用内部的 支持的媒体类型列表 判读是否支持.
	public boolean canRead(Class<?> clazz, @Nullable MediaType mediaType) {
		return supports(clazz) && canRead(mediaType);
	}

	protected boolean canRead(@Nullable MediaType mediaType) {
		if (mediaType == null) {
			return true;
		}
		for (MediaType supportedMediaType : getSupportedMediaTypes()) {
			if (supportedMediaType.includes(mediaType)) {
				return true;
			}
		}
		return false;
	}
    

    // HttpMessageConverter中的方法, 实现逻辑和上面的一样
	public boolean canWrite(Class<?> clazz, @Nullable MediaType mediaType) {
		return supports(clazz) && canWrite(mediaType);
	}

	protected boolean canWrite(@Nullable MediaType mediaType) {
		if (mediaType == null || MediaType.ALL.equalsTypeAndSubtype(mediaType)) {
			return true;
		}
		for (MediaType supportedMediaType : getSupportedMediaTypes()) {
			if (supportedMediaType.isCompatibleWith(mediaType)) {
				return true;
			}
		}
		return false;
	}

    // 实现接口中的read()方法, 直接委托给内部的 readInternal() 方法.
    // 由于读取请求的过程没有什么相识的步骤. 不同的requset的内容不一定相同, 所以没有额外的逻辑.
	public final T read(Class<? extends T> clazz, HttpInputMessage inputMessage)
			throws IOException, HttpMessageNotReadableException {

		return readInternal(clazz, inputMessage);
	}

    // 和read()方法不同, 虽然都是委托给内部的抽象方法 writeInternal().
    // 但是一个服务的生成的 responses 通常会具有一些共有的header.
    // 所以这个 write() 方法提供了一个模板方法, 
    // addDefaultHeaders()
	public final void write(final T t, @Nullable MediaType contentType, HttpOutputMessage outputMessage)
			throws IOException, HttpMessageNotWritableException {

		final HttpHeaders headers = outputMessage.getHeaders();
		addDefaultHeaders(headers, t, contentType);

		if (outputMessage instanceof StreamingHttpOutputMessage) {
			StreamingHttpOutputMessage streamingOutputMessage = (StreamingHttpOutputMessage) outputMessage;
			streamingOutputMessage.setBody(outputStream -> writeInternal(t, new HttpOutputMessage() {
				@Override
				public OutputStream getBody() {
					return outputStream;
				}
				@Override
				public HttpHeaders getHeaders() {
					return headers;
				}
			}));
		}
		else {
			writeInternal(t, outputMessage);
			outputMessage.getBody().flush();
		}
	}

}
```


## StringHttpMessageConverter
首先看一个比较简单具体实现 *StringHttpMessageConverter*
可以将 request 对读取成为 String, 将String对象写入到 response 中的消息转换器.
```java
public class StringHttpMessageConverter extends AbstractHttpMessageConverter<String> {

	public static final Charset DEFAULT_CHARSET = StandardCharsets.ISO_8859_1;

	@Nullable
	private volatile List<Charset> availableCharsets;

	private boolean writeAcceptCharset = true;


    // 从构造器过程中可以看出, 这个消息转换器支持的媒体类型是 text-plain, all
	public StringHttpMessageConverter() {
		this(DEFAULT_CHARSET);
	}

	public StringHttpMessageConverter(Charset defaultCharset) {
		super(defaultCharset, MediaType.TEXT_PLAIN, MediaType.ALL);
	}

    // 实现 supports(Class<?> class), 只支持String类型
	public boolean supports(Class<?> clazz) {
		return String.class == clazz;
	}

    // 实现 readInternal, 首先是确定使用的 Charset, 然后通过 InputStream 读取byte,用确定好的编码集转换成对应的String
    // 1. 通过Content-type请求头确定,
    // 2. 如果这个content-type虽然不为空, 但是没有提供使用编码集的信息 content-type: text/plain
    //   就判断这个 contenty-type 是否和 application/json 兼容.(text/* 和 text/plain, text/html兼容)
    //   如果如果判断通过就使用 UTF-8 编码集.
    // 3. 最后使用设置的 defaultCharset.
    // 默认的无参构造器设置了 ISO-8859-1 为默认编码集,
    // 另一个构造器是由传入的值决定, 所以默认编码集可能是空的所以在 getContentTypeCharset() 方法中还进行了对应的不为null断言.
	protected String readInternal(Class<? extends String> clazz, HttpInputMessage inputMessage) throws IOException {
		Charset charset = getContentTypeCharset(inputMessage.getHeaders().getContentType());
		return StreamUtils.copyToString(inputMessage.getBody(), charset);
	}

}
```


# GenericHttpMessageConverter
接下来看一个特定的子接口, 


和父接口HttpMessageConverter很像. 是为了泛型类型的特化.
定义的方法和父接口的方法也就多了一个 Type, 代表泛型类型.
```java
public interface GenericHttpMessageConverter<T> extends HttpMessageConverter<T> {

	boolean canRead(Type type, @Nullable Class<?> contextClass, @Nullable MediaType mediaType);

	T read(Type type, @Nullable Class<?> contextClass, HttpInputMessage inputMessage)
			throws IOException, HttpMessageNotReadableException;

	boolean canWrite(@Nullable Type type, Class<?> clazz, @Nullable MediaType mediaType);

	void write(T t, @Nullable Type type, @Nullable MediaType contentType, HttpOutputMessage outputMessage)
			throws IOException, HttpMessageNotWritableException;

}
```
