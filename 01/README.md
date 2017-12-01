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
