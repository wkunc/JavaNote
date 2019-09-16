```java
public abstract class AbstractRefreshableConfigApplicationContext extends AbstractRefreshableApplicationContext
		implements BeanNameAware, InitializingBean {

	@Nullable
	private String[] configLocations;

	private boolean setIdCalled = false;


	public AbstractRefreshableConfigApplicationContext() {
	}

	public AbstractRefreshableConfigApplicationContext(@Nullable ApplicationContext parent) {
		super(parent);
	}


	public void setConfigLocation(String location) {
		setConfigLocations(StringUtils.tokenizeToStringArray(location, CONFIG_LOCATION_DELIMITERS));
	}

	public void setConfigLocations(@Nullable String... locations) {
		if (locations != null) {
			Assert.noNullElements(locations, "Config locations must not be null");
			this.configLocations = new String[locations.length];
			for (int i = 0; i < locations.length; i++) {
				this.configLocations[i] = resolvePath(locations[i]).trim();
			}
		}
		else {
			this.configLocations = null;
		}
	}

	@Nullable
	protected String[] getConfigLocations() {
		return (this.configLocations != null ? this.configLocations : getDefaultConfigLocations());
	}

	@Nullable
	protected String[] getDefaultConfigLocations() {
		return null;
	}

	protected String resolvePath(String path) {
		return getEnvironment().resolveRequiredPlaceholders(path);
	}


	@Override
	public void setId(String id) {
		super.setId(id);
		this.setIdCalled = true;
	}

	@Override
	public void setBeanName(String name) {
		if (!this.setIdCalled) {
			super.setId(name);
			setDisplayName("ApplicationContext '" + name + "'");
		}
	}

	@Override
	public void afterPropertiesSet() {
		if (!isActive()) {
			refresh();
		}
	}
}
```