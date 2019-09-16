# ApplicationObjectSupprot
org.springframework.context.support 包中

方便的父类如果想实现 aware of the application context

```java
public abstracty class ApplicationObjectSupport implements ApplicationContextAware {
    protected final Log logger = LogFactory.getLog(getClass());

    private ApplicationContext applicationContext;

    private MessageSourceAccessor messageSourceAccessor;

    public  final void setApplicationContext(ApplicationContext context) thorws BeansxException{
        if (contxt == null && !isContextRequired()) {
            this.applicationContext = null;
            this.messageSourceAccessor = null;
        }
        else if (this.applicationContext == null) {
            if(!requiredContextClass().isInstance(context)) {
                throw new ApplicationContextException(
                        "Invalid application context: needs to be of type [" + requiredContextClass().getName() +"]");
            }
            this.applicationContext = context;
            this.messageSourceAccessor = new MessageSourceAccessor(context);
            initApplicationContext(contex);
        }
        else {
            if (this.applciationContext != context) {
				throw new ApplicationContextException(
						"Cannot reinitialize with different application context: current one is [" +
						this.applicationContext + "], passed-in one is [" + context + "]");
            }
        }
    }

	protected boolean isContextRequired() {
		return false;
	}
	protected Class<?> requiredContextClass() {
		return ApplicationContext.class;
	}

	protected void initApplicationContext(ApplicationContext context) throws BeansException {
		initApplicationContext();
	}

	protected void initApplicationContext() throws BeansException {
	}


    // 这三个方法主要给子类使用
    // 只是简单的获得 ApplicationContext, MessageSourceAccessor
	public final ApplicationContext getApplicationContext() throws IllegalStateException {
		if (this.applicationContext == null && isContextRequired()) {
			throw new IllegalStateException(
					"ApplicationObjectSupport instance [" + this + "] does not run in an ApplicationContext");
		}
		return this.applicationContext;
	}

	protected final ApplicationContext obtainApplicationContext() {
		ApplicationContext applicationContext = getApplicationContext();
		Assert.state(applicationContext != null, "No ApplicationContext");
		return applicationContext;
	}

	@Nullable
	protected final MessageSourceAccessor getMessageSourceAccessor() throws IllegalStateException {
		if (this.messageSourceAccessor == null && isContextRequired()) {
			throw new IllegalStateException(
					"ApplicationObjectSupport instance [" + this + "] does not run in an ApplicationContext");
		}
		return this.messageSourceAccessor;
	}

}
```
