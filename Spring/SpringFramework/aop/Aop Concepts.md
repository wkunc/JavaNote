# AOP Concepts
|name|description|
|---|----|
|Aspect(切面)| 方面: 跨越多个类基本的关注点的模块化. @Aspect
|Join point(连接点)|程序执行期间的一个点, 例如执行方法或处理异常. 在Spring AOP 中, 连接点始终表示方法执行
|Advice(通知)|特定连接点的某个方面采取的操作. 有不同的类型, 前置后置环绕等.
|Pointcut(切点)|匹配连接点的谓词, Advice 和切点表达式关联, 并在匹配的任何连接点处运行.
|Introduction|
|Target Obeject|
|AOP proxy|
|Weaving|

Spring 支持的通知类型(Advice): 
![](Advice.png)
1. BeforeAdvice
2. After returning
3. After throwing
4. After advice
5. Around Advice

总结起来就是, 前置, 后置, 和环绕. 前置中有两个子类型.

# 切点表达式

* execution
* within
* this
* target
* args
* @target
* @args
* @within
* @annotation

execution(<修饰符模式>? 返回值类型模式 方法名模式(参数模式) <异常模式>?)
? 0个或多个, 正则语法, 代表可选, 所以修饰符模式和异常模式都是可选的.
必须要有返回值类型模式, 方法名模式, 参数模式.
相当u
```java
```
