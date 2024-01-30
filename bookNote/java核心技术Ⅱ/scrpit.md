# Script(脚本）
## 获取脚本引擎
脚本引擎是一个可以执行用某种特定语言编写的脚本的类库.
当虚拟机启动时, 它会发现可用的脚本引擎.
为了枚举这些引擎, 需要构造一个 ScriptEngineManager, 
并调用 getEngineFactories 方法. 可以向每个引擎工厂询问
它们所支持的引擎名, MIME 类型和文件扩展名

# 编译器 API

```java
JavaCompiler compiler = ToolProvider.getSystemJavaCompiler();
OutputStream outStream = ...;
OutputStream errStream = ...;
int result = compiler.run(null,outStream,errStream,"-sourcepath","src","Test.java");
```

## 使用编译工具
可以通过使用 CompilationTask 对象来对编译过程进行更多的控制.特别是, 你可以:
* 控制程序代码的来源, 例如, 在字符串构建器而不是文件中提供代码.
* 控制类文件的放置位置, 例如, 存储在数据库中.
* 监听在编译过程中产生的错误和警告
* 在后台运行编译器

