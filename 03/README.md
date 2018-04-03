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

我们非常简单的通过实现类的构造方法来提供实现类的委托，将其定义为 ```create```：

```java
static WikiDatabaseService create(JDBCClient dbClient, HashMap<SqlQuery, String> sqlQueries, Handler<AsyncResult<WikiDatabaseService>> readyHandler) {
    return new WikiDatabaseServiceImpl(dbClient, sqlQueries, readyHandler);
}
```

Vert.x 代码生成器创建代理类，并将其命名为为类名加上 ```VertxEBProxy``` 后缀。代理类的构造方法需要 Vert.x 上下文引用以及 event bus 的目标地址。

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
