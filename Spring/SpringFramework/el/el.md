# Spring EL

提供的功能

* Literal expressions (文字表达式)
* Boolean and relational operators (布尔和关系运算符)
* Regular expressions (正则表达式)
* Class expressions (类表达式)
* Accessing properties, arrays, lists, and maps (访问属性, 数组, list, map)
* Method invocation (方法调用)
* Relational operators (关系运算符)
* Assignment (分配)
* Calling constructors (调用构造器)
* Bean references (Bean 引用)
* Array construction (构造数组)
* Inline lists (内联list)
* Inline maps (内联map)
* Ternary operator (三元运算符)
* Variavles (变量)
* User-defined functions (用户定义函数)
* Collection projection (collection 投影)
* Collection selection (collection 选择)
* Templated expressions (模板化的表达)

# 从Root对象计算表达式

# 核心接口

org.springframework.expression
spel.support

* ExpressionParser : 表达式解析器, 将给定的SpringEL字符串(parseExpression方法), 生成一个Expression对象.
* Expression : 表达式抽象, 通过getValue()方法可以触发表达式的执行, 获得表达式结果
* EvaluationContext :

EvaluationContext 接口提供了属性, 方法, 字段解析器, 以及类型转换器.
默认实现类StandardEvluationContext的内部使用反射来操作对象.



