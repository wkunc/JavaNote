# The IoC contatiner
org.springframework.beans 和 org.springframework.context 包是 Spring IoC 容器的基础

BeanFactory 接口提供了一种能够管理任何对象的高级配置机制

ApplicationContext 是 BeanFactory 的子接口 它补充了 "更容易集成 Spring'AOP, 消息资源处理(用于国际化),
事件发布, 特定于应用程序的上下文

简而言之, BeanFactory 提供了配置框架和基本功能而 ApplicationContext 添加了更多的特定于企业的功能

ApplicationContext 是 BeanFactory 的完整超集

# Container Overview(大纲)

## Configuration Metadata(元数据)
Configuration metadata 代表你如何告诉Spring 容器在应用程序中 instantiate(实例化), configure (配置)
和 assemble(组装对象)

Spring IoC contanier 本身完全与与实际编写此配置元数据的格式分离


# Bean Overview

## Instantiating Beans
A bean definition is essentially a recipe for creating one or more object

### Instantiation with a Constructor

### Instantiation with a Static Factory Method

