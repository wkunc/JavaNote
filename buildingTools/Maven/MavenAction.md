# 生命周期和插件

Maven 拥有三套相互独立的生命周期, 它们分别为Clean, default, site

1. clean 生命周期的目的是清理项目
2. default 生命周期的目的是构建项目
3. site 生命周期的目的是建立项目站点

## clean 生命周期

clean 生命周期包含三个阶段

1. pre-clean
2. clean
3. post-clean

## default 生命周期

default 定义了真正构建时所需要执行的所有步骤
它是所有生命周期中最核心的部分

1. validate
2. initialize
3. generate-sources
4. process-sources
5. generate-resources
6. process-resources
7. compile
8. process-classes
9. generate-test-sources
10. process-test-sources
11. generate-test-resources
12. process-test-resources
13. test-comppile
14. process-test-classes
15. test
16. prepare-package
17. package
18. pre-integration-test
19. integration-test
20. post-integration-test
21. verify
22. install
23. deploy

## site 生命周期

pre-site
site
post-site
site-deploy

# 插件目标

Maven 的核心仅仅定义了抽象的生命周期, 具体的任务是交给插件完成的.
插件以独立的构件形式存在, 因此, Maven 核心的分发包只有3MB的大小,
Maven 会在需要的适合下载并使用插件

对于插件本身, 为了能够复用代码, 它往往能够完成多个任务.
例如: maven-dependency-plugin, 它能够基于项目依赖做很多事情
它能够分析项目依赖找出无用依赖; 它能列出项目依赖树等待

为每个这样的功能编写独立的插件是不可取的, 因为这些任务背后有很多
可以复用的代码, 因此将这些功能聚集在一个插件里, 每个插件就是一个插件的目标

# 内置插件

maven-clean-plugin
maven-resources-plugin
maven-compiler-plugin
maven-resources-plugin
maven-compiler-plugin
maven-surefire-plugin
maven-jar-plugin
maven-install-plugin
maven-deploy-plugin

# Maven 多模块

# 聚合和继承

聚合就是多模块, 但是只是简单的在一个工程中配置了多个模块, 它们之间还没有联系
继承就是为了简化这些模块的 pom.xml 中的依赖,插件等
