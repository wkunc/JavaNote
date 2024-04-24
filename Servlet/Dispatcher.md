# 分派请求

构建 Web 应用时, 把请求转发给另一个 servlet 处理, 或在 response 中包含另一个 servlet 的输出通常是很有用的.
RequestDispatcher 接口的对象提供了一个机制来实现这种功能

当请求启用异步处理时, AsyncContext 允许用户将这个请求转发到 servlet 容器

# 获得 RequestDispatcher

实现了 RequestDispatcher 接口的对象, 可以从 ServletContext 中的下面方法得到:
getRequestDispatcher()
getNamedDispatcher()

