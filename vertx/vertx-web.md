# Basic Vert.x-web concepts

Router 包含0个或多个Routes. Router是Vert.x-Web的核心抽象,

一个Router接收一个Http请求并找到该请求的第一个匹配的Route, 然后将请求传递到该路由.

一个Route可以有一个与之关联的Handler, 然后该处理程序接收请求, 然后对request做一些事情.
然后结束它或将它传递给下一个匹配的处理程序.
