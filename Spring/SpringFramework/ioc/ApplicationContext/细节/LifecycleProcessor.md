# LifecycleProcessor

```java
public interface Lifecycle {
    void start();
    void stop();
    boolean isRunning();
}
```

```java
public interface LifecycleProcessor extends Lifecycle {
    void onRefresh();
    void onClose();
}
```

```java
public class DefaultLifecycleProcessor implements LifecycleProcessor, BeanFactoryAware {
    private final Log logger = LogFactory.getLog(getClass());
    private volatile long timeoutPerShutdownPhase = 30000;
    private volatile boolean running;
    private voilatle ConfigurableLisabletBeanFactory beanFactory;

    public void setTimeoutPerShutdownPhase(long timeoutPerShutdownPhase) {
        this.timeoutPerShutdownPhase = timeoutPerShutdownPhase;
    }

    @Override
    public void setBeanFactory(BeanFactory beanFactory) {
        if (!(beanFactory instanceof ConfigurableListableBeanFactory)) {
			throw new IllegalArgumentException(
					"DefaultLifecycleProcessor requires a ConfigurableListableBeanFactory: " + beanFactory);
        }
        this.beanFactory = (ConfigurableListableBeanFactory)beanFactory;
    }

    // Lifecycle 实现
    @Override
    public void start() {
        startBeans(false);
        this.running = true;
    }

    @Override
    public void stop() {
        stopBeans();
        this.runing = false;
    }

    @Override
    public void onRefresh() {
        startBeans(true);
        this.running = true;
    }

    @Override
    public void onClose() {
        stopBeans();
        this.running = false;
    }

    @Override
    public boolean is Running() {
        return this.runnig;
    }

    // Internal helpers


}
```
