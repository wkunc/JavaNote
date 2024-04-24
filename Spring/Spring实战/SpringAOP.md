# AOP(面向切片编程)

1. 术语

   |名词|解释
          |---|---
   |通知(Advice)|具体事务(何事,何时)
   |连接点(join point)|可以切入的点
   |切点(point cut)|准备切入的连接点(何处)
   |切面(Aspect)|切点和通知的集合
   |引入(Introducion)|
   |织入(Weaving)|

2. 在代码中的体现

   使用这些注释必须开启自动代理工能
   在javaConfig类上用 @EnableAspectJAutoProxy
   Aspect是一个类用@Aspect标注
   Advice是类中被标注的方法,根据执行时间不同可分为5种,
   (1.@Before,
   2.@After,
   3.AfterThrowing,
   4.AfterReturning,
   5.@Around)

   point cut是通知标注中的表达式如:@Before("execution (** concert.Performance.perform(..))")
   如果这个切点被多个通知使用也可以单独写出来如:
   @pointcut("execution (** concert.Performance.perform(..))")
   public void Performance(){}

# Spring 切点表达式

Spring 仅支持 AspectJ 切点指示器的一个子集

| AspectJ指示器  | 描述
|-------------|-------------------------------------------------------
| arg()       | 限制连接点匹配参数为指定类型的执行方法
| @args()     | 限制连接点匹配参数由指定注解的执行方法
| execution() | 用于匹配连接的执行方法
| this()      | 限制连接点匹配 AOP 代理的 bean 引用为指定类型的类
| target      | 限制连接点匹配目标对象为指定类型的类
| @target     | 限制连接点匹配特定的执行对象, 这些对象对应的类要具有指定类型的注解
| within()    | 限制连接点匹配指定类型
| @within()   | 限制连接点匹配指定注解锁标注的类型(当使用 Spring AOP 时,方法定义在由指定的注解所标注的类里)
| @annotation | 限定匹配带有指定注解的连接点

这些指示器中只有 execution() 是实际执行匹配的, 其他指示器都是用来限制所匹配的连接点的, 所以 execution() 是我们编写切点时
最主要使用的**指示器**
