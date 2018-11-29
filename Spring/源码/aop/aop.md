# 创建切面
Spring 通过 org.springframework.aop.Pointcut 接口描述切点

Pointcut 由 ClassFilter 和 MethodMatcher 构成.
通过 ClassFilter 定位到某些特定类上, 通过 MethodMatcher 定位到特定方法上
这样 Pointcut 就拥有了描述某些类的某些特定方法的能力.

# 切点类型
Spring 定义了**6种**切点类型

1. 静态方法切点:
> org.springframework.aop.supprot.StaticMethodMatcherPointCut 是静态方法切点的抽象基类,
> 默认情况下它匹配所有的类. StaticMethodMatcherPointCut 包括两个主要子类,
> 分别是: NameMatchMethodPointcut 和 AbstractRegexpMethodPointcut
> 前者提供简单字符匹配方法签名, 而后者使用正则表达式匹配方法签名
2. 动态方法切点:
> org.springframework.aop.supprot.DynamicMethodMatcherPointcut 是动态方法切点的的抽象基类
> 默认情况下它匹配所有的类
3. 注解切点:
> org.springframework.aop.annotation.AnnotationMatchingPointcut 实现类表示注解切点.
> 使用 AnnotationMatchingPointcut 支持在 Bean 中直接通过 Java5.0 注解标签定义的切点
4. 表达式切点:
> org.springframework.aop.support.ExpressionPointcut 接口主要是为了支持 AspectJ 
> 切点表达式语法而定义的语法
5. 流程切点:
> org.springframework.aop.support.ControlFlowPointcut 实现类表示控制流程切点.
> ControlFlowPointcut 是一种特殊的切点. 它根据程序
6. 复合切点:

