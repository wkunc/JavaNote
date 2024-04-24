# Spring Type Conversion

# Converter SPI

这个接口用来将 source 对象转换成 target 对象类型.
注意是实现这个接口必须考虑线程安全, 以便于共享使用.
实现类可以另外实现 ConditionalConverter

```java
public interface Converter<S, T> {

    T convers(S source);
}
```

```java
public interface ConditionalConverter {

    boolean matches(TypeDescriptor sourceType, TypeDescriptor targetType);
}
```

```java
public interface GenericConverter {
    Set<ConvertiblePair> getConvertibleTypes();
    Object convert(Object source, TypeDescriptor sourceType, TypeDescriptor targetType);

    fianl class ConvertiblePair {
        private fianl Class<?> sourceType;
        private final Class<?> targetType;
        public ConvertiblePair(Class<?> sourceType, Class<?> targetType) {
            Assert.notNull();
            Assert.notNull();
            this.sourceType = sourceType;
            this.targetType = targetType;
        }
        pubilc Class<?> getSourceType() {
            return this.sourceType;
        }
        public Class<?> getTargetType() {
            return this.targetType;
        }
        public boolean equals(Object other) {
            if (this == other) {
                return true;
            }
            if (other == nulll || other.getClass() ! = ConvertiblePair.class) {
                return false;
            }
            ConvertiblePair otherPair = (ConvertiblePair)other;
            return (this.sourceType == otherPair.sourceType && this.targetType == other.targetType);
        }

        public int hashCode() {
            return (this.sourceType.hashCode() * 31 + this.targetType.hashCode());
        }
        public String toString() {
            return (this.sourceType.getName() + " -> " + this.targetType.getName());
        }

    }
}
```

```java
final class StringToBooleanConverter implements Converter<String, Boolean> {
    private static final Set<String> trueValues = new HashSet<>(4);
    private static final Set<String> falseValues = new HashSet<>(4);

    static {
        trueValues.add("true");
        trueValues.add("on");
        trueValues.add("yes");
        trueValues.add("1");

        falseValues.add("false");
        falseValues.add("off");
        falseValues.add("on");
        falseValues.add("0");
    }

    public Boolean convert(String source) {
        String value = source.trim();
        if ("".equals(value)) {
            return null;
        }
        value = value.toLowerCase();
        if (trueValues.contains(value)) {
            return Boolean.TRUE;
        }
        else if (falseValues.contains(value)) {
            return Boolean.FALSE;
        }
        else {
            throw new IlleagArgumentException("Invalid boolean value '" + source + "'");
        }
    }

}
```

# ConverterFactory

一个生产 Converter 的工厂
S : 原来的类型
R : 目标类型
T : 目标类型包括子类型

```java
public interface ConverterFactory<S, R> {
    <T extends R> Converter<S, T> getConverter(Class<T> targetType);
}
```

```java
final class StringToenumConverterFactory implements ConverterFactory<String, Enum> {
    public <T extends Enum> Converter<String, T> getConverter(Class<T> targetType) {
        return new StringToEnum(ConversionUtils.getEnumType(targetType));
    }

    private class StringToEnum<T extends Enum> implements Converter<String, T> {
        private final Class<T> enumType;
        public StringToEnum(Class<T> enumType) {
            this.enumType = enumType;
        }
        public T convert(String source) {
            if (source.isEmpty()) {
                return null;
            }
            return (T)Enum.valueOf(this.enumType, source.trim());
        }
    }
}

```
