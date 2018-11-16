# 基本过程 
定义一个测试类的要求是, 这个类必须是 public 的并且包含了一个 无参数的构造函数

Junit 在调用每一个 @Test 方法之前, 为测试类创建一个新的实例.
这有助于提供测试方法之间的独立性, 并且避免在测试代码中产生意外的副作用.
因为每一个方法都运行于一个新的测试类实例上, 所以我们就不能在测试方法之间重用各个实例变量值.

当你需要一次运行多个测试类时, 你就要创建另一个叫做测试集(test suite 或者 Suite)的对象.
你的测试集也是一个特定的测试运行器(Runner), 因此可以像运行测试类那样运行它.

Junit 中的核心抽象就是
* test class (一个包含或者多个测试的类)
* test suite (一组测试)
* test runner (执行测试集的程序)

# Runner
Junit4 的测试运行器
| 运行器 | 目的|
|-----|-----|
|org.junit.internal.runners.Junit38ClassRunner|用来兼容
|org.junit.runners.JUnit4|
|org.junit.runners.Parameterized|使用不同的参数集运行同一个测试集
|org.junit.runners.Suite|Suite 是一个包含不同测试的容器, 同时 Suite 也是一个运行器,可以运行一个测试类中所有以 @Test 注释的方法

可以用 **@RunWith** 注解指定 Runner 

## facade(门面)
为了能够尽可能快地运行测试, JUnit 提供了一个 facade (org.junit.runner.JUnitCore),
它可以运行任何测试运行器(Runner).
JUnit 设计这个facade 来执行你的测试, 并收集测试结果与统计信息

JUnit 的facade决定使用哪个Runner来运行你的测试, 它支持 JUnit3.8 的测试, JUnit4的测试,以及两者混合


# Sutie
运行多个测试类, 为了简化这个任务, JUnit 提供了测试 Suite. 
这个 Suite 是个容器, 用来把几个测试归在一起, 并且它们作为一个集合一起运行

@SuiteClasses 来组合多个测试类

不过IDE, Maven, Gradle 都提供了运行测试集的功能
