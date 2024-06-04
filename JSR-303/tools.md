# 扩展实现bean valiation, 支持用SpringEL进行跨字段复杂逻辑校验

自己实现JSR303的自定义验证器, 利用SpringEL进行复杂校验逻辑
PS: 缺点IDEA不认识自定义的注解, 无法为校验脚本编写提供代码提示
```java
import javax.validation.Constraint;
import javax.validation.Payload;
import java.lang.annotation.Documented;
import java.lang.annotation.Repeatable;
import java.lang.annotation.Retention;
import java.lang.annotation.Target;

import static java.lang.annotation.ElementType.FIELD;
import static java.lang.annotation.ElementType.TYPE;
import static java.lang.annotation.RetentionPolicy.RUNTIME;

/**
 * 采用SpringEL实现脚本校验逻辑.
 * 可以用于类或者字段上.
 * SpringEL的Root对象取决于注解的位置, 类上时root是整个对象, 字段上时root是对应的字段
 *
 * 参考 {@link org.hibernate.validator.constraints.ScriptAssert} 功能实现
 */
@Documented
@Constraint(validatedBy = { SpELConstraintValidator.class })
@Target({ TYPE, FIELD })
@Retention(RUNTIME)
@Repeatable(SpEL.List.class)
public @interface SpEL {

    String message() default "{cec.pams.validator.SpEL.message}";

    Class<?>[] groups() default { };

    Class<? extends Payload>[] payload() default { };

    /**
     *
     * @return The Spring expression script to be executed. The script must return
     *         {@code Boolean.TRUE}, if the annotated element could
     *         successfully be validated, otherwise {@code Boolean.FALSE}.
     *         Returning null or any type other than Boolean will cause a
     *         {@link javax.validation.ConstraintDeclarationException} upon validation. Any
     *         exception occurring during script evaluation will be wrapped into
     *         a ConstraintDeclarationException, too. Within the script, the
     *         validated object can be accessed from the {@link javax.script.ScriptContext
     *         script context} using the name specified in the
     *         {@code alias} attribute.
     */
    String script();

    /**
     * @return The name of the property for which you would like to report a validation error.
     * If given, the resulting constraint violation will be reported on the specified property.
     * If not given, the constraint violation will be reported on the annotated bean.
     *
     * @since 5.4
     */
    String reportOn() default "";

    /**
     * Defines several {@code @ScriptAssert} annotations on the same element.
     */
    @Target({ TYPE })
    @Retention(RUNTIME)
    @Documented
    public @interface List {
        SpEL[] value();
    }

}


@Component
public class ValidatorExpressionEvaluator extends CachedExpressionEvaluator {

    private final Map<ExpressionKey, Expression> scriptCache = new ConcurrentHashMap<>(64);

    public EvaluationContext createEvaluationContext(Object root, @Nullable BeanFactory beanFactory) {
        StandardEvaluationContext evaluationContext = new StandardEvaluationContext(root);
        if (beanFactory != null) {
            evaluationContext.setBeanResolver(new BeanFactoryResolver(beanFactory));
        }

        return evaluationContext;
    }

    public Boolean executeValidateScript(String keyExpression, AnnotatedElementKey methodKey, EvaluationContext evalContext) {
        return getExpression(this.scriptCache, methodKey, keyExpression).getValue(evalContext, Boolean.class);
    }

}

```

Example
```java

@SpEL(groups = ValiInsert.class, script = "(joinDate != null and regularDate != null ) ? T(java.time.Period).between(joinDate, regularDate).getYears() >= 1 : true", reportOn = "regularDate", message = "满1年后才能转正")
@SpEL(groups = ValiInsert.class, script = "(leaveDate != null and joinDate != null ) ? !joinDate.isAfter(leaveDate) : true", reportOn = "leaveDate", message = "离开时间应大于等于入党时间")
@Getter
@Setter
public class Member {

    private String userId;

    @NotBlank(groups = {ValiInsert.class}, message = "关联人员id不能为空")
    @ApiModelProperty("人员id")
    private String personId;

    @PastOrPresent(groups = {ValiInsert.class, ValiUpdate.class}, message = "入党日期不能超过当前日期")
    @NotNull(groups = ValiInsert.class, message = "入党日期不为空")
    @ApiModelProperty("入党日期")
    private LocalDate joinDate;

    @PastOrPresent(groups = {ValiInsert.class, ValiUpdate.class}, message = "转正日期不能超过当前日期")
    @ApiModelProperty("转正日期")
    private LocalDate regularDate;

    @NotNull(groups = ValiInsert.class, message = "党籍状态不为空")
    @ApiModelProperty("党籍状态")
    private MemberStatus status;

}

```
