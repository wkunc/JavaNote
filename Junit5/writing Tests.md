# Writing Tests

## Annotaions

| Annotation | Descrpition |
|------------|-------------|
|@Test| 一个标记注解, 类似JUnit4的@Test, 但是没有任何参数. 不同类型的测试, 会有专门的注解.
|@ParameterizedTest| 表示方法是一个参数化测试.
|@RepeatedTest| 表示方法是一个 *测试模板* 对于 *重复测试*.
|@TestFactory| 表示方法是一个 *测试模板* 对于 *动态测试*.
|@TestTemplate| 表示方法是一个 *template for test cases*.
|@TestMethodOrder| 用来配置, 测试方法的执行顺序.
|@TestInstance| 
|@DisplayName|
|@DisplayNameGeneration|
|@BeforeEach|
|@AfterEach|
|@BeforeAll|
|@AfterAll|
|@Nested|
|@Tag|
|@Disabled|
|@Timeout|
|@ExtendWith|
|@RegisterExtension|
|@TempDir|

## Test Class and methods (测试类与方法)

**Test Class**: 
任何 顶层class, 静态内部类class, or @Nested class 并且 至少包含一个测试方法.
测试类不能是抽象的, 并且只能有一个构造器.

**Test Method**:
任何被 @Test, @RepeatedTest, @ParameterizedTest, @TestFactory or @TestTemplate 标记的方法

**Lifecycle Method**: 任何被直接或间接标注: @BeforeAll, @AfterAll, @BeforeEach, @AfterEach


## Display Names(显示的名字)
测试类和测试方法都可以声明自定义显示的名称, 通过 @DisplaypName , 和 空格, 特殊字符, 甚至 emojis 表情.
它们将会显示在 test report 和 IDE

```java
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;

@DisplayName("A special test case")
class DisplayNameDemo {

    @Test
    @DisplayName("Custom test name containing spaces")
    void testWithDisplayNameContainingSpaces() {
    }

    @Test
    @DisplayName("╯°□°）╯")
    void testWithDisplayNameContainingSpecialCharacters() {
    }

    @Test
    @DisplayName("😱")
    void testWithDisplayNameContainingEmoji() {
    }
}
```

Display Name Generators



