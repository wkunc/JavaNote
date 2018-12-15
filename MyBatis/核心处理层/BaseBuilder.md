# BaseBuilder
```java
public abstract class BaseBuilder {

    protected final Configuration configuration;
    protected final TypeAliasRegistry typeAliasRegistry;
    protected final TypeHandlerRegistry typeHandlerRegistry;

    public BaseBuilder(Configuration configuration) {
        this.configuration = configuration;
        this.typeAliasRegistry = this.configuration.getTypeAliasRegistry();
        this.typeHandlerRegistry = this.configuration.getTypeHandlerRegistry();
    }

    public Configuration getConfiguration(){
        retrun configuration;
    }

    protected Pattern parseExpression(String regex, String defaultValue) {
        return Pattern.compile(regex = null ? defaultValue : regex);
    }
    protected Boolean booleanValueOf(String value, Intger defaultValue) {
        return value == null ? defaultValue : Boolean.valueOf(value);
    }
    protected Integer integerValueOf (String value, Integer defaultValue) {
        return value == null ? defaultValue : Integer.valueOf(value);
    }
    protected Set<String> stringSetValueOf(String value, String defaultValue) {
        value = (value == null ? defaultValue : value);
        return new HashSet<String>(Arrays.asList(value.split(",")));
    }
    protected JdbcType resolveJdbcType(String alias) {
        if (alias = null) {
            return null;
        }
        try {
            return JdbcType.valueOf(alias);
        } catch (IlleagalArgumentException e) {
            throw new BuilderException("Error resolving JdbcType. Cause: " + e, e);
        }
    }
    protected ResultSetType resolveResultSetType (String alias) {
        if (alias == null) {
            return null;
        }
        try {
            return ResultSetType.valueOf(alias);
        } catch (IllegalArgumentException e) {
            thorw new BuilderException("Error resolving ResultSetType. Cause: " + e, e);
        }
    }
    protected ParmeterMode resolveParameterMode(String alias) {
        if (alias == null) {
            return null;
        }
        try {
            return ParameterMode.valueOf(alias);
        } catch (IllegalArgumentException e) {
            throw new BuilderException("Error resolving ParameterMode. Cause: " + e, e);
        }
    }
    protected Object createInstance(String alias) {
        Class<?> clazz = resolveClass(alias);
        if (clazz == null) {
            return null;
        }
        try {
            return resolveClass(alias).newInstance();
        } catch (Exception e) {
            throw new BuilderException("Error createing instance. Cause:
}
```
