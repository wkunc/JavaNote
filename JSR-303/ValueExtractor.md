# ValueExtractor Definition

验证容器元素的约束以及通用容器类型的级联验证时, 需要访问存储在容器中的值.
存储在容器中的值的检索通过 ValueExtractor 接口实现来处理.

```java
// 值提取器.
// 泛型类型容器的值提取器(如 List, Map) 与 T 的一个特定参数类型绑定.
public interface ValueExtractor<T> {

    void extractValues(T originalValue, ValueReceiver receiver);

    interface ValueReceiver {

        void value(String nodeName, Object object);

        void iterableValue(String nodeName, Object object);

        void indexedValue(String nodeName, int i, Object object);

        void keyedValue(String nodeName, Object key, Object object);
    }
}
```
