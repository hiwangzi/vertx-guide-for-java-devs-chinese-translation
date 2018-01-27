## 再重构：Vert.x 服务

> 提示：
>
> 相关的源代码可以在本手册（注：英文原版）的仓库 [step-3](https://github.com/vert-x3/vertx-guide-for-java-devs/tree/master/step-3) 目录下找到。

相较于最初的实现来说，之前的重构已经是一个巨大的进步：其基于 event bus，独立可配置的 verticle 通过异步消息来连接；同时，我们部署的几个 verticle 实例可以更好的处理负载与施展 CPU 核心效能。

在下面这个部分之中，我们将看到如何设计并使用 Vert.x 服务（services）。服务的优势在于，它为 verticle 需要做的具体操作定义了接口。同时，不再像之前那样自己去处理消息，而是使用 event bus 消息管道。

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

第二，我们需要 Vert.x 代码生成模块，作为编译时唯一的依赖（hence the ```provided``` scope）：

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

为了移除生成的多余文件，可以更新 ```maven-clean-plugin```：

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

定义一个服务接口就像定义 Java 接口一样简单，除了必须要遵守一些规则，来生成代码以及保障 Vert.x 中的其他代码可以与之相互操作。

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

The Vert.x code generator creates the proxy class and names it by suffixing with VertxEBProxy. Constructors of these proxy classes need a reference to the Vert.x context as well as a destination address on the event bus:

Vert.x 代码生成器创建的代理类名字为类名加上 ```VertxEBProxy``` 后缀。代理类的构造方法需要 Vert.x 上下文引用以及 event bus 的目标地址。

> 注意：在上一版中，我们将 ```SqlQuery``` 和 ```ErrorCodes``` 枚举类型定义为了内部类，而这一版中，它们分别定义在 ```SqlQuery.java``` 和 ```ErrorCodes.java``` 之中。（译者注：可以直接去最前面提到的地址中获取代码）
