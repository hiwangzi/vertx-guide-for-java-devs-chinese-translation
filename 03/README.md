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
