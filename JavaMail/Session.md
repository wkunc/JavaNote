# Session具体实现分析

## 字段分析
```java
private final Properties props;
private final Authenticator authenticator;
private final Hashtable<URLName, PasswordAuthentication> authTable = new Hashtable<>();

private boolean debug = false;

private PrintStream out;
private MailLogger logger;

private List<Provider> providers;
private final Map<String, Providre> providersByProtocol = new HashMap<>();
private final Map<String, Providre> providersByClassName = new HashMap<>();
private final Properties addressMap = new Properties();
private boolean loadedProviders;
private final EventQueue q;
private static Session defaultSession = null;
private static final String confDir;
```

```java
static {
	String dir = null;
	try {
	    dir = AccessController.doPrivileged(
		new PrivilegedAction<String>() {
		    @Override
		    public String run() {
			String home = System.getProperty("java.home");
			String newdir = home + File.separator + "conf";
			File conf = new File(newdir);
			if (conf.exists())
			    return newdir + File.separator;
			else
			    return home + File.separator +
				    "lib" + File.separator;
		    }
		});
	} catch (Exception ex) {
	    // ignore any exceptions
	}
	confDir = dir;
}
```


构造器
```java
private Session(Properties props, Authenticator authenticator) {
    this.props = props;
    this.authenticator = authenticator;

    if (Boolean.valueOf(props.getProperty("mail.debug")).booleanValue())
        debug = true;
    
    initLogger();
    logger.log();

    //
    Class<?> cl;
    if (authenticator != null)
        cl = authenticator.getClass();
    else 
        cl = this.getClass();
    
    loadAddressMap(cl);
    q = new EventQueue((Executor)props.get("mail.evet.executor"));
}
```

```java
private void loadAddressMap(Class<?> cl) {
    StreamLoader loader = new StreamLoader() {
        public void load(InputStream is) throws IOException {
            addressMap.load(is);
        }
    }

    loadResource("/META-INF/javamail.default.address.map", cl, loader, true);
    loadAllResources("/META-INF/javamail.address.map", cl, loader);

    try {
        if (confDir != null)
            loadFile(confDir + "javamail.address.map", loader);
        } catch (SecurityException ex) {}

    if (addressMap.isEmpty()) {
        logger.config("failed to load address map, using defaults");
        addressMap.put("rfc822", "smtp");
    }
}
```
