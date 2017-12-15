## 重构：独立可重用的 Verticle

> 提示：
>
> 相关的源代码可以在本手册（注：英文原版）的仓库 [step-2](https://github.com/vert-x3/vertx-guide-for-java-devs/tree/master/step-2) 目录下找到。

经过第一次迭代，我们得到了一个可以使用的 Wiki 应用。但它的实现之中仍有一些问题：

1. 处理 HTTP 请求的代码与访问数据库的代码交织在同一个方法之中
2. 许多配置数据（例如：端口号、JDBC 驱动等等）是以字符串的形式硬编码在代码之中
