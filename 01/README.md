## 使用 Vert.x 实现一个最小可实施的 wiki 系统

* 提示：

    相关的源代码可以在本手册（注：英文原版）的仓库 [step-1](https://github.com/vert-x3/vertx-guide-for-java-devs/tree/master/step-1) 目录下找到。

我们将从第一个迭代版本开始，以尽可能简单的代码，使用 Vert.x 实现一个 wiki 系统。在之后的迭代版本，代码将会更加优雅，同时引入适当的测试，我们将可以看到使用 Vert.x 快速构建原型兼具简单性与实际性。

在当前阶段，这个 wiki 系统将会采用服务端渲染 HTML 的方式，通过 JDBC 连接实现数据持久化。为了实现这些，我们将会使用到以下库：

1. [Vert.x web](http://vertx.io/docs/vertx-web/java/) 同 Vert.x core 库（但其并未提供 API 处理路由、处理请求负载等等）一样支持创建 HTTP server。
2. [Vert.x JDBC 客户端](http://vertx.io/docs/vertx-jdbc-client/java/)用来提供基于 JDBC 的异步 API。
3. [Apache FreeMarker](http://freemarker.org/) 是一个简单的模板引擎，用来在服务端渲染页面。
4. [Txtmark](https://github.com/rjeschke/txtmark) 用来将 Markdown 文本渲染为 HTML，可以实现使用 Markdown 编辑 wiki 页面。
