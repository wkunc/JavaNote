# Test Execution Order

默认情况下, 将使用确定性但故意不明显的算法对测试方法进行排序.
这样可以保证测试套件的后续运行以相同的顺序执行测试方法.

虽然真正单元测试通常不应该依赖于它们执行的顺序, 但有时还是有
必须强制执行特定的测试方法执行顺序, 例如在编写集成测试或者功能测试时,
测试顺序是*重要的*. 尤其是与``@TestInstance(Lifecycle.RER_CLASS)``结合使用时


要控制执行测试方法的顺序, 请使用``@TestMethodOrder``注释测试类或测试接口.
并指定所需的``MethodOrderer``实现. 你可以实现自定义的``MethodOrderer``实现
也可以使用内置的的实现之一:

* Alphanumeric: 使用测试方法的名称和形式参数列表按字母数字顺序对其进行排序
* OrderAnnotation: 通过@Order注释指定的值对测试
* Random: 对测试方法进行伪随机排序, 支持自定义随机数种子设置.


```java
@API(status = EXPERIMENTAL, since = "5.4")
public interface MethodOrderer {

	void orderMethods(MethodOrdererContext context);

	default Optional<ExecutionMode> getDefaultExecutionMode() {
		return Optional.of(ExecutionMode.SAME_THREAD);
	}
    // 下面是内置的实现.

	class Alphanumeric implements MethodOrderer {

		@Override
		public void orderMethods(MethodOrdererContext context) {
			context.getMethodDescriptors().sort(comparator);
		}

		private static final Comparator<MethodDescriptor> comparator = Comparator.<MethodDescriptor, String> //
				comparing(descriptor -> descriptor.getMethod().getName())//
				.thenComparing(descriptor -> parameterList(descriptor.getMethod()));

		private static String parameterList(Method method) {
			return ClassUtils.nullSafeToString(method.getParameterTypes());
		}
	}

	class OrderAnnotation implements MethodOrderer {

		@Override
		public void orderMethods(MethodOrdererContext context) {
			context.getMethodDescriptors().sort(comparingInt(OrderAnnotation::getOrder));
		}

		private static int getOrder(MethodDescriptor descriptor) {
			return descriptor.findAnnotation(Order.class).map(Order::value).orElse(Integer.MAX_VALUE);
		}
	}

	class Random implements MethodOrderer {

		private static final Logger logger = LoggerFactory.getLogger(Random.class);

		private static final long DEFAULT_SEED;

		static {
			DEFAULT_SEED = System.nanoTime();
			logger.info(() -> "MethodOrderer.Random default seed: " + DEFAULT_SEED);
		}

		public static final String RANDOM_SEED_PROPERTY_NAME = "junit.jupiter.execution.order.random.seed";

		@Override
		public void orderMethods(MethodOrdererContext context) {
			Collections.shuffle(context.getMethodDescriptors(),
				new java.util.Random(getCustomSeed(context).orElse(DEFAULT_SEED)));
		}

		private Optional<Long> getCustomSeed(MethodOrdererContext context) {
			return context.getConfigurationParameter(RANDOM_SEED_PROPERTY_NAME).map(configurationParameter -> {
				Long seed = null;
				try {
					seed = Long.valueOf(configurationParameter);
					logger.config(
						() -> String.format("Using custom seed for configuration parameter [%s] with value [%s].",
							RANDOM_SEED_PROPERTY_NAME, configurationParameter));
				}
				catch (NumberFormatException ex) {
					logger.warn(ex,
						() -> String.format(
							"Failed to convert configuration parameter [%s] with value [%s] to a long. "
									+ "Using default seed [%s] as fallback.",
							RANDOM_SEED_PROPERTY_NAME, configurationParameter, DEFAULT_SEED));
				}
				return seed;
			});
		}
	}

}
```

# Test Instance Lifecycle
为了允许单独执行各个测试方法并避免由于可变的测试实例状态而导致的意外副作用,
JUnit在执行每种测试方法之前, 请为每个测试类创建一个新实例.
``PER-METHOD``测试实例生命周期是``JUnit Jupiter``中的默认行为, 类似于所有先前的JUnit版本.


# Nested Tests
