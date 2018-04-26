## 再重构：Vert.x 服务

> 提示：
>
> 相关的源代码可以在本手册（注：英文原版）的仓库 [step-3](https://github.com/vert-x3/vertx-guide-for-java-devs/tree/master/step-3) 目录下找到。

相较于最初的实现来说，之前的重构已经是一个巨大的进步：其基于 event bus，独立可配置的 verticle 通过异步消息来连接；同时，我们部署的几个 verticle 实例可以更好的应对负载以及更好的利用 CPU 核心。

在下面这个部分之中，我们将看到如何设计并使用 Vert.x 服务（services）。服务的优势在于，它为 verticle 需要做的具体操作定义了接口。同时，不再像之前那样自己去处理消息，而是利用代码生成来使用 event bus 消息。

Java 代码将被重构为以下几个包：
```
step-3/src/main/java/
└── io
    └── vertx
        └── guides
            └── wiki
                ├── MainVerticle.java
                ├── database
                │   ├── ErrorCodes.java
                │   ├── SqlQuery.java
                │   ├── WikiDatabaseService.java
                │   ├── WikiDatabaseServiceImpl.java
                │   ├── WikiDatabaseVerticle.java
                │   └── package-info.java
                └── http
                    └── HttpServerVerticle.java
```

* ```io.vertx.guides.wiki``` 现在包含 main verticle
* ```io.vertx.guides.wiki.database``` 包含数据库 verticle 以及 service
* ```io.vertx.guides.wiki.http``` 包含 HTTP 服务器 verticle

### 调整 Maven 配置

首先，我们的项目需要添加以下两个依赖。第一个是 ```vertx-service-proxy``` API：

```xml
<dependency>
    <groupId>io.vertx</groupId>
    <artifactId>vertx-service-proxy</artifactId>
</dependency>
```

第二，我们需要 Vert.x 代码生成模块，其只在编译时依赖（因此 scope 被设置为 ```provided```）：

```xml
<dependency>
    <groupId>io.vertx</groupId>
    <artifactId>vertx-codegen</artifactId>
    <scope>provided</scope>
</dependency>
```

另外为了能够生成代码，我们还需要稍微调整一下 ```maven-compiler-plugin``` 配置，通过一个 ```javac``` 注解处理器来实现：

```xml
<plugin>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>3.5.1</version>
    <configuration>
        <source>1.8</source>
        <target>1.8</target>
        <useIncrementalCompilation>false</useIncrementalCompilation>

        <annotationProcessors>
            <annotationProcessor>io.vertx.codegen.CodeGenProcessor</annotationProcessor>
        </annotationProcessors>
        <generatedSourcesDirectory>${project.basedir}/src/main/generated</generatedSourcesDirectory>
        <compilerArgs>
            <arg>-AoutputDirectory=${project.basedir}/src/main</arg>
        </compilerArgs>

    </configuration>
</plugin>
```

注意：生成的代码放置于 ```src/main/generated```，像 IntelliJ IDEA 这类 IDE 会自动将其加入 classpath。

为了移除生成的多余文件，更新 ```maven-clean-plugin``` 如下：

```xml
<plugin>
    <artifactId>maven-clean-plugin</artifactId>
    <version>3.0.0</version>
    <configuration>
        <filesets>
        <fileset>
            <directory>${project.basedir}/src/main/generated</directory>
        </fileset>
        </filesets>
    </configuration>
</plugin>
```

### 数据库服务接口

定义一个服务接口就像定义 Java 接口一样简单，除了必须要遵守一些特定规则，以生成代码及保证 Vert.x 中的其他代码可以与之相互操作。

接口最开始定义为如下形式：

```java
@ProxyGen
public interface WikiDatabaseService {

    @Fluent
    WikiDatabaseService fetchAllPages(Handler<AsyncResult<JsonArray>> resultHandler);

    @Fluent
    WikiDatabaseService fetchPage(String name, Handler<AsyncResult<JsonObject>> resultHandler);

    @Fluent
    WikiDatabaseService createPage(String title, String markdown, Handler<AsyncResult<Void>> resultHandler);

    @Fluent
    WikiDatabaseService savePage(int id, String markdown, Handler<AsyncResult<Void>> resultHandler);

    @Fluent
    WikiDatabaseService deletePage(int id, Handler<AsyncResult<Void>> resultHandler);

    // (...)
```

1. 通过 ```ProxyGen``` 注解的使用来触发此 service 的客户端代理代码的生成。
2. ```Fluent``` 注解是可选的，它意味着接口的方法可以链式调用。当 service 可能会被其他 JVM 语言使用的时候，这对于代码生成器来说很有用。
3. 参数类型默认只能是字符串、Java 基础类型、JSON 对象或数组、枚举类型、或者之前提到的类型的集合类型（```java.util```）（```List```、```Set```、```Map```）。若想使用其他任意的 Java 类作为 Vert.x 数据对象，需要为它们添加 ```@DataObject``` 注解。另外，还可以传递 service 引用类型变量。
4. 因为 service 提供了异步结果，所以最后一个参数定义为 ```Handler<AsyncResult<T>>```，在代码生成时，```T``` 是上面描述的任意一种适当的类型。

service 接口提供一个静态方法来为所有实际的实现提供实例对象，以及提供一个底层基于 event bus 的客户端代理，这样是比较好的实践。

我们简单得通过实现类的构造方法来将任务委派给实现类，将其定义为 ```create```：

```java
static WikiDatabaseService create(JDBCClient dbClient, HashMap<SqlQuery, String> sqlQueries, Handler<AsyncResult<WikiDatabaseService>> readyHandler) {
    return new WikiDatabaseServiceImpl(dbClient, sqlQueries, readyHandler);
}
```

Vert.x 代码生成器创建代理类，并使用类名加上 ```VertxEBProxy``` 后缀作为命名。代理类的构造方法需要 Vert.x 上下文引用以及 event bus 的目标地址。

```java
static WikiDatabaseService createProxy(Vertx vertx, String address) {
  return new WikiDatabaseServiceVertxEBProxy(vertx, address);
}
```

> 注意：在上一版中，我们将 ```SqlQuery``` 和 ```ErrorCodes``` 枚举类型定义为了内部类，而这一版中，它们分别定义在 ```SqlQuery.java``` 和 ```ErrorCodes.java``` 之中。（译者注：可以直接去最前面提到的地址中获取代码）

### 数据库服务实现

数据库服务的实现就是上一版 ```WikiDatabaseVerticle``` 的简易版本。最主要的区别就是构造函数提供异步结果 handler（来报告初始化结果），以及方法也同样提供了异步结果（来告知操作是否成功）。

类代码如下所示：

```java
class WikiDatabaseServiceImpl implements WikiDatabaseService {

    private static final Logger LOGGER = LoggerFactory.getLogger(WikiDatabaseServiceImpl.class);

    private final HashMap<SqlQuery, String> sqlQueries;
    private final JDBCClient dbClient;

    WikiDatabaseServiceImpl(JDBCClient dbClient, HashMap<SqlQuery, String> sqlQueries, Handler<AsyncResult<WikiDatabaseService>> readyHandler) {
        this.dbClient = dbClient;
        this.sqlQueries = sqlQueries;

        dbClient.getConnection(ar -> {
        if (ar.failed()) {
            LOGGER.error("Could not open a database connection", ar.cause());
            readyHandler.handle(Future.failedFuture(ar.cause()));
        } else {
            SQLConnection connection = ar.result();
            connection.execute(sqlQueries.get(SqlQuery.CREATE_PAGES_TABLE), create -> {
            connection.close();
            if (create.failed()) {
                LOGGER.error("Database preparation error", create.cause());
                readyHandler.handle(Future.failedFuture(create.cause()));
            } else {
                readyHandler.handle(Future.succeededFuture(this));
            }
            });
        }
        });
    }

    @Override
    public WikiDatabaseService fetchAllPages(Handler<AsyncResult<JsonArray>> resultHandler) {
        dbClient.query(sqlQueries.get(SqlQuery.ALL_PAGES), res -> {
        if (res.succeeded()) {
            JsonArray pages = new JsonArray(res.result()
            .getResults()
            .stream()
            .map(json -> json.getString(0))
            .sorted()
            .collect(Collectors.toList()));
            resultHandler.handle(Future.succeededFuture(pages));
        } else {
            LOGGER.error("Database query error", res.cause());
            resultHandler.handle(Future.failedFuture(res.cause()));
        }
        });
        return this;
    }

    @Override
    public WikiDatabaseService fetchPage(String name, Handler<AsyncResult<JsonObject>> resultHandler) {
        dbClient.queryWithParams(sqlQueries.get(SqlQuery.GET_PAGE), new JsonArray().add(name), fetch -> {
        if (fetch.succeeded()) {
            JsonObject response = new JsonObject();
            ResultSet resultSet = fetch.result();
            if (resultSet.getNumRows() == 0) {
            response.put("found", false);
            } else {
            response.put("found", true);
            JsonArray row = resultSet.getResults().get(0);
            response.put("id", row.getInteger(0));
            response.put("rawContent", row.getString(1));
            }
            resultHandler.handle(Future.succeededFuture(response));
        } else {
            LOGGER.error("Database query error", fetch.cause());
            resultHandler.handle(Future.failedFuture(fetch.cause()));
        }
        });
        return this;
    }

    @Override
    public WikiDatabaseService createPage(String title, String markdown, Handler<AsyncResult<Void>> resultHandler) {
        JsonArray data = new JsonArray().add(title).add(markdown);
        dbClient.updateWithParams(sqlQueries.get(SqlQuery.CREATE_PAGE), data, res -> {
        if (res.succeeded()) {
            resultHandler.handle(Future.succeededFuture());
        } else {
            LOGGER.error("Database query error", res.cause());
            resultHandler.handle(Future.failedFuture(res.cause()));
        }
        });
        return this;
    }

    @Override
    public WikiDatabaseService savePage(int id, String markdown, Handler<AsyncResult<Void>> resultHandler) {
        JsonArray data = new JsonArray().add(markdown).add(id);
        dbClient.updateWithParams(sqlQueries.get(SqlQuery.SAVE_PAGE), data, res -> {
        if (res.succeeded()) {
            resultHandler.handle(Future.succeededFuture());
        } else {
            LOGGER.error("Database query error", res.cause());
            resultHandler.handle(Future.failedFuture(res.cause()));
        }
        });
        return this;
    }

    @Override
    public WikiDatabaseService deletePage(int id, Handler<AsyncResult<Void>> resultHandler) {
        JsonArray data = new JsonArray().add(id);
        dbClient.updateWithParams(sqlQueries.get(SqlQuery.DELETE_PAGE), data, res -> {
        if (res.succeeded()) {
            resultHandler.handle(Future.succeededFuture());
        } else {
            LOGGER.error("Database query error", res.cause());
            resultHandler.handle(Future.failedFuture(res.cause()));
        }
        });
        return this;
    }
}
```

为了能够生成代理代码，还需要做一件事：在 service 包下增加一个 ```package-info.java``` 注释以用来定义一个 Vert.x 模块：

```java
@ModuleGen(groupPackage = "io.vertx.guides.wiki.database", name = "wiki-database")
package io.vertx.guides.wiki.database;

import io.vertx.codegen.annotations.ModuleGen;
```

### 在数据库 verticle 中暴露数据库服务

因为大部分数据库处理代码都被转移到了 `WikiDatabaseServiceImpl` 之中，所以 `WikiDatabaseVerticle` 类现在只包含两个方法： `start` 方法用于注册服务；另一个功用方法用于加载 SQL 语句：

```java
public class WikiDatabaseVerticle extends AbstractVerticle {

  public static final String CONFIG_WIKIDB_JDBC_URL = "wikidb.jdbc.url";
  public static final String CONFIG_WIKIDB_JDBC_DRIVER_CLASS = "wikidb.jdbc.driver_class";
  public static final String CONFIG_WIKIDB_JDBC_MAX_POOL_SIZE = "wikidb.jdbc.max_pool_size";
  public static final String CONFIG_WIKIDB_SQL_QUERIES_RESOURCE_FILE = "wikidb.sqlqueries.resource.file";
  public static final String CONFIG_WIKIDB_QUEUE = "wikidb.queue";

  @Override
  public void start(Future<Void> startFuture) throws Exception {

    HashMap<SqlQuery, String> sqlQueries = loadSqlQueries();

    JDBCClient dbClient = JDBCClient.createShared(vertx, new JsonObject()
      .put("url", config().getString(CONFIG_WIKIDB_JDBC_URL, "jdbc:hsqldb:file:db/wiki"))
      .put("driver_class", config().getString(CONFIG_WIKIDB_JDBC_DRIVER_CLASS, "org.hsqldb.jdbcDriver"))
      .put("max_pool_size", config().getInteger(CONFIG_WIKIDB_JDBC_MAX_POOL_SIZE, 30)));

    WikiDatabaseService.create(dbClient, sqlQueries, ready -> {
      if (ready.succeeded()) {
        ProxyHelper.registerService(WikiDatabaseService.class, vertx, ready.result(), CONFIG_WIKIDB_QUEUE); // (1)
        startFuture.complete();
      } else {
        startFuture.fail(ready.cause());
      }
    });
  }

  /*
   * Note: this uses blocking APIs, but data is small...
   */
  private HashMap<SqlQuery, String> loadSqlQueries() throws IOException {

    String queriesFile = config().getString(CONFIG_WIKIDB_SQL_QUERIES_RESOURCE_FILE);
    InputStream queriesInputStream;
    if (queriesFile != null) {
      queriesInputStream = new FileInputStream(queriesFile);
    } else {
      queriesInputStream = getClass().getResourceAsStream("/db-queries.properties");
    }

    Properties queriesProps = new Properties();
    queriesProps.load(queriesInputStream);
    queriesInputStream.close();

    HashMap<SqlQuery, String> sqlQueries = new HashMap<>();
    sqlQueries.put(SqlQuery.CREATE_PAGES_TABLE, queriesProps.getProperty("create-pages-table"));
    sqlQueries.put(SqlQuery.ALL_PAGES, queriesProps.getProperty("all-pages"));
    sqlQueries.put(SqlQuery.GET_PAGE, queriesProps.getProperty("get-page"));
    sqlQueries.put(SqlQuery.CREATE_PAGE, queriesProps.getProperty("create-page"));
    sqlQueries.put(SqlQuery.SAVE_PAGE, queriesProps.getProperty("save-page"));
    sqlQueries.put(SqlQuery.DELETE_PAGE, queriesProps.getProperty("delete-page"));
    return sqlQueries;
  }
}
```

* 注：

1. 我们在此处注册服务。

注册一个服务需要：一个接口类、Vert.x 上下文对象、实现类以及 event bus 目标地址。

The `WikiDatabaseServiceVertxEBProxy` generated class handles receiving messages on the event bus and then dispatching them to the `WikiDatabaseServiceImpl`. What it does is actually very close to what we did in the previous section: messages are being sent with a `action` header to specify which method to invoke, and parameters are encoded in JSON.

`WikiDatabaseServiceVertxEBProxy` 生成的类将处理来自 event bus 的消息，并将他们分发给 `WikiDatabaseServiceImpl` 去处理。这与之前我们编写的代码所做的事情非常接近：在上一节中，发送的消息中携带着一个名为 `action` 的头部，其指定了调用哪一个方法，参数采用 JSON 的形式。

### 使用一个数据库服务代理

将项目重构为使用 Vert.x 服务的最后一步就是改写 HTTP server verticle，其代码的 handler 中不再直接使用 event bus，转而使用数据库服务代理。

首先，我们需要在 verticle 启动时创建一个代理：

```java
private WikiDatabaseService dbService;

@Override
public void start(Future<Void> startFuture) throws Exception {

  String wikiDbQueue = config().getString(CONFIG_WIKIDB_QUEUE, "wikidb.queue"); // (1)
  dbService = WikiDatabaseService.createProxy(vertx, wikiDbQueue);

  HttpServer server = vertx.createHttpServer();
  // (...)
```

* 注：

1. 我们只需要保证此处的 event bus 目标地址与我们在 `WikiDatabaseVerticle` 发布服务时的地址一致即可。

然后，我们需要将代码中 event bus 的调用改为数据库服务：

```java
private void indexHandler(RoutingContext context) {
  dbService.fetchAllPages(reply -> {
    if (reply.succeeded()) {
      context.put("title", "Wiki home");
      context.put("pages", reply.result().getList());
      templateEngine.render(context, "templates", "/index.ftl", ar -> {
        if (ar.succeeded()) {
          context.response().putHeader("Content-Type", "text/html");
          context.response().end(ar.result());
        } else {
          context.fail(ar.cause());
        }
      });
    } else {
      context.fail(reply.cause());
    }
  });
}

private void pageRenderingHandler(RoutingContext context) {
  String requestedPage = context.request().getParam("page");
  dbService.fetchPage(requestedPage, reply -> {
    if (reply.succeeded()) {

      JsonObject payLoad = reply.result();
      boolean found = payLoad.getBoolean("found");
      String rawContent = payLoad.getString("rawContent", EMPTY_PAGE_MARKDOWN);
      context.put("title", requestedPage);
      context.put("id", payLoad.getInteger("id", -1));
      context.put("newPage", found ? "no" : "yes");
      context.put("rawContent", rawContent);
      context.put("content", Processor.process(rawContent));
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
      context.fail(reply.cause());
    }
  });
}

private void pageUpdateHandler(RoutingContext context) {
  String title = context.request().getParam("title");

  Handler<AsyncResult<Void>> handler = reply -> {
    if (reply.succeeded()) {
      context.response().setStatusCode(303);
      context.response().putHeader("Location", "/wiki/" + title);
      context.response().end();
    } else {
      context.fail(reply.cause());
    }
  };

  String markdown = context.request().getParam("markdown");
  if ("yes".equals(context.request().getParam("newPage"))) {
    dbService.createPage(title, markdown, handler);
  } else {
    dbService.savePage(Integer.valueOf(context.request().getParam("id")), markdown, handler);
  }
}

private void pageCreateHandler(RoutingContext context) {
  String pageName = context.request().getParam("name");
  String location = "/wiki/" + pageName;
  if (pageName == null || pageName.isEmpty()) {
    location = "/";
  }
  context.response().setStatusCode(303);
  context.response().putHeader("Location", location);
  context.response().end();
}

private void pageDeletionHandler(RoutingContext context) {
  dbService.deletePage(Integer.valueOf(context.request().getParam("id")), reply -> {
    if (reply.succeeded()) {
      context.response().setStatusCode(303);
      context.response().putHeader("Location", "/");
      context.response().end();
    } else {
      context.fail(reply.cause());
    }
  });
}
```

* `WikiDatabaseServiceVertxProxyHandler` 生成的类来处理转发的调用（就像之前的 event bus 消息一样）。

> 提示：
>
> 虽然生成了代理类来做这件事，但依然可以通过 event bus 消息来直接使用 Vert.x 服务。