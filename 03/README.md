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
