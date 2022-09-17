# 8.测试

Spring Boot提供了大量实用工具和注解来帮助测试应用程序。测试支持由两个模块提供:spring-boot-test包含核心项目，spring-boot-test-autoconfigure支持测试的自动配置。

大多数开发人员使用Spring - Boot - Starter -test“Starter”，它导入Spring Boot测试模块以及JUnit Jupiter、AssertJ、Hamcrest和许多其他有用的库。

-----

如果您有使用JUnit 4的测试，那么可以使用JUnit 5的老式引擎来运行它们。要使用vintage引擎，添加一个对junit-vintage-engine的依赖，如下所示:

```xml
<dependency>
    <groupId>org.junit.vintage</groupId>
    <artifactId>junit-vintage-engine</artifactId>
    <scope>test</scope>
    <exclusions>
        <exclusion>
            <groupId>org.hamcrest</groupId>
            <artifactId>hamcrest-core</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

hamcrest-core被排除，是因为org.hamcrest:hamcrest是spring-boot-starter-test的一部分。

-------

## 8.1. 测试范围的依赖

spring-boot-starter-test“Starter”(在测试范围内)包含以下提供的库:

- junit5：Java应用程序单元测试的实际标准。
- Spring Test & Spring Boot Test：支持Spring Boot应用程序的工具和集成测试。
- AssertJ：流畅的断言库。
- Hamcrest：对象匹配器相关的库。
- Mockito：java数据模拟框架
- JSONassert：JSON的断言库。
- JsonPath：JSON的XPath。

在编写测试时，我们通常会发现这些通用库非常有用。如果这些库不能满足您的需求，您可以添加自己的额外测试依赖项。

## 8.2. 测试spring应用程序

依赖注入的一个主要优点是，它应该使您的代码更容易进行单元测试。您可以使用new操作符实例化对象，甚至不需要涉及Spring。您还可以使用模拟对象而不是真正的依赖项。

通常，您需要在单元测试之上开始集成测试(使用Spring ApplicationContext)。能够在不需要部署应用程序或连接到其他基础设施的情况下执行集成测试是很有用的。

Spring框架包含一个专门用于这种集成测试的测试模块。可以直接声明org.springframework:spring-test依赖，或者用spring-boot-starter-test “Starter”将其拉入进来。

如果您以前没有使用过Spring测试模块，那么应该从阅读Spring框架参考文档的相关部分开始。

## 8.3. 测试springboot应用程序

一个Spring Boot应用程序是一个Spring ApplicationContext，所以除了普通Spring context通常要做的以外，没有什么特别的事情需要做。

> 只有使用SpringApplication创建Spring Boot时，Spring Boot的外部属性、日志记录和其他特性才会默认安装在context中。

SpringBoot提供了@SpringBootTest注解，在你需要SpringBoot特性时，可以用它替代标准的Spring -test @ContextConfiguration注解。注解的工作原理是通过SpringApplication创建测试中使用的ApplicationContext。除了@SpringBootTest，还提供了其他一些注解，用于测试应用程序的更具体的部分。

> 如果您正在使用JUnit 4，不要忘记还向您的测试添加@RunWith(SpringRunner.class)，否则注解将被忽略。如果您正在使用JUnit 5，则不需要添加等效的@Extendwith (SpringExtension.class)作为@ springboottest，其他的@…测试注解已经使用它进行了注释。

默认情况下，@SpringBootTest不会启动服务器。你可以使用@SpringBootTest的webEnvironment属性来进一步细化测试的运行方式:

- MOCK(默认):加载web ApplicationContext并提供一个模拟的web环境。使用此注释时，嵌入式服务器未启动。如果一个web环境在你的类路径中不可用，这个模式会透明地返回到创建一个常规的非web ApplicationContext。它可以与@AutoConfigureMockMvc或@AutoConfigureWebTestClient一起使用，用于对你的web应用进行基于模拟的测试。

- RANDOM_PORT:加载WebServerApplicationContext并提供一个真实的web环境。嵌入式服务器在一个随机端口上启动并侦听。
- DEFINED_PORT：加载WebServerApplicationContext并提供一个真实的web环境。嵌入式服务器启动并监听一个已定义的端口(来自application.properties)或默认端口8080。
- NONE：使用SpringApplication加载ApplicationContext，但不提供任何web环境(模拟或其他)。

> 如果测试是@Transactional，默认情况下，它会在每个测试方法结束时回滚事务。但是，由于使用RANDOM_PORT或DEFINED_PORT的这种安排隐式地提供了一个真正的servlet环境，HTTP客户机和服务器在单独的线程中运行，因此，在单独的事务中运行。在这种情况下，服务器上发起的任何事务都不会回滚。

> @SpringBootTest 的webEnvironment = WebEnvironment.RANDOM_PORT配置，如果应用程序为管理服务器使用了不同的端口，还将在一个单独的随机端口上启动管理服务器。