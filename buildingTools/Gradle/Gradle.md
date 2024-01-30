# Projects and tasks
Gradle 中的所有内容都基于两个基本概念: `Project` and `Task`

每个`project`由一个或多个 `task` 组成. task 代表构建执行的一些原子性工作.
可能是编译某些类, 创建 JAR, 生成 Javadoc 或者将一些 archives 发布到仓库中.

project 不一定代表要build的事物, 它可能表示要完成的事情, 例如将应用程序部署到生产环境.

org.gradle.api.Project


This interface is the main API you use to interact with Gradle from your build file.
From a Project, you have programmatic access to all of Gradle's features.

这个接口是您用来从构建文件与Gradle交互的主要API.从项目中, 您可以以编程方式访问Gradle所有功能.

Lifecycle

There is a one-to-one relationship between a Project and a "build.gradle" file.
During Build initialisation, Gradle assembles a Project object for each project 
which is to participate in the build, as follows: 

* Create a org.gradle.api.initialization.Settings instance for the build.
* Evaluate the "settings.gradle" script, if present, against the org.gradle.api.initialization.Settings
object to configure it
* Use the configured org.gradle.api.initialization.Settings object to create the hierarchy of Project instances
* Finally, evaluate eache Project by executing its "build.gradle" file, if present, against the project.
The projects are evaluated in breadth-wise order, such that a project is evaluated in breadth-wise order,
such that a project is evaluated before its child projects. This order can be overridden by calling 
`evaluationDependsOnChildren()` or by adding an explicit evaluation dependency using evaluationDependsOn(String).


Tasks

A project is essentially a collection of Task objects.
Each task performs some basic piece of work, such as compiling classes, or running unit tests,
or zipping up a WAR file. You add tasks to a project using one of the create() methods on
`TaskContainner`, such as TaskContainner.create(String). You can locate existing tasks using one of the 
lookup methods on TaskContainer, such as org.gradle.api.tasks.TaskCollection.getByName(String).

Dependencies
