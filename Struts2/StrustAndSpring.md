# 集成Spring
要使用一个 struts-spirng 的包

## spring 管理 Action 的创建
集成 spring 的要点 Action 的 scope 必须是 prototype

struts 配置文件中 action 的 class 不再是显示的全类名, 而是 Spring 中的 Bean id

## Struts 管理 Action 的创建
使用 Spring 的自动注入来将Action依赖的服务注入, 主要注入数据操作类依赖
