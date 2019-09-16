# 基于注解的AOP的使用


切点表达式

execution(signature)
> signature 是完整的方法签名即:
> 返回值类型的全类名 目标方法所属的全类名.方法名(参数类型的全类名)
> 允许使用通配符,
> execution(\* \*(..)): 就意味着匹配任何返回值, 任何方法名, 任何参数的方法,即匹配所有方法.
> execution(int \*(..)): 匹配返回值为 int 类型的任何方法.
> execution(\* set(..)): 匹配任何返回值, 名字叫set的方法.


1. Kinded : exection()
2. Scoping : within()
3. Context : this, target, @annotation
