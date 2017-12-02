## 使用 Vert.x 实现一个最小可实施的 wiki 系统

* 提示：

    相关的源代码可以在本手册（注：英文原版）的仓库 [step-1](https://github.com/vert-x3/vertx-guide-for-java-devs/tree/master/step-1) 目录下找到。

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

* 注意：

    另外，Fabric8 项目下有一个 [Vert.x Maven plugin](https://vmp.fabric8.io/)。它用于初始化、构建、打包并运行一个 Vert.x 项目。
    生成一个与克隆自 Git starter 仓库相似的项目：
    ```
    mkdir vertx-wiki
    cd vertx-wiki
    mvn io.fabric8:vertx-maven-plugin:1.0.7:setup -DvertxVersion=3.5.0
    git init
    ```

### 添加需要的依赖

首先在 Maven ```pom.xml``` 文件中添加用于 web 处理和渲染的依赖：

```
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

* 提示：

    正如 ```vertx-web-templ-freemarker``` 名字所表示的那样，对于流行的模板引擎，Vert.x web 提供了插件式的支持：Handlebars、Jade、MVEL、Pebble、Thymeleaf 以及 Freemarker。

然后添加 JDBC 数据访问相关的依赖：

```
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

* 提示：

    Vert.x 也提供了 [MySQL 和 PostgreSQL client](http://vertx.io/docs/vertx-mysql-postgresql-client/java/) 专用的库。

    当然你也可以使用通用的 Vert.x JDBC client 来连接 MySQL 或者 PostgreSQL 数据库，但上面的库使用这两种数据库的网络协议，而不是通过阻塞式的 JDBC API，因此会提供更好的性能

* 提示：

    Vert.x 也提供了处理流行的非关系型数据库 [MongoDB](http://vertx.io/docs/vertx-mongo-client/java/) 和 [Redis](http://vertx.io/docs/vertx-redis-client/java/) 的库。社区里也提供了其他存储系统的集成，例如 Apache Cassandra、OrientDB 和 ElasticSearch。

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
