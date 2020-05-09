# Conditional Test Execution (条件测试执行)

``ExecutionCondition``扩展 API 在JUnit Jupiter中允许开发者是否 enable 或者
disable 一个容器(一个测试类?)或者测试.

最简单的例子就是内建的``DisabledCondition``支持``@Disabled``注解.
其他基于注解的 conditions 包含在 ``org.juit.jupiter.api.condition``包中.

允许开发者声明式的启动或者不启动一个测试或一个测试类.

下面列出一些内建的条件化测试注解, 它们都是由内建的``ExecutionCondition``实现的.

* Operating System Conditions(当前允许的系统环境, 比如WINDOW, MAC, LINUX)
 1. @EnableOnOs: 
 2. @DisableOnOs: 
* Java Runtime Environment Conditions(当前允许的JVM版本, 如:JAVA_8, JAVA_10)
 1. @EnableOnJre:
 2. @DisableOnJre:
* System Property Conditions (JVM系统属性中是否包含指定的参数和值)
 1. @EnableIfSystemProperty:
 2. @DisableIfSystemProperty:
* Environment Variable Conditions: (是否包含指定的环境变量和值, 在当前操作系统中.)
 1. @EnabledIfEnvironmentVariable:
 2. @DisabledifEnvironmentVariable:
* Script-based Conditions: (可以使用任何 Java Script API 支持的脚本语言编写, 这个特性将在后续版本中移除)
 1. @EnableIf
 2. @DisabledIf
* @Disabled: 也是基于执行条件实现的
