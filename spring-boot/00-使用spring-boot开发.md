# 1.构建系统

## 1.1. 依赖管理

spring boot 提供了一些常见的依赖，并进行相应的版本管理。

在maven依赖中，一般放在了`spring-boot-starter-parent`项目中。

## 1.2. maven

使用maven来构建项目

## 1.5. Starters

介绍springboot常见的starter依赖包。

# 2.代码结构

项目工程目录结构的最佳实践。

# 3.配置类

## 3.1.导入外在的配置类

## 3.2.导入xml配置

# 4.自动配置

## 4.1.逐步取代自动配置

## 4.2.禁用指定的自动配置类

```java
@SpringBootApplication(exclude = { DataSourceAutoConfiguration.class })
public class MyApplication {

}
```

# 5.spring beans与依赖注入

# 6.@SpringBootApplication注解的使用

```java
// Same as @SpringBootConfiguration @EnableAutoConfiguration @ComponentScan
@SpringBootApplication
public class MyApplication {

    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }

}
```

# 7.运行应用

## 7.1.使用ide运行

## 7.2.使用打包好的应用包来运行

```shell
$ java -jar target/myapplication-0.0.1-SNAPSHOT.jar
```



## 7.3.使用maven插件来运行

```shell
$ mvn spring-boot:run
```

## 7.5.热插拔

借助开发工具来实现。

# 8.开发工具

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-devtools</artifactId>
        <!--没有依赖传递-->
        <optional>true</optional>
    </dependency>
</dependencies>
```



# 9.打包生产级别的应用包

这是需要添加对应的监控依赖，方便查看应用的状态。

一般使用`spring-boot-actuator`

```xml
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-actuator</artifactId>
      <version>2.7.0</version>
    </dependency>
```

