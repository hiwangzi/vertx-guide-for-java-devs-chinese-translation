# vertx-guide-for-java-devs-chinese-translation

这是 Vert.x 官方文档 [A gentle guide to asynchronous programming with Eclipse Vert.x for Java developers](http://vertx.io/docs/guide-for-java-devs) 非官方中文翻译。

其英文原版在 GitHub 上的位置： https://github.com/vert-x3/vertx-guide-for-java-devs

翻译不当之处，欢迎指正！

## 目录

* [00. 介绍](00/README.md)
  * [关于此手册](00/README.md#关于此手册)
  * [什么是 Vert.x？](00/README.md#什么是-vertx)
  * [Vert.x 核心概念](00/README.md#vertx-核心概念)

* [01. 使用 Vert.x 实现一个最小可实施的 wiki 系统](01/README.md)
  * [开始一个 Maven 项目](01/README.md#开始一个-maven-项目)
  * [添加需要的依赖](01/README.md#添加需要的依赖)
  * [剖析 verticle](01/README.md#剖析-verticle)
  * [A word on Vert.x future objects and callbacks](01/README.md#a-word-on-vertx-future-objects-and-callbacks)
  * [Wiki verticle 初始化的步骤](01/README.md#wiki-verticle-初始化的步骤)
  * [HTTP router handlers](01/README.md#http-router-handlers)
  * [运行我们的应用](01/README.md#运行我们的应用)

* [02. 重构：独立可重用的 Verticle](02/README.md)
  * [架构与技术选择](02/README.md#架构与技术选择)
  * [HTTP 服务器 verticle](02/README.md#http-服务器-verticle)
  * [数据库 verticle](02/README.md#数据库-verticle)
  * [在 main verticle 中部署 verticles](
    02/README.md#在-main-verticle-中部署-verticles)

* [03. 再重构：Vert.x 服务](03/README.md)
  * [调整 Maven 配置](03/README.md#调整-maven-配置)
  * [数据库服务接口](03/README.md#数据库服务接口)
  * [数据库服务实现](03/README.md#数据库服务实现)

* [04. 测试 Vert.x 代码](04/README.md)
  * [开始](04/README.md#开始)
  * [测试数据库操作](04/README.md#测试数据库操作)
