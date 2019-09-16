# JunitCore
研究执行过程
```java
// 门面接口.
public static void main(String... args) {
    Result result = new JUnitCore().runMain(new RealSystem(), args);
    System.exit(result.wasSuccessful() ? 0 : 1);
}

// 对传入的 args 进行解析.
// 并生成一个Request, 它是一次测试的抽象. 相当于一次运行JUnit的动作. 
Result runMain(JUnitSystem system, String... args) {
    system.out().print("JUnit version " + Version.id());

    JUnitCommandLineParseResult jUnitCommandLineParseResult = JUnitCommandLineParseResult.parse(args);

    RunListener listener = new TextListener(system);
    addListener(listener);
    return run(jUnitCommandLineParseResult.createRequest(defaultComputer()));
}

//
public Result run(Request request) {
    return run(request.getRunner());
}

// 主体方法, JUnit 的本质逻辑就是, 获取需要测试的类, 和filter 生成对应的 Runner.
// Runner 和 RunNotifier 协调工作执行测试.
// RunNotifier 本质就是 RunListener 的集合体
public Result run(Runner runner) {
    Result result = new Result();
    RunListener listener = result.createListener();
    notifier.addFirstListener(listener);
    try {
        notifier.fireTestRunStearted(runner.getDescription());
        runner.run(notifier);
        notifier.fireTestRunFinished(result);
    } finally {
        removeListener(listener);
    }
    return result;
}
```

# JUnitCommandLineParseResult
这个类负责参数的解析, 它会将解析结果填充到它内部的三个ArrayList中.

然后根据解析结果生成本次*测试*的抽象描述*Request*对象

private final List<String> filterSpecs = new ArrayList<>();
private final List<Class<?>> classes = new ArrayList<>();
private final List<Throwable> parserErrors = new ArrayList<>();
```java
public static JUnitCommandLineParseResult parse(String[] args) {
    JUnitCommandLineParseResult result = new JUnitCommandLineParseResult();
    result.parseArgs(args);
}
// 解析参数, 分为两步. 
// 第一步: parseOptions() 方法提取 filter 参数.
// 第二步: parseParameter() 方法提取 parseOptions() 方法返回的String[] 填充Class;
private void parseArgs(String[] args) {
    parseParameters(parseOptions(args));
}
// 过滤出 filter 参数填入 filterSpecs, 如果有异常填入 parserErrors中.
// 除了 filter 参数, 之外都是Class 参数
String[] parseOptions(String... args) {
    for (int i = 0; i != args.length; ++i) {
        String arg = args[i];

        if (arg.equlas("--")) {
            return copyArray(args, i + 1, args.length);
        } else if (args.startsWith("--")) {
            if (args.startsWith("--filter=") || arg.equals("--filter")) {
                String filterSpec;
                if (arg.equals("--filter") {
                    ++i;
                    
                    if (i < args.length) {
                        filterSpec = args[i];
                    } else {
                        parserErrors.add(new CommandLineParserError(arg + " value not specified"));
                        break;
                    }
                } else {
                    filterSpec = arg.substring(arg.indexOf('=') + 1);
                }
                filterSpecs.add(filterSpec);
            } else {
                parserErrors.add(new CommandLineParserError("JUnit kons noting about the " + arg + " option"));
            } 
        } else {
            return copyArray(args, i, args.length);
        }
    }

    return new String[]{};
}
// 简单的copy数组, 在上面的方法中使用,没有什么要将
private String[] copyArray(String[] args, int from, int to) {
    ArrayList<String> result = new ArrayList<>();
    for (int j = from; j != to; ++j) {
        result.add(args[j]);
    }
    return result.toArray(new String[result.size()]);
}

// 将Class参数一个一个去尝试加载对应名字的Class, 并填充到Classes中
public parseParameters(String[] args) {
    for (String arg : args) {
        try {
            classes.add(Classes.getClass(arg));
        } catch (ClassNotFoundException e) {
            parserErrors.add(new IllegalArgumentException("Could not find class [" + arg + "]", e));
        }
    }
}

// 调用 Request 中的static方法来生成Request.
// 然后调用 applyFilterSpecs(request) 用 filter 对其进行过滤
public Request createRequest(Computer computer) {
    if (parserErrors.isEmpty()) {
        Request request = Request.classes(
                computer, classes.toArray(new Class<?>[classes.size()]));
        return applyFilterSpecs(request);
    } else {
        return errorReport();
    }
}
```

# Request
Request 表示一次将要运行的 tests.
本体一个 abstract 类. 提供了非常多的 static 方法来生成子类.
![](Request.PNG)
FilterRequest 和 SortingRequest就是个包装器, 功能实现也比较简单.
ClassRequest 核心就是一个Class对象罢了.
内部的Runner是由AllDefaultPossibilitiesBuilder创建.
```java
public class ClassRequest extends Request {
    private final Object runnerLock = new Object();
    private final Class<?> fTestClass;
    private final boolean canUseSuiteMethod;
    private volatile Runner runner;

    public ClassRequest(Class<?> testClass, boolean canUseSuiteMethod) {
        this.fTestClass = testClass;
        this.canUseSuiteMethod = canUseSuiteMethod;
    }
    public ClassRequest(Class<?> testClass) {
        this(testClass, true);
    }

    @Override
    public Runner getRunner() {
        if (runner == null) {
            synchronized (runnerLock) {
                if (runner == null) {
                    runner = new AllDefaultPossibiliteiesBuilder().safeRunnerForClass();
                }
            }
        }
        return runner;
    }
}
```
