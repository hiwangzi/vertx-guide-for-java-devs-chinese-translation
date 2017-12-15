## 使用 Vert.x 实现一个最小可实施的 wiki 系统

> 提示：
>
> 相关的源代码可以在本手册（注：英文原版）的仓库 [step-1](https://github.com/vert-x3/vertx-guide-for-java-devs/tree/master/step-1) 目录下找到。

我们将从第一个迭代版本开始，以尽可能简单的代码，使用 Vert.x 实现一个 wiki 系统。在之后的迭代版本，代码将会更加优雅，同时引入适当的测试，我们将可以看到使用 Vert.x 快速构建原型兼具简单性与实际性。

在当前阶段，这个 wiki 系统将会采用服务端渲染 HTML 的方式，通过 JDBC 连接实现数据持久化。为了实现这些，我们将会使用到以下库：

1. [Vert.x web](http://vertx.io/docs/vertx-web/java/) 同 Vert.x core 库（但其并未提供 API 处理路由、处理请求负载等等）一样支持创建 HTTP server。
2. [Vert.x JDBC 客户端](http://vertx.io/docs/vertx-jdbc-client/java/)用来提供基于 JDBC 的异步 API。
3. [Apache FreeMarker](http://freemarker.org/) 是一个简单的模板引擎，用来在服务端渲染页面。
4. [Txtmark](https://github.com/rjeschke/txtmark) 用来将 Markdown 文本渲染为 HTML，可以实现使用 Markdown 编辑 wiki 页面。

### 开始一个 Maven 项目

本手册使用 [Apache Maven](https://maven.apache.org/) 作为构建工具，主要是因为其比较好的集成在了大多数集成开发环境。你也可以选择使用其他的构建工具（例如：[Gradle](https://gradle.org/)）。

Vert.x 社区提供了一个项目结构模板，可以在[这里](https://github.com/vert-x3/vertx-maven-starter)获取。如果你选择使用 Git 作为版本控制系统的话，（搭建起项目的）最快方式就是克隆这个仓库，删掉它的 ```.git/``` 目录，重新创建为一个新的 Git 仓库。

```
git clone https://github.com/vert-x3/vertx-maven-starter.git vertx-wiki
cd vertx-wiki
rm -rf .git
git init
```

这个项目提供了一个示例 verticle 和一个单元测试。删除 ```src/``` 目录下所有的 ```.java``` 文件来自定义（hack） wiki 项目是安全的，但在此之间，可以先尝试构建项目，并测试是否可以运行：

```
mvn package exec:java
```

Maven 项目的 ```pom.xml``` 做了两件有趣的事：

1. 它使用 [Maven Shade](https://maven.apache.org/plugins/maven-shade-plugin/) 插件创建一个包含所有需要的依赖的 Jar 打包文件（也被叫做“a fat Jar”），其后缀为 ```-fat.jar```
2. 它使用 [Exec Maven](http://www.mojohaus.org/exec-maven-plugin/) 插件来提供 ```exec:java``` 以用于通过 Vert.x ```io.vertx.core.Launcher``` 类来依次启动应用。这等价于通过 ```vertx``` 命令行工具（在 Vert.x 分布节点中传送命令）来运行（项目）。

在代码变更之后，你可以选择使用 ```redeploy.sh``` 和 ```redeploy.bat``` 脚本来自动编译和重新部署（项目）。但要注意，使用这些脚本需要确保脚本中的 ```VERTICLE``` 变量与 main verticle 中实际用到的一样。

> 注意：
>
> 另外，Fabric8 项目下有一个 [Vert.x Maven plugin](https://vmp.fabric8.io/)。它用于初始化、构建、打包并运行一个 Vert.x 项目。
> 生成一个与克隆自 Git starter 仓库相似的项目：
> ```
> mkdir vertx-wiki
> cd vertx-wiki
> mvn io.fabric8:vertx-maven-plugin:1.0.7:setup -DvertxVersion=3.5.0
> git init
> ```

### 添加需要的依赖

首先在 Maven ```pom.xml``` 文件中添加用于 web 处理和渲染的依赖：

```xml
<dependency>
  <groupId>io.vertx</groupId>
  <artifactId>vertx-web</artifactId>
</dependency>
<dependency>
  <groupId>io.vertx</groupId>
  <artifactId>vertx-web-templ-freemarker</artifactId>
</dependency>
<dependency>
  <groupId>com.github.rjeschke</groupId>
  <artifactId>txtmark</artifactId>
  <version>0.13</version>
</dependency>
```

> 提示：
>
> 正如 ```vertx-web-templ-freemarker``` 名字所表示的那样，对于流行的模板引擎，Vert.x web 提供了插件式的支持：Handlebars、Jade、MVEL、Pebble、Thymeleaf 以及 Freemarker。

然后添加 JDBC 数据访问相关的依赖：

```xml
<dependency>
  <groupId>io.vertx</groupId>
  <artifactId>vertx-jdbc-client</artifactId>
</dependency>
<dependency>
  <groupId>org.hsqldb</groupId>
  <artifactId>hsqldb</artifactId>
  <version>2.3.4</version>
</dependency>
```

Vert.x JDBC client 库可以支持任何 JDBC-兼容 数据库的访问。自然而然，在我们的项目中，classpath 中需要有 JDBC driver。

[HSQLDB](http://hsqldb.org/) 是一个非常知名的关系型数据（使用 Java 编写）。它广泛作为嵌入型数据库（译者注：嵌入你的程序之中）使用，因为这样可以避免依赖第三方数据库服务器而独立运行。在单元和集成测试时，它也常被用作提供易失性内存存储。

在开始阶段，HSQLDB 作为嵌入型数据库非常适合（我们的项目）。它在本地存储文件，并且 HSQLDB library Jar 提供了 JDBC driver，因此 Vert.x JDBC 的配置将会非常简单。

> 提示：
>
> Vert.x 也提供了 [MySQL 和 PostgreSQL client](http://vertx.io/docs/vertx-mysql-postgresql-client/java/) 专用的库。
>
> 当然你也可以使用通用的 Vert.x JDBC client 来连接 MySQL 或者 PostgreSQL 数据库，但上面的库使用这两种数据库的网络协议，而不是通过阻塞式的 JDBC API，因此会提供更好的性能

> 提示：
>
> Vert.x 也提供了处理流行的非关系型数据库 [MongoDB](http://vertx.io/docs/vertx-mongo-client/java/) 和 [Redis](http://vertx.io/docs/vertx-redis-client/java/) 的库。社区里也提供了其他存储系统的集成，例如 Apache Cassandra、OrientDB 和 ElasticSearch。

### 剖析 verticle

我们的 wiki 系统只有 ```io.vertx.guides.wiki.MainVerticle``` 这一个 verticle Java 类。这个类扩展自 ```io.vertx.core.AbstractVerticle```，作为 verticle 的基类，其主要：

1. 提供生命周期 ```start``` 与 ```stop``` 方法来重写
2. 提供一个名为 ```vertx``` 的受保护（protected）字段：verticle 被部署所在的 Vert.x 环境的引用
3. 提供一个对某些配置对象的访问器，用来传递一些外部配置给 verticle

为了启动我们的 verticle，只需重写如下的 ```start``` 方法：

```java
public class MainVerticle extends AbstractVerticle {

  @Override
  public void start(Future<Void> startFuture) throws Exception {
    startFuture.complete();
  }
}
```

```start``` 和 ```stop``` 方法有 2 种形式：一种没有参数，另一种带有一个 future 对象引用。无参情况表示 verticle 初始化或者 house-keeping phases 总是执行成功，除非有异常被抛出。带有 future 对象参数时，提供了一种更加细粒度的访问，来表明操作成功与否（译者注：通过 future 对象参数回掉判断）。实际上，一些初始化或者 cleanup 代码可能会要求异步操作，因此通过一个 future 对象给出结果理所当然，这符合异步的惯用表现形式。

### A word on Vert.x future objects and callbacks

Vert.x future 并不是 JDK 的 future：Vert.x future 可以在非阻塞式程序中被组织并检查结果。他们应被用于异步任务的简单协调，尤其是在部署 verticle 时检查部署成功与否。

Vert.x core API 基于回掉来实现异步事件的通知。富有经验的开发者自然会认为这开启了“回掉地狱”之门，如同下面这个虚构的例子一样，多层的异步嵌套会使得代码难以理解：

```
foo.a(1, res1 -> {
  if (res1.succeeded()) {
    bar.b("abc", 1, res2 -> {
      if (res.succeeded()) {
         baz.c(res3 -> {
           dosomething(res1, res2, res3, res4 -> {
               // (...)
           });
         });
      }
    });
  }
});
```

尽管 core API 在设计上就更加偏向（使用） promise 和 future，但回掉允许不同的编程概念（一起）被使用，因此使用回掉实际上也有道理。Vert.x 并不是一个固执己见的项目，许多异步编程模型的实现都可以使用回掉：reactive extensions (via RxJava)、promises 和 futures、fibers (using bytecode instrumentation) 等等。

既然在像 RxJava 这样的概念发挥影响力之前，所有的 Vert.x API 都是“回掉导向型”的，那么本手册在最开始时将**只使用**回掉，以确保读者可以熟悉 Vert.x 中的核心概念。可以说在刚刚开始时，在许多部分的异步代码块之中，使用回掉来画出一条线来更加容易。但一旦在示例代码中回掉开始让代码变得不再易读，我们就将会引入 RxJava 来展示同样的异步代码如果以处理事件流（streams of processed events）来考虑，将可以表示得更加优雅。

### Wiki verticle 初始化的步骤

为了让我们的 wiki 运行起来，需要分 2 步进行初始化：

1. 我们需要建立 JDBC 数据库连接，同时确保数据库模式的存在
2. 同时需要为我们的 web 应用来开启一个 HTTP 服务器

每个步骤都有失败的可能性（例如，HTTP 服务器需要的 TCP 端口可能已经被占用），并且他们不应当并行进行，web 应用的代码首先需要的是数据库可以正常访问。

为了使我们的代码更加简洁，我们为每个步骤定义一个方法，并且采用返回一个 future / promise 对象的形式来告知我们的步骤执行成功与否：

```java
private Future<Void> prepareDatabase() {
  Future<Void> future = Future.future();
  // (...)
  return future;
}

private Future<Void> startHttpServer() {
  Future<Void> future = Future.future();
  // (...)
  return future;
}
```

由于每个方法返回一个 future 对象，那么 ```start``` 方法的实现就可成为一个 composition：

```java
@Override
public void start(Future<Void> startFuture) throws Exception {
  Future<Void> steps = prepareDatabase().compose(v -> startHttpServer());
  steps.setHandler(startFuture.completer());
}
```

当 ```prepareDatabase``` 的 future 成功完成，然后 ```startHttpServer``` 就被调用，而 ```steps``` future 完成情况取决于 ```startHttpServer``` future 的结果。 如果 ```prepareDatabase``` 遇到了错误，```startHttpServer``` 则不会被调用，在这种情况下，```steps``` future 将以一个失败的状态完成，并携带一个描述错误的异常。

最终 ```steps``` 完成：```setHandler``` 定义了一个 hander，以供完成时调用。在上面的例子中，我们只是想使用 ```steps``` 来完成```setHandler```，并且通过 ```completer``` 方法来获得一个 handler。也可以写成：

```java
Future<Void> steps = prepareDatabase().compose(v -> startHttpServer());
steps.setHandler(ar -> {  // 注
  if (ar.succeeded()) {
    startFuture.complete();
  } else {
    startFuture.fail(ar.cause());
  }
});
```

注：```ar``` 的类型是 ```AsyncResult<Void>```。```AsyncResult<T>``` 被用来传递异步处理的结果，当成功时可能会有一个 ```T``` 类型的结果，当失败时传递一个失败异常。

#### 数据库初始化

Wiki 数据库模式由一张表 ```pages``` 构成，其字段信息如下：

|列名|类型|描述|
|---|----|----|
|Id|Integer|主键|
|Name|Characters|Wiki 页的名字，必须是唯一的|
|Content|Text|Wiki 页 Markdown 内容|

数据库操作是典型的“查、插、删、改”操作。在最开始，我们简单的将 SQL 语句以静态常量的形式存储在 ```MainVerticle``` 类中。请注意，他们是 HSQLDB 可解析的特定 SQL，有可能在其他关系型数据库中并不支持：

```java
private static final String SQL_CREATE_PAGES_TABLE = "create table if not exists Pages (Id integer identity primary key, Name varchar(255) unique, Content clob)";
private static final String SQL_GET_PAGE = "select Id, Content from Pages where Name = ?"; // 注
private static final String SQL_CREATE_PAGE = "insert into Pages values (NULL, ?, ?)";
private static final String SQL_SAVE_PAGE = "update Pages set Content = ? where Id = ?";
private static final String SQL_ALL_PAGES = "select Name from Pages";
private static final String SQL_DELETE_PAGE = "delete from Pages where Id = ?";
```

注：语句中的 ```?``` 是在执行时传递数据的占位符，因此 Vert.x JDBC client 可以防止 SQL 注入。

我们的应用 verticle 需要保持一个 ```JDBCClient``` 对象（来自 ```io.vertx.ext.jdbc``` 包）的引用来提供数据库的连接。我们在 ```MainVerticle``` 中声明了 ```dbClient```，并且创建了一个来自 ```org.slf4j``` 包的通用日志记录器。

```java
private JDBCClient dbClient;

private static final Logger LOGGER = LoggerFactory.getLogger(MainVerticle.class);
```

下面是一个 ```prepareDatabase``` 方法的完整实现。它尝试获取一个 JDBC client 连接，然后执行 SQL，在 ```Pages``` 表不存在的情况下来创建表：

```java
private Future<Void> prepareDatabase() {
  Future<Void> future = Future.future();

  dbClient = JDBCClient.createShared(vertx, new JsonObject()  // 注 1
    .put("url", "jdbc:hsqldb:file:db/wiki")   // 注 2
    .put("driver_class", "org.hsqldb.jdbcDriver")   // 注 3
    .put("max_pool_size", 30));   // 注 4

  dbClient.getConnection(ar -> {    // 注 5
    if (ar.failed()) {
      LOGGER.error("Could not open a database connection", ar.cause());
      future.fail(ar.cause());    // 注 6
    } else {
      SQLConnection connection = ar.result();   // 注 7
      connection.execute(SQL_CREATE_PAGES_TABLE, create -> {
        connection.close();   // 注 8
        if (create.failed()) {
          LOGGER.error("Database preparation error", create.cause());
          future.fail(create.cause());
        } else {
          future.complete();  // 注 9
        }
      });
    }
  });

  return future;
}
```

注：

1. ```createShared``` 创建一个共享的连接，其在 ```vertx``` 实例已知的 verticle 之间共享，通常来说这是一件好事。
2. 通过传递一个 JSON 对象来创建 JDBC client 连接。其中 ```url`` 指的是 JDBC url。
3. 使用 ```url```、```driver_class``` 等等来配置 JDBC driver 并且指出 JDBC driver 类。
4. ```max_pool_size``` 指的是并发连接数。这里设为 30 是武断决定，任意选择了一个数字。
5) 获取一个连接是异步操作，其提供给我们一个 ```AsyncResult<SQLConnection>```。它在使用之前必须检测是否可以建立连接（```AsyncResult``` 实际上是 ```Future``` 的超类接口）。
6. 如果不能得到 SQL 连接，future 就以失败为结果完成，并返回提供了异常（通过 ```cause``` 方法得到）的 ```AsyncResult```。
7. ```SQLConnection``` 是成功的 ```AsyncResult``` 的结果。我们可以使用它来执行 SQL 查询。
8. 在我们检查 SQL 查询执行成功与否之前，我们必须先通过调用 ```close``` 方法来释放连接，否则 JDBC client 连接池最终将干涸（无连接可用）。
9. 我们成功完成 future 操作。

> 提示：
>
> Vert.x 项目支持的 SQL 数据库模块目前主要关注提供对数据库的异步访问，除了传递 SQL 查询外，并没有提供其余更多（例如，对象-关系映射）。然而，完全可以使用[来自社区的更先进的模块](https://github.com/vert-x3/vertx-awesome)，我们尤其推荐了解一下像 [jOOq generator for Vert.x](https://github.com/jklingsporn/vertx-jooq) 或 [POJO mapper](https://github.com/BraintagsGmbH/vertx-pojo-mapper) 这样的项目。

#### 关于日志

上面引入了一个日志记录器，这里选择使用 [SFL4J library](https://www.slf4j.org/)。关于日志记录，Vert.x 也不是固执己见的：你可以选择任何流行的 Java 日志库。这里之所以推荐 SLF4J 是因为在 Java 生态之中，它是非常流行的日志抽象和统一库。

我们同样推荐使用 [Logback](https://logback.qos.ch/) 来作为日志记录器的实现。可以通过添加两个依赖，来同时集成 SLF4J 和 Logback，或者仅仅添加 ```logback-classic```，它将同时添加两个库的依赖（顺便一提，他们来自同一个作者）。

```
<dependency>
  <groupId>ch.qos.logback</groupId>
  <artifactId>logback-classic</artifactId>
  <version>1.2.3</version>
</dependency>
```

默认情况下，SLF4J 将会输出很多来自 Vert.x、Netty、C3PO 以及 wiki 应用的日志事件到控制台。我们可以通过添加一个 ```src/main/resources/logback.xml``` 配置文件来减少冗杂（查看 https://logback.qos.ch/ 这里获得更多信息）。

```xml
<configuration>

  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
    </encoder>
  </appender>

  <logger name="com.mchange.v2" level="warn"/>
  <logger name="io.netty" level="warn"/>
  <logger name="io.vertx" level="info"/>
  <logger name="io.vertx.guides.wiki" level="debug"/>

  <root level="debug">
    <appender-ref ref="STDOUT"/>
  </root>

</configuration>
```

最后但同样重要，HSQLDB 在内嵌时，与日志记录器集成得并不太好。默认情况下，它将尝试重新配置日志系统，所以我们在执行应用时，需要通过加 ```Dhsqldb.reconfig_logging=false``` 属性给 Java 虚拟机来禁用它这一点。

#### HTTP 服务器初始化

HTTP 服务器通过使用 ```vertx-web``` 项目，来比较容易得为传入的 HTTP 请求来定义分发路由。实际上，Vert.x core API 允许开启 HTTP 服务器并监听传入的连接，但它没有提供任何机制来根据请求 URL 不同来提供不同的 handler。根据 URL、HTTP 方法等等来分发请求到不同的处理 handler，这便是 router 的作用。

初始化过程由设置请求路由，开启 HTTP 服务器组成：

```java
private Future<Void> startHttpServer() {
  Future<Void> future = Future.future();
  HttpServer server = vertx.createHttpServer();   // 注 1

  Router router = Router.router(vertx);   // 注 2
  router.get("/").handler(this::indexHandler);
  router.get("/wiki/:page").handler(this::pageRenderingHandler); // 注 3
  router.post().handler(BodyHandler.create());  // 注 4
  router.post("/save").handler(this::pageUpdateHandler);
  router.post("/create").handler(this::pageCreateHandler);
  router.post("/delete").handler(this::pageDeletionHandler);

  server
    .requestHandler(router::accept)   // 注 5
    .listen(8080, ar -> {   // 注 6
      if (ar.succeeded()) {
        LOGGER.info("HTTP server running on port 8080");
        future.complete();
      } else {
        LOGGER.error("Could not start a HTTP server", ar.cause());
        future.fail(ar.cause());
      }
    });

  return future;
}
```

注：

1. ```vertx``` 上下文对象提供了方法来创建 HTTP 服务器、客户端，TCP/UDP 服务器、客户端等等。
2. ```Router``` 类来自 ```vertx-web```: ```io.vertx.ext.web.Router```。
3. 路由拥有自己的 handler，它们可以根据 URL 或 HTTP 方法来被定义。对于较短的 handler，可以采用 Java lambda 表达式的形式，但对于更复杂的 handler 来说，引用一个私有方法则更佳。注意 URL 可以带有参数，例如 ```/wiki/:page``` 将匹配类似 ```/wiki/Hello``` 这样的请求，这样 ```page``` 参数将被设置为 ```Hello```。
4. 这将会使所有 HTTP POST 请求通过 ```io.vertx.ext.web.handler.BodyHandler``` 这个 handler。它将自动的解码来自 HTTP 请求（例如，表单提交）（它们可以被作为 Vert.x buffer 对象来使用）的 body 体。
5. router 对象可以被用来作为 HTTP 服务器的 handler，然后分发请求给之前定义的其他 handler。
6. 开启一个 HTTP 服务器是异步操作，因此需要 ```AsyncResult<HttpServer>``` 来检查操作是否成功。```8080``` 参数具体指定了服务器的 TCP 端口。

### HTTP router handlers

```startHttpServer``` 方法的 HTTP 路由实例根据 URL 模式及 HTTP 方法的不同指向不同的 handler。每一个 handler 处理一个 HTTP 请求，执行数据库查询，以及使用 FreeMarker 模板来渲染 HTML 页面。

#### 索引页（主页） handler

主页提供了所有 wiki 页面的入口及一个创建新 wiki 的区域。

![index](https://github.com/zill057/vertx-guide-for-java-devs-chinese-translation/blob/master/01/images/index.png)

通过一个简单的 ```select *``` SQL 查询，并将数据交由 FreeMarker 引擎来渲染得到 HTML 响应。

```indexHandler``` 方法代码如下所示：

```java
private final FreeMarkerTemplateEngine templateEngine = FreeMarkerTemplateEngine.create();

private void indexHandler(RoutingContext context) {
  dbClient.getConnection(car -> {
    if (car.succeeded()) {
      SQLConnection connection = car.result();
      connection.query(SQL_ALL_PAGES, res -> {
        connection.close();

        if (res.succeeded()) {
          List<String> pages = res.result() // 注 1
            .getResults()
            .stream()
            .map(json -> json.getString(0))
            .sorted()
            .collect(Collectors.toList());

          context.put("title", "Wiki home");  // 注 2
          context.put("pages", pages);
          templateEngine.render(context, "templates", "/index.ftl", ar -> {   // 注 3
            if (ar.succeeded()) {
              context.response().putHeader("Content-Type", "text/html");
              context.response().end(ar.result());  // 注 4
            } else {
              context.fail(ar.cause());
            }
          });

        } else {
          context.fail(res.cause());  // 注 5
        }
      });
    } else {
      context.fail(car.cause());
    }
  });
}
```

注：

1. SQL 查询结果以 ```JsonArray``` 和 ```JsonObject``` 实例的形式返回。
2. ```RoutingContext``` 实例可以放置任意内容的键值对，可以供之后的模板或路由 handler 使用。
3. 渲染一个模板同样是异步操作，也使用 ```AsyncResult``` 处理方式。
4. 当成功时，```AsyncResult``` 包含的是渲染后的内容（以 ```String``` 形式），因此我们可以使用它来结束 HTTP 响应流。
5. 当失败时，```RoutingContext``` 的 ```fail``` 方法提供了一个合理的途径返回 HTTP 500 error 给 HTTP 客户端。

FreeMarker 模板应当被放置在 ```src/main/resources/templates``` 目录。```index.ftl``` 模板代码如下所示：

```html
<#include "header.ftl">

<div class="row">

  <div class="col-md-12 mt-1">
    <div class="float-xs-right">
      <form class="form-inline" action="/create" method="post">
        <div class="form-group">
          <input type="text" class="form-control" id="name" name="name" placeholder="New page name">
        </div>
        <button type="submit" class="btn btn-primary">Create</button>
      </form>
    </div>
    <h1 class="display-4">${context.title}</h1>
  </div>

  <div class="col-md-12 mt-1">
  <#list context.pages>
    <h2>Pages:</h2>
    <ul>
      <#items as page>
        <li><a href="/wiki/${page}">${page}</a></li>
      </#items>
    </ul>
  <#else>
    <p>The wiki is currently empty!</p>
  </#list>
  </div>

</div>

<#include "footer.ftl">
```

通过 FreeMarker 变量 ```context```，可以使用存储在 ```RoutingContext``` 对象之中的键值对数据。

因为很多模板都有着共同的页头与页脚，所以我们将其分离为 ```header.ftl``` 和 ```footer.ftl```：

* ```header.ftl```
  ```html
  <!DOCTYPE html>
  <html lang="en">
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">
    <meta http-equiv="x-ua-compatible" content="ie=edge">
    <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/4.0.0-alpha.5/css/bootstrap.min.css"
          integrity="sha384-AysaV+vQoT3kOAXZkl02PThvDr8HYKPZhNT5h/CXfBThSRXQ6jW5DO2ekP5ViFdi" crossorigin="anonymous">
    <title>${context.title} | A Sample Vert.x-powered Wiki</title>
  </head>
  <body>

  <div class="container">
  ```

* ```footer.ftl```
  ```html
  </div> <!-- .container -->

  <script src="https://ajax.googleapis.com/ajax/libs/jquery/3.1.1/jquery.min.js"
          integrity="sha384-3ceskX3iaEnIogmQchP8opvBy3Mi7Ce34nWjpBIwVTHfGYWQS9jwHDVRnpKKHJg7"
          crossorigin="anonymous"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/tether/1.3.7/js/tether.min.js"
          integrity="sha384-XTs3FgkjiBgo8qjEjBk0tGmf3wPrWtA6coPfQDfFEY8AnYJwjalXCiosYRBIBZX8"
          crossorigin="anonymous"></script>
  <script src="https://maxcdn.bootstrapcdn.com/bootstrap/4.0.0-alpha.5/js/bootstrap.min.js"
          integrity="sha384-BLiI7JTZm+JWlgKa0M0kGRpJbF2J8q+qreVrKBC47e3K6BW78kGLrCkeRX6I9RoK"
          crossorigin="anonymous"></script>

  </body>
  </html>
  ```

#### Wiki 页面渲染 handler

此 handler 处理 HTTP GET 请求，生成一个渲染过的 Wiki 页面，就像下图一样：

![page](https://github.com/zill057/vertx-guide-for-java-devs-chinese-translation/blob/master/01/images/page.png)

此页面同样提供了一个编辑按钮来以 Markdown 形式编辑内容。当按钮被点击时，不需要使用不同的 handler 与模板，只需简单的使用 JavaScript 和 CSS 来切换编辑器的开与关即可。

![edit](https://github.com/zill057/vertx-guide-for-java-devs-chinese-translation/blob/master/01/images/edit.png)

```pageRenderingHandler``` 方法代码如下：

```java
private static final String EMPTY_PAGE_MARKDOWN =
  "# A new page\n" +
    "\n" +
    "Feel-free to write in Markdown!\n";

private void pageRenderingHandler(RoutingContext context) {
  String page = context.request().getParam("page");   // 注 1

  dbClient.getConnection(car -> {
    if (car.succeeded()) {

      SQLConnection connection = car.result();
      connection.queryWithParams(SQL_GET_PAGE, new JsonArray().add(page), fetch -> {  // 注 2
        connection.close();
        if (fetch.succeeded()) {

          JsonArray row = fetch.result().getResults()
            .stream()
            .findFirst()
            .orElseGet(() -> new JsonArray().add(-1).add(EMPTY_PAGE_MARKDOWN));
          Integer id = row.getInteger(0);
          String rawContent = row.getString(1);

          context.put("title", page);
          context.put("id", id);
          context.put("newPage", fetch.result().getResults().size() == 0 ? "yes" : "no");
          context.put("rawContent", rawContent);
          context.put("content", Processor.process(rawContent));  // 注 3
          context.put("timestamp", new Date().toString());

          templateEngine.render(context, "templates", "/page.ftl", ar -> {
            if (ar.succeeded()) {
              context.response().putHeader("Content-Type", "text/html");
              context.response().end(ar.result());
            } else {
              context.fail(ar.cause());
            }
          });
        } else {
          context.fail(fetch.cause());
        }
      });

    } else {
      context.fail(car.cause());
    }
  });
}
```

注：

1. URL 参数（```/wiki/:name```）可以通过 context request 对象取得。
2. 通过 ```JsonArray``` 按照 ```?``` 的顺序，来传递参数给 SQL 查询。
3. ```Processor``` 类来自我们使用的 *txtmark* Markdown 渲染库。