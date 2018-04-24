## 测试 Vert.x 代码

> 提示：
>
> 相关的源代码可以在本手册（注：英文原版）的仓库 [step-4](https://github.com/vert-x3/vertx-guide-for-java-devs/tree/master/step-4) 目录下找到。

截至目前，我们开发 wiki 系统过程中并没有进行测试。这当然不是一个好习惯，所以让我们看看怎么编写测试代码。

### 开始

在 Vert.x 中，`vertx-unit` 模块提供了测试异步操作的工具。除此之外，你还可以使用像是 JUint 这样的测试框架。

使用 JUnit 进行测试的话，Maven 依赖应当如下所示：

```xml
<dependency>
  <groupId>junit</groupId>
  <artifactId>junit</artifactId>
  <version>4.12</version>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>io.vertx</groupId>
  <artifactId>vertx-unit</artifactId>
  <scope>test</scope>
</dependency>
```

JUnit 测试代码需要添加注解 `VertxUnitRunner`，以使用 `vertx-unit` 的特性：

```java
@RunWith(VertxUnitRunner.class)
public class SomeTest {
    // (...)
}
```

引入这个 runner 后，JUnit 测试方法及生命周期方法都可以接受一个名为 `TestContext` 参数。其提供了基础断言、存储数据了的上下文，以及一些异步的工具。

让我们假设一个异步场景，我们想要检查一个 timer 任务是否被调用了一次，一个 periodic 任务是否被调用了三次。因为代码是异步的，所以测试方法会在测试完成之前就结束执行。因此我们的测试也需要以异步形式编写。

```java
@Test /*(timeout=5000)*/  // (8)
public void async_behavior(TestContext context) { // (1)
  Vertx vertx = Vertx.vertx();  // (2)
  context.assertEquals("foo", "foo");  // (3)
  Async a1 = context.async();   // (4)
  Async a2 = context.async(3);  // (5)
  vertx.setTimer(100, n -> a1.complete());  // (6)
  vertx.setPeriodic(100, n -> a2.countDown());  // (7)
}
```

1. `TestContext` 是 runner 提供的参数。
2. 我们需要创建一个 Vert.x 上下文来完成这个测试。
3. 这是一个基础 `TestContext` 断言的例子。
4. 我们得到一个 `Async` 对象，其可以稍后被标记为完成或失败。（译者注：它是一个测试的异步退出点）
5. 这个 `Async` 对象就如同一个计数器一样，在 3 次调用后，会被标记为完成。
6. 当 timer 执行的时候，测试正确完成。
7. 周期任务每次执行都会触发计数器。当 `Async` 对象完成的时候，测试通过。
8. 对于异步测试来说，有一个默认的超时限制，但可以在 JUnit `@Test` 注解中覆盖默认设定。

### 测试数据库操作

数据库服务非常适合编写测试代码。

首先我们先部署数据库 verticle。我们将其配置为 JDBC 连接 HSQLDB（内存存储数据库），当成功后我们取得一个“服务代理”来进行我们的测试。

因为这些操作过于细节（involving），所以我们引入 JUnit `before`，`after` 生命周期方法：

```java
private Vertx vertx;
private WikiDatabaseService service;

@Before
public void prepare(TestContext context) throws InterruptedException {
  vertx = Vertx.vertx();

  JsonObject conf = new JsonObject()  // (1)
    .put(WikiDatabaseVerticle.CONFIG_WIKIDB_JDBC_URL, "jdbc:hsqldb:mem:testdb;shutdown=true")
    .put(WikiDatabaseVerticle.CONFIG_WIKIDB_JDBC_MAX_POOL_SIZE, 4);

  vertx.deployVerticle(new WikiDatabaseVerticle(), new DeploymentOptions().setConfig(conf),
    context.asyncAssertSuccess(id ->  // (2)
      service = WikiDatabaseService.createProxy(vertx, WikiDatabaseVerticle.CONFIG_WIKIDB_QUEUE)));
}
```

1. 我们只覆盖一部分 verticle 的设置，其他的会采用默认值。
2. `asyncAssertSuccess` 提供了一个 handler 来检查异步操作的结果。它还有一个无参形式的重载，但对于有参形式来讲（如此处所示），我们可以将结果链接到另一个 handler。

清理Vert.x上下文非常简单，另外此处我们还是使用 `asyncAssertSuccess` 来检查是否有错误发生。

```java
@After
public void finish(TestContext context) {
    vertx.close(context.asyncAssertSuccess());
}
```

service的操作本质上是查插删改，所以 JUnit 最好将他们组合起来进行测试：

```java
@Test
public void crud_operations(TestContext context) {
  Async async = context.async();

  service.createPage("Test", "Some content", context.asyncAssertSuccess(v1 -> {

    service.fetchPage("Test", context.asyncAssertSuccess(json1 -> {
      context.assertTrue(json1.getBoolean("found"));
      context.assertTrue(json1.containsKey("id"));
      context.assertEquals("Some content", json1.getString("rawContent"));

      service.savePage(json1.getInteger("id"), "Yo!", context.asyncAssertSuccess(v2 -> {

        service.fetchAllPages(context.asyncAssertSuccess(array1 -> {
          context.assertEquals(1, array1.size());

          service.fetchPage("Test", context.asyncAssertSuccess(json2 -> {
            context.assertEquals("Yo!", json2.getString("rawContent"));

            service.deletePage(json1.getInteger("id"), v3 -> {

              service.fetchAllPages(context.asyncAssertSuccess(array2 -> {
                context.assertTrue(array2.isEmpty());
                async.complete();  // (1)
              }));
            });
          }));
        }));
      }));
    }));
  }));
  async.awaitSuccess(5000); // (2)
}
```

1. 这是 `Async` 结束完成的唯一退出点。
2. 还可以通过通过 JUnit 超时来退出测试。此处的测试线程会等待 `Async` 完成或者时间超时。