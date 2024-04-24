# TODO 奇怪的Class No tFound

起因: 底层库修改了依赖版本, 导致其代码中使用的类的确不存在与当前项目的依赖中.

奇怪的现象:
> jar包启动时会立刻抛出 `ClassNotFoundException`异常,
> 通过IDEA启动的SpringBoot时不会提示, 只有触发了这个Executor的拒绝策略时才会提示`ClassNotFoundException`

## 原因代码

```java
import cec.common.exception.CecServerException;
// 其余import ....

@Configuration
public class CecJobAutoConfiguration {

    @Bean("bizThreadPool")
    @ConditionalOnMissingBean(name = "bizThreadPool")
    ThreadPoolExecutor threadPoolExecutor() {
        return new ThreadPoolExecutor(
                0,
                200,
                60L,
                TimeUnit.SECONDS,
                new LinkedBlockingQueue<>(2000),
                r -> new Thread(r, "xxl-rpc, EmbedServer bizThreadPool-" + r.hashCode()),
                (r, executor) -> {
                // 这个 CecServerException 类不存在
                    throw new CecServerException("xxl-job, EmbedServer bizThreadPool is EXHAUSTED!");
                });
    }

}
```

## 思考

1. 一开始思考类加载的关系, 以为是`import`语句会导致Class被加载.
   查阅java类加载相关内容后, 很显然`import`并不会导致Class被加载

> Java Language Specification 12.4
> 1. T is a class and an instance of T is created. (创建一个Class的实例时先会加载对应的Class)
> 2. A static method declared by T is invoked. (调用类声明的静态方法时会加载对应的Class, 注意是会加载声明方法的类,
     如果通过子类调用父类声明的静态方法,那么被加载的会是父类)
> 3. A static field declared by T is assigned. (给类的静态字段赋值)
> 4. A static field declared by T is used and the field is not a constant variable (§4.12.4). (访问类的静态字段,
     注意是 `static` 不能带有 `final`, `static final`不会触发类的加载)
> 5. 还有就是加载类时, 如果父类没有加载那就会加载父类, 以及相关的接口. 而加载`interface`时不会加载`super interface`
> 6. 反射相关方法比如 `Class.forName()` 等也会导致类加载

2. 根据上面知道的类加载知识, 发现问题代码里符合条件的也就是只有下面代码符合第一条, 创建类实例导致类加载.

```java
(r, executor) -> {
    throw new CecServerException("xxl-job, EmbedServer bizThreadPool is EXHAUSTED!");
}
```

正好符合本地启动后不报错, 触发拒绝策略后提示`ClassNotFoundException`的行为.

3. 继续思考为啥jar包启动会报错, 而本地启动不报错.
   感觉代码没啥问题, 但是想到之前看到的`Java Lambda expression`的实现不是简单的匿名内部类语法糖, 而是做了优化的.
   所以开始从`Java Lambda expression`和匿名内部类的区别入手

```java
class A {
    public ThreadPoolExecutor test() {
        return new ThreadPoolExecutor(
                0,
                10,
                60L,
                TimeUnit.SECONDS,
                new LinkedBlockingQueue<>(200),
                r -> new Thread(r, "xxl-rpc, EmbedServer bizThreadPool-" + r.hashCode()),
                (r, executor) -> {
                    throw new CecException("xxl-job, EmbedServer bizThreadPool is EXHAUSTED!");
                }
//                new RejectedExecutionHandler() {
//                    @Override
//                    public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
//                        throw new CecException("xxl-job, EmbedServer bizThreadPool is EXHAUSTED!");
//                    }
//                }
        );
    }
}

public class Test {
    public static void main(String[] args) throws ClassNotFoundException {
        Class<?> clazz = A.class;
        // Spring中是通过 CecJobAutoConfiguration 的 class 对象获取声明的方法(用来分析那些是Bean方法)时报错
        // 所以这里的逻辑也是调用反射获取声明的方法
        Arrays.stream(clazz.getDeclaredMethods()).forEach(System.out::println);
    }
}
```

测试发现, 采用匿名内部类时运行不会报错.
而采用lambda表达式时运行就会抛出`ClassNotFoundException`.

### lambda 的实现

简单的根据搜索的内容理解, 就是lambda的实现没采用的匿名内部类的原因是防止类文件膨胀.
总所周知, 匿名内部类变异后会产生一个`A$1.class`的文件.
也就是匿名内部类越多`.class`文件就越多, 如此就增加了类加载的工作量, 会影响启动速度.
所以java把lambda变成一个私有的静态方法

```
private static void cec.demo.A.lambda$test$1(java.lang.Runnable,java.util.concurrent.ThreadPoolExecutor)
private static java.lang.Thread cec.demo.A.lambda$test$0(java.lang.Runnable)
public java.util.concurrent.ThreadPoolExecutor cec.demo.A.test()

```

查看正常运行时的输出的结果. 很容易发现, 除了A.test() 方法以外. class 中还有两个私有静态方法
正好分别对应`r -> new Thread(r, "xxl-rpc, EmbedServer bizThreadPool-" + r.hashCode())`
和`(r, executor) -> { throw new CecException("xxl-job, EmbedServer bizThreadPool is EXHAUSTED!"); }`
两个lambda表达式.

## 总结

-XX:TieredStopAtLevel=1 -noverify
IDEA启动和jar包启动行为不同的关键参数
