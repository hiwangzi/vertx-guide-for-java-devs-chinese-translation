## 介绍

这本手册是关于使用 Vert.x 异步编程的易读介绍，针对那些熟悉主流“非异步” web 开发框架或者库（例如 Java EE、Spring）的开发者。

### 关于此手册

我们假定读者熟悉 Java 编程语言及相关生态。

我们将从一个 wiki web 程序开始（其使用关系型数据库并在服务端渲染网页），然后通过一步步的改进，使其最终成长为一个拥有“实时” web 特性的现代单页应用。在这个过程中，你将会学到：

1. 设计一个 web 程序，其在服务端通过模板渲染网页，并使用关系型数据库来持久化（存储）数据。

2. 清晰的抽离出每个技术组件，以作为可重复使用的事件处理单元（被称作 verticle）。

3. 不同的 verticle 之间（这些 verticle 使用同一个 JVM 进程或处于同一个集群下不同节点）可以互相无缝地通信交流，通过提取出 Vert.x 服务来优化这些 verticle 的设计。

4. 通过异步操作来测试代码。

5. 将暴露了 HTTP/JSON web API 的第三方服务融入进项目之中。

6. 实现一个 HTTP/JSON web API。

7. 使用 HTTPS，为浏览器会话产生的用户认证，为第三方应用访问提供的 JWT token，来实现对资源的保护及访问控制。

8. 借助流行的 RxJava 库和它在 Vert.x 中的集成来重构部分代码，以实现响应式编程。

9. 客户端采用 AngularJS 来实现单页应用。

10. 使用统一的集成于 SockJS 之上的 Vert.x event bus 机制来实现实时 web 项目。

* 注意：

    所有的文档及代码示例都可以在这里找到：https://github.com/vert-x3/vertx-guide-for-java-devs
    我们欢迎提供任何 issue reports，反馈及 pull-request！

### 什么是 Vert.x？

> Eclipse Vert.x 是一个可以在 JVM 上构建响应式应用的工具包。
>
> — Vert.x 官网

Eclipse Vert.x（以下简称 Vert.x） 是 Eclipse 基金会门下的一个开源项目，其最初是由 Tim Fox 在 2012 年发起的。

Vert.x 是一个工具包集合而不是一个框架：核心库为编写异步网络应用定义了基本的 API，你可以（自由地）为你的项目选择有用的模块（例如：数据库连接、监控、身份认证、日志、服务发现、集群支持等等）。Vert.x 基于 Netty 项目，Netty 是一个为 JVM 设计的高性能异步网络库。通常来说，使用 Vert.x 提供的高层次 API 会更有利于你（编写代码），相较原生 Netty 来说，也毫无性能损失。但如果你有所需要， Vert.x 同样允许你访问 Netty 的内部。

Vert.x 并不强制要求任何包或者构建环境。因为 Vert.x core 自身就是一个常规的 Jar（每一个 Jar 包含所有依赖）库，所以它可以作为 Jar 嵌入应用之中，甚至被部署到流行的组件和应用容器内。

Vert.x 被设计用于异步通信，因此相较于 Java servlets 或 java.net socket classes 这些同步 API 来说，可以使用较少的线程来解决更多的并发网络连接（问题）。Vert.x 对于多种类型的应用都非常有用：高性能消息/事件处理、微服务、API 网关、为移动应用设计的 HTTP API 等等。Vert.x 及其相关生态为构建端到端响应式应用提供了多种多样的技术工具。

虽然可能听起来 Vert.x 仅仅应用于高性能应用，但这篇导引手册可以证明 Vert.x 对于传统 web 应用同样非常有用。代码仍将保持简洁易理解，即便遇到突然的流量高峰，已经采用异步事件处理方式编写的代码可以轻松处理。

同时值得一提的是，Vert.x 支持多种流行的 JVM 语言：Java、Groovy、Scala、Kotlin、JavaScript、Ruby和Ceylon。Vert.x 支持多种语言的目标不只是提供各种语言 API 的访问，而是确保每一种语言都可以采用其特有的语言特性、以符合语言习惯的方式来调用 API（例如：使用 Scala 的 future 替换 Vert.x 的 future）。Vert.x 也可以很好的支持在一个应用之中，采用不同的 JVM 语言来开发不同的技术模块。

### Vert.x 核心概念

Vert.x 中有两个关键的概念：

1. 什么是 verticle
2. event bus 如何使不同的 verticle 之间通信

#### 线程与编程模型

很多网络库和框架依赖于一种简单的线程策略：为每一个客户端都分配一个线程用于连接，并且直到断开连接前，每个线程处理不同客户端的业务。Servlet 或者由 java.io 和 java.net 包编写的网络程序都是如此。虽然“同步 I/O”线程模型对于保持简单与易理解来说很有优势，但线程资源并不便宜，并且在高负载的情况下，操作系统会花费大量的时间浪费在线程调度管理上，因此对于大量并发请求来说，此模型扩展性较差。

Vert.x 中的可被部署的单位（或单元）被称作 Verticle。Verticle 基于「事件循环」来处理传入的事件，事件可以是接受网络缓冲、定时事件或者来自其他 verticle 的消息。在异步编程模型中，「事件循环」非常典型：

![event loop](https://github.com/zill057/vertx-guide-for-java-devs-chinese-translation/blob/master/intro/images/event-loop.png)

每一个事件都应在合理的时间内处理完成，不应让其阻塞「事件循环」。这意味着可能阻塞线程的操作不应当在「事件循环」中被执行，就像在图形界面处理事件时（不应当）卡住 Java 或者 Swing 界面去做一个很慢的网络请求一样。在本手册的后面将会看到，Vert.x 提供了一种在「事件循环」外处理阻塞操作的机制。当「事件循环」在处理一个事件耗时过长时，Vert.x 总会在日志中发出警告。为了匹配应用的需求（例如：运行环境是相对较慢的物联网 ARM 主板），这个特性同样也是可以被配置。

每一个「事件循环」都运行在线程上。默认情况下，每一个 CPU 核心线程运行 2 个「事件循环」。所以正常情况下，verticle 将始终在同一个线程上处理事件，因此无需使用线程协调机制来调整 verticle 的状态（例如：Java class fields）。

verticle 可以进行一些配置(configuration)（例如：证书、网络地址等等），并且可以被部署多次：

![verticle threading config](https://github.com/zill057/vertx-guide-for-java-devs-chinese-translation/blob/master/intro/images/verticle-threading-config.png)

传入的网络数据将会被接收线程收到，并作为事件交由相应的 verticle 处理。当一个 verticle 开启了网络服务器并被部署多次时，事件将会采用轮询形式被分发给 verticle 实例，这有利于处理大量并发网络请求时最大化 CPU 利用率。Verticle 的生命周期只有简单的开启与结束，并且 verticle 可以 deploy 其他 verticle。

#### Event bus

在 Vert.x 中，verticle 将代码组织成可部署的单元。Vert.x event bus 是不同 verticle 之间通过异步消息传递进行通信的主要工具。例如假设存在一个 verticle 负责 HTTP 请求，另一个 verticle 负责管理数据库访问。Event bus 将允许 HTTP verticle 发送请求给数据库 verticle 来执行 SQL 查询，并返回结果给 HTTP verticle。

![event bus](https://github.com/zill057/vertx-guide-for-java-devs-chinese-translation/blob/master/intro/images/event-bus.png)

因为 JSON 可以被各种语言所编写的 verticle 用来通信，同时也是非常流行的通用型半结构化数据编组格式，所以其被推荐用来作为信息的交换格式，但 event bus 本身并不限制传递的数据类型。

消息可以发送到可接收字符串的目的地。 Event bus 支持以下沟通模式：

1. 点对点消息
2. 请求-响应消息
3. 发布-订阅消息（广播）

即使不在同一个 JVM 进程中，event bus 也可以使 verticle 之间透明的沟通：

* 当网络集群被激活时，event bus 即是分布式的，因此消息可以被发送到运行于另一个应用节点的 verticle 上
* 为了与其他第三方应用沟通（通信），可以通过简单的 TCP 协议访问 event bus
* Event bus 也可以被设置为通用的消息桥梁（例如：AMQP、Stomp）
* SockJS bridge 允许 web 应用就像其他 verticle 一样，基于 event bus 在运行于浏览器中的 JavaScript 无缝地接收和发布消息