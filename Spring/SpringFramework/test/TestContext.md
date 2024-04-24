# TestContext

Spring TextContext Framework (位于org.springframework.test.context包中)
提供了通用的, 注解驱动的单元测试和集成测试支持, 它与使用中的测试框架无关.
TestContext框架非常重视约定优于配置,
合理的默认值可以通过基于注解的配置覆盖.

除了通用测试基础结构之外, TestContext框架还为JUnit4, JUnit 5 和TestNG
提供了显式支持.对于JUnit4和TestNG, Spring 提供抽象支持类.
此外Spring为 JUnit4 提供了一个自定义JUnit Runner和自定义JUnit规则.

## Key Abstractions (关键抽象)

框架的核心包括TestContextManager类和TestContext,
TestExcutionListener, SmartContextLoader 接口.

为每个测试类创建一个TestContextManager
(例如, 用于在JUnit5中的单个测试类中执行所有测试方法).
反过来, TestContextManager管理一个包含当前测试上下文的TestContext.
TestContextManager还会在测试进行时更新TestContext的状态,
并委托给TestExcutionListener实现, 这些实现提供依赖注入, 管理事务等来检测
实际的测试执行.
SmartContextLoader负责给为给定的测试类加载ApplicationContext.

*TestContext*封装了执行测试的上下文(与使用中的实际测试框架无关),
并为其负责的测试实例提供 **上下文管理和缓存支持**
TestContext还委托SmartContextLoader在需要时加载ApplicationContext.

*TestContextManager*是Spring TestContext Framework 的主要入口点,
负责管理单个 TestContext 并在明确定义的测试执行点向每个注册的
TestExceutionListener 发送事件.

* @BeforeClass 方法之前
* 测试实例化之后
* @Before 方法之前
* 在执行@Test 测试方法之前, 但在测试设置之后
* 在执行@Test 测试方法之后, 但在测试设置之后
* @After 方法之后
* @AfterClass 方法之后


