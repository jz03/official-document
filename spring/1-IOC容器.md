## 1.1. spring ioc容器和bean的简介

本章介绍spring框架基于ioc控制反转原理的实现。ioc也可以称为依赖注入（DI）。一个对象的依赖，通常需要通过构造器的参数，在对象实例构造时工厂方法的参数，在对象实例上设置属性来实现。ioc容器在创建bean时注入这些依赖项，这个过程基本上是普通对象创建的逆向过程（因此成为控制反转）。通过使用类的直接构造，服务定位器模式等机制来控制其依赖项的实例化位置。

`org.springframework.beans`和`org.springframework.context`是spring ioc容器的基础包。

BeanFactory接口提供了一种高级配置机制，能够管理任何类型的对象。

ApplicationContext是BeanFactory的一个子接口。添加了如下内容：

- 与spring aop更容易集成
- 消息资源处理（主要用在国际化）
- 事件发布
- 定制化应用context，例如用在web应用的WebApplicationContext

简而言之，BeanFactory提供了基本的功能和配置框架，ApplicationContext添加了更多的基于企业的特定功能。

在spring中，构成应用程序的主干，并由spring容器管理的对象称之为bean。bean是由spring容器实例化、组装，管理的对象。否则bean只是应用程序中普通对象的一个。bean以及他们之间的依赖关系反映在容器使用的配置元数据中。

## 1.2. 容器概述

`org.springframework.context.ApplicationContext`接口表示spring ioc容器，他主要负责实例化，配置和组装bean。容器通过读取配置元数据信息获取关于实例化、配置和组装对象的指令。配置元数据主要表现为xml、Java注解和Java代码三种形式。他能够表达出组成应用程序的对象和对象之间的关系。

spring提供了ApplicationContext接口的几个实现，在独立的应用中，通常会创建ClassPathXmlApplicationContext和FileSystemXmlApplicationContext的实例。虽然xml是定义元数据的传统格式，但是可以通过提供少量的xml配置以声明的方式支持一些额外的元数据格式，从而指示容器使用Java注解和Java代码作为元数据的格式。

在大多数的应用程序中，用户不需要显式的来实例化spring ioc容器。

![container magic](container-magic.png)

上图展示了spring ioc容器如何工作的高级视图。应用程序类与元数据相结合，在ApplicationContext创建和初始化之后，产生一个可配置、可执行的系统（应用程序）。

### 1.2.1. 配置元数据

如上图所示，spring ioc容器使用一种形式的配置元数据，这个配置元数据表示作为应用程序的开发人员，是如何告诉spring容器实例化，配置和组装应用程序中的对象。

配置元数据传统上以简单直观的xml格式来提供，本章的大部分内容都使用这种格式来传达spring ioc容器的关键概念和特性。

> 基于xml的元数据不是唯一的配置格式，spring ioc容器本身与实际编写此配置元数据的格式是完全解耦的。现在的许多开发人员喜欢使用基于Java的配置格式。

有关spring容器的其他形式的配置，请参见：

- 基于注解的配置：spring 2.5引入了基于注解的配置
- 基于Java的配置：从spring3.0开始，spring Javaconfig项目提供的许多特性成为核心spring框架的一部分。你可以使用Java代码来代替xml的配置。要使用这些新特性，请参阅@Configuration、@Bean、@Import和@DependsOn这些注解。

spring配置中，ioc容器必须管理至少一个bean（通常不止一个）。基于xml的配置将这些bean配置在<bean>的元素中。基于Java的配置通常在@Configuration注解的类中使用@Bean注释的方法。

这些bean定义对应应用程序中的实际对象。通常需要定义service层的对象、数据访问层的对象（DAO）、表示对象（struts action实例）、基础设施对象（Hibernate 的SessionFactories）JMS Queues等等。通常不需要在容器中配置细粒度的域对象，因为创建和加载这些对象通常是dao层和业务逻辑层的职责。但是，可以使用spring与aspect的集成来配置在ioc容器控制之外的对象。详细请参阅aspectJ

下面展示了基于xml配置的基本结构：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="..." class="...">  ①②
        <!-- collaborators and configuration for this bean go here -->
    </bean>

    <bean id="..." class="...">
        <!-- collaborators and configuration for this bean go here -->
    </bean>

    <!-- more bean definitions go here -->

</beans>

```

①id属性是一个字符串，用来表示单个bean的定义。

②class属性定义bean的class类型，并且使用完全限定的类名。

id属性的值指向的是协作对象，本例中没有显示用于协作对象，有关更多信息，请参阅dependencies部分。

### 1.2.2. 容器实例化

提供给ApplicationContext构造的资源路径是字符串类型，他允许从各种外部资源加载配置元数据，例如：本地文件系统，Java CLASSPATH等等。

```java
ApplicationContext context = new ClassPathXmlApplicationContext("services.xml", "daos.xml");
```

> 在了解了spring ioc容器之后，可能更多的了解资源抽象，spring提供了一个方便的从uri语法中定义位置，读取资源（InputStream ）的机制。具体来说，资源路径用于构建应用程序的context，如应用程序context和资源路径中所述。

下面的示例展示了服务层对象的配置文件(services.xml)：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

    <!-- services -->

    <bean id="petStore" class="org.springframework.samples.jpetstore.services.PetStoreServiceImpl">
        <property name="accountDao" ref="accountDao"/>
        <property name="itemDao" ref="itemDao"/>
        <!-- additional collaborators and configuration for this bean go here -->
    </bean>

    <!-- more bean definitions for services go here -->

</beans>

```

紧跟着上边的服务层对象，下面展示的是数据访问层的对象配置文件（daos.xml）：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="accountDao"
        class="org.springframework.samples.jpetstore.dao.jpa.JpaAccountDao">
        <!-- additional collaborators and configuration for this bean go here -->
    </bean>

    <bean id="itemDao" class="org.springframework.samples.jpetstore.dao.jpa.JpaItemDao">
        <!-- additional collaborators and configuration for this bean go here -->
    </bean>

    <!-- more bean definitions for data access objects go here -->

</beans>
```

在前边的示例中，服务层是由PetStoreServiceImpl类和JpaAccountDao、JpaItemDao（基于jpa对象关系映射标准）的数据访问对象组成。property的name元素是Java bean的属性名称，ref元素引用的是另一个bean定义的名称。id和ref元素之间的连接表示协作对象之间的依赖关系。关于配置的依赖关系，请参考dependencies章节。

#### 编写基于xml的配置元数据

让bean定义跨多个xml文件可能很有用，通常，每个单独的xml配置文件代表着体系结构中的一个逻辑层或模块。

可以使用应用程序context构造函数从所有这些xml片段加载bean定义。这个构造函数能够接受多个xml文件，如上一节所示。或者使用<import>元素从另一个或多个文件加载bean定义。下边的例子展示了如何做：

```xml
<beans>
    <import resource="services.xml"/>
    <import resource="resources/messageSource.xml"/>
    <import resource="/resources/themeSource.xml"/>

    <bean id="bean1" class="..."/>
    <bean id="bean2" class="..."/>
</beans>
```

在上边的例子中，外部bean定义是从三个文件（services.xml、messageSource.xml、themeSource.xml）加载的，所有的位置路径都是相对于当前的配置文件来导入的。所以services.xml文件必须和执行导入的xml配置文件在同一个路径下，messageSource.xml和themeSource.xml必须位于导入xml配置文件的下方resources目录下。如你所见，`/`会被忽略掉。考虑到这些路径都是相对的，最好不要用`\`。根据Spring Schema的定义，被导入的文件内容，包括顶级<bean>元素，必须是有效的xml bean定义。

> 使用相对路径`../`来访问上一级目录是可以的，但是不建议这样做，这样做会对当前应用程序之外的文件创建依赖关系。特别说明的是，不推荐使用classpath：url（例如：classpath:../services.xml），在运行解析时会选择最近的路径，然后查看上一级目录。类路径配置可能发生改变，导致选择一个不同的不正确的目录。
>
> 你也总是使用完全限定的资源路径而不是相对路径，例如：`file:C:/config/services.xml`或`classpath:/config/services.xml`。但是请注意，你正在将应用程序的配置耦合到特定的绝对位置。通常更可取的做法是对这种绝对位置保持一个参数，例如，通过在运行时根据jvm系统属性解析`${}`中的值。

命名空间提供了导入指令。除了普通的bean定义之外，spring提供了一系列的xml命名空间，还提供了更多的配置特性，例如：contex和util命名空间。

### 1.2.3. 使用容器

ApplicationContext是高级工厂的接口，能够维护不同的bean及其依赖项。通过使用方法`T getBean(String name, Class<T> requiredType)`可以检索到bean实例。

ApplicationContext允许访问bean定义并访问他们。示例如下：

```xml
// create and configure beans
ApplicationContext context = new ClassPathXmlApplicationContext("services.xml", "daos.xml");

// retrieve configured instance
PetStoreService service = context.getBean("petStore", PetStoreService.class);

// use configured instance
List<String> userList = service.getUsernameList();
```

最灵活的变体是GenericApplicationContext与阅读器委托的组合。例如：xml文件的XmlBeanDefinitionReader，如下所示。

```java
GenericApplicationContext context = new GenericApplicationContext();
new XmlBeanDefinitionReader(context).loadBeanDefinitions("services.xml", "daos.xml");
context.refresh();
```

也可以在同一个ApplicationContext上混合和匹配这样的阅读器委托，从不同的配置源读取bean定义。

然后可以使用getBean方法来检索bean的实例。ApplicationContext接口还有一些其他的检索方法，但是在理想情况下，应用程序的代码永远不应该使用它。

实际上，在应用程序中的代码，根本不应该调用getBean方法，因此根本不用依赖spring api。例如，spring与web框架的集成为各种web框架的集成了各种组件（控制器和jsf管理的bean），提供了依赖注入，允许开发者通过元数据配置声明对特定bean的依赖。

## 1.3. bean的概述

spring ioc容器管理一个或多个bean，这些bean是用开发者提供给容器的配置元数据创建的（例如，xml文件中的<bean>定义）。

在容器内部，这些bean定义被表示为BeanDefinition对象，包含以下元数据：

- 一个限定的类名：通常是定义bean的实际实现类。
- bean行为配置元素，规定了bean在容器中有哪些行为（scope, lifecycle callbacks等等）。
- 对bean执行工作所需的其他bean的引用，这些引用称作协作者或依赖项。
- 在新创建的对象中设置其他配置，例如：bean连接池中的大小限制或连接数。

此元数据转化成了每个bean定义的一组属性。下表描述了这些属性：

| 属性                     | 说明               |
| ------------------------ | ------------------ |
| class                    | 实例化bean         |
| name                     | 对bean进行命名     |
| scope                    | bean的适用范围     |
| Constructor arguments    | 依赖注入           |
| Properties               | 依赖注入           |
| Autowiring mode          | 自动组装协作的bean |
| Lazy initialization mode | 延迟初始化bean     |
| Initialization method    | 初始化回调         |
| Destruction method       | 销毁回调           |

除了包含如何创建特定bean的信息bean定义之外，ApplicationContext实现类还允许注册在容器外部（由用户）创建现有对象。这是通过`getBeanFactory()`方法访问ApplicationContext的BeanFactory来完成的，该方法返回`DefaultListableBeanFactory`的实现。DefaultListableBeanFactory支持这个通过`registerSingleton(..)`和`registerBeanDefinition(..)`方法来注册。但是，典型的应用程序只使用通过常规的方法来创建bean定义。

> bean元数据和手动提供的单例实例需要尽早注册，以便容器在自动装配和其他自省步骤中正确的使用他们。虽然在某种程度上支持现有的元数据和单例，但是官方并不支持在运行时注册新bean，这可能会导致并发访问异常、bean在容器中状态不一致，或者两者都有。

### 1.3.1. bean的命名

每个bean都有一个或多个标识符，这些标识符在ioc容器中必须是唯一的，一个bean通常只有一个标识符。但是，如果需要一个以上，额外的可以称为别名。

> 相关代码在BeanDefinitionHolder类中

在基于xml的配置元数据中，可以使用id、name属性或两者来指定bean的标识符。id属性允许你指定一个id。按照惯例，这些名字是字母数字组成（'myBean', 'someService'），但是也可以包含特殊字符。如果希望引入bean的其他别名，还可以在name属性中定义他们，用逗号、分号和空格进行分隔。作为一个历史记录，在spring3.1之前的版本中，id属性被定义为xsd:id类型，限制了可能的字符。从3.1开始，他被定义为xsd:string类型。请注意，bean的id唯一性仍然由容器强制控制，不再是由xml解析器强制控制。

开发者不需要为bean提供名称和id。如果没有显式的提供名称和id，容器将为该bean生成一个唯一的名称，但是，如果想通过名称来引用一个bean，可以通过使用ref元素或者是服务定位的形式来查找，但是必须需要提供一个名称。不提供名称的机制与使用内部bean和自动装配有关。

**bean的命名规约**

规约规定了在命名bean时对实例字段名称使用标准的Java约定。也就是说，bean名称以小写字母开头，并从那里开始使用驼峰大小写。例如：accountManager`, `accountService`, `userDao`, `loginController等等。

一致的命名bean使你的配置更容易阅读和理解。另外，如果使用spring aop，在将通知应用到一组与名称相关的bean时，会有很大的帮助。

> 在通过classpath扫描组件中，spring为未命名的组件生成bean名，遵循前面所述的规则：首字母小写的驼峰格式。但是，在特殊情况下，当有多个字符并且第一个和第二个都是大写字母时，将保留原来的大小写，这些规则与`java.beans.Introspector.decapitalize`定义的规则相同。

#### 在一个bean定义之外起一个别名

在bean自身的定义中，通过使用id属性来指定一个bean的名称，和name属性中定义多个其他名称相组合，来提供多个名称。这些名称可以是同一个bean的等价别名，在某些情况下十分有用，例如：通过使用该组件的特定名称，让应用程序中的每一个组件引用一个公共依赖。

但是，指定bean实际定义的别名并不总是足够的，有时需要为在其他地方来引入bean的别名。这种情况经常在大型场景中出现，配置被划分为多个子系统，每个子系统都有自己的定义。在基于xml的配置元数据中，可以使用<alias>元素来完成这样的任务。下面的例子展示了如何使用：

```xml
<alias name="fromName" alias="toName"/>
```

在这种情况下，在使用别名定义之后，名为fromName的bean（在同一个容器中）也可以称为toName。

例如，子系统A的配置元数据可能是以subsystemA-dataSource来命名的，引用数据源。子系统B是以subsystemB-dataSource名称来引用数据源。当组合使用这两个子系统的主应用程序时，主应用程序是以myApp-dataSource名称来引用数据源。要想使三个系统指向同一个对象时，可以使用如下方式来实现：

```xml
<alias name="myApp-dataSource" alias="subsystemA-dataSource"/>
<alias name="myApp-dataSource" alias="subsystemB-dataSource"/>
```

现在每个组件和主应用程序都可以使用同一个数据源对象，该名称不会与任何其他定义冲突。

> 基于java配置
>
> 如果使用Javaconfiguration的方式，可以使用@Bean注解来实现，具体使用方法请参考`Using the `@Bean` Annotation`章节。

###  1.3.2. 实例化bean

bean定义本质上是创建一个或多个对象的方法。容器在被请求查看bean的配置，并使用该配置完成实际对象的创建。

如果使用基于xml的配置元数据，则需要<bean>元素的class属性中指定要实例化的对象类型。class属性通常是强制性的，使用class属性有两种方式：

- 一般情况下，在容器本身通过反射调用构造器来直接创建对象，这相当于Java代码中的new操作符。
- 在特殊情况下，指定包含静态工厂方法来创建对象，容器调用类上的静态工厂方法来创建对象。调用静态工厂方法返回的对象类型可能是同一个类，也可以是另外一个类。

**嵌套的类名**(静态内部类)

如果希望为嵌套内部类配置bean定义，可以使用嵌套类的二级制名称或源名称。

例如：在`com.example`包中有一个类叫SomeThing，这个SomeThing类有一个静态内部类称为OtherThing，他们可以用一个$或`.`来分隔。因此这个类的class属性可以是`com.example.SomeThing$OtherThing`或`com.example.SomeThing.OtherThing`来表示。

#### 使用构造器来实例化

通过构造器创建bean时，spring可以兼容所有的普通类。也就是说，正在开发的类不需要实现任何特定的接口，也不需要以特定的方式编码，简单的指定bean类就行了。但是，根据对特定的bean类型，可能需要提供一个默认的构造器。

spring ioc容器实际上可以管理任何类，并不局限于真正的Java bean，大多数情况下，spring用户更加喜欢实际的Java bean，只有一个默认的构造器，并根据容器中的属性创建适当的getter和setter方法。spring容器中还有更多的非bean奇特类。例如，如果完全不遵守Java bean规范的连接池，spring也可以进行管理。

基于xml的配置元数据，如下所示：

```xml
<bean id="exampleBean" class="examples.ExampleBean"/>

<bean name="anotherExample" class="examples.ExampleBeanTwo"/>
```

关于向构造器中传递参数和在对象构造之后设置属性的机制，请参考Injecting Dependencies部分。

#### 使用静态工厂方法来实例化

当使用静态工厂方法创建定义一个bean时，使用class属性来指定包含静态工厂方法的类，并使用factory-method属性来指定工厂方法。开发者能够调用这个方法（带有参数的稍后进行描述）并返回一个对象，该对象随后将被视为通过构造器创建的对象。这种定义方式适合在遗留的代码中使用。

下面的bean定义是通过调用工厂方法来创建的，定义没有指定返回对象的类型，只指定了包含工厂的类。在这个例子中createInstance()方法必须是静态方法。

```xml
<bean id="clientService"
    class="examples.ClientService"
    factory-method="createInstance"/>
```

下面展示了Java中的静态工厂：

```java
public class ClientService {
    private static ClientService clientService = new ClientService();
    private ClientService() {}

    public static ClientService createInstance() {
        return clientService;
    }
}
```

关于为工厂方法提供参数，对象在从工厂方法返回后设置属性的详细信息，请参考依赖和配置。

#### 使用实例化工厂方法类实例化

与通过静态工厂方法进行的实例化类似，使用实例化的工厂方法进行实例化，从容器中调用现有bean的非静态方法来创建新bean。

要使用此功能，请将class属性设置为空，并在factory-bean属性中指定在容器中包含工厂方法的bean名称，使用factory-method属性设置工厂方法名称。下面的例子展示了如何设置：

```xml
<!-- the factory bean, which contains a method called createInstance() -->
<bean id="serviceLocator" class="examples.DefaultServiceLocator">
    <!-- inject any dependencies required by this locator bean -->
</bean>

<!-- the bean to be created via the factory bean -->
<bean id="clientService"
    factory-bean="serviceLocator"
    factory-method="createClientServiceInstance"/>
```

下面展示的是Java相关代码：

```java
public class DefaultServiceLocator {

    private static ClientService clientService = new ClientServiceImpl();

    public ClientService createClientServiceInstance() {
        return clientService;
    }
}
```

一个工厂有多个工厂方法，每个方法返回不同类型的对象，下面的示例展示了这些：

```xml
<bean id="serviceLocator" class="examples.DefaultServiceLocator">
    <!-- inject any dependencies required by this locator bean -->
</bean>

<bean id="clientService"
    factory-bean="serviceLocator"
    factory-method="createClientServiceInstance"/>

<bean id="accountService"
    factory-bean="serviceLocator"
    factory-method="createAccountServiceInstance"/>
```

下面展示了对应的工厂类：

```java
public class DefaultServiceLocator {

    private static ClientService clientService = new ClientServiceImpl();

    private static AccountService accountService = new AccountServiceImpl();

    public ClientService createClientServiceInstance() {
        return clientService;
    }

    public AccountService createAccountServiceInstance() {
        return accountService;
    }
}
```

这种方法表明工厂bean本身可以通过依赖注入来配置管理。

> 在spring文档中，“factory bean”指的是spring容器中的bean，通过实例或静态工厂方法来创建对象。相比之下，FactoryBean（注意大小写）指的是一个特定与spring的FactoryBean的实现类。

#### 确定bean的运行时类型

特定bean的运行时类型不容易被确定。bean元数据定义中的类只是一个初始类的引用，可能与声明的工厂方法结合在一起，也可能导致bean的不同运行时类型的FactoryBean类型，或者在实例化的工厂方法中根本不用设置。另外aop代理可以使用基于接口的代理，有限的公开目标bean的实际类型。

了解特定bean实际的运行时类型，建议使用的BeanFactory.getType方法。使用getType方法获取bean。这将考虑上述所有的情况，BeanFactory.getBean方法返回相同的bean名称。

## 1.4. 依赖

典型的企业应用不是由单个对象组成。即便是最简单的应用程序也有几个对象一起工作，以呈现给用户最终使用的程序。接下来将介绍如何从多个单独的bean对象到完全实现的应用程序，通过对象之间的协作来实现这个目的。

### 1.4.1. 依赖注入

依赖注入的过程：对象通过构造函数的参数，工厂方法的参数，在对象实例构造完成之后，在对象实例上设置属性来定义他们之间的依赖项，然后容器在创建bean时注入这些依赖项。这个过程从根本上说是bean本身的逆向过程（因此称为控制反转），通过构造器和服务定位器来控制依赖项的位置和实例化。

使用依赖注入原则，让代码更加清晰，当对象和他的依赖项在一起时，解耦更加有效。对象不需要查找其依赖项，也不用知道依赖项的位置和类型。因此，类变得更加容易测试，特别是依赖关系是接口或者是抽象类时，容易在单元测试中模拟实现。

DI主要有两种变体：基于构造器的依赖注入和基于setter的依赖注入。

#### 基于构造器的依赖注入

基于构造器的依赖注入是由容器调用带有多个参数的构造器完成的，每个参数代表一个依赖项。调用带有特定参数的静态工厂方法来构造bean几乎是等价的。本章将以类似的方式处理构造器和静态工厂方法的参数。下面展示了只能通过构造器实现的依赖注入：

```java
public class SimpleMovieLister {

    // the SimpleMovieLister has a dependency on a MovieFinder
    private final MovieFinder movieFinder;

    // a constructor so that the Spring container can inject a MovieFinder
    public SimpleMovieLister(MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }

    // business logic that actually uses the injected MovieFinder is omitted...
}
```

注意，这个类没什么特别之处，不依赖于容器的特定接口、基类或注解。

##### 构造函数参数解析

构造器参数解析匹配通过使用参数的类型开始。如果bean定义的构造器参数不存在异议，那么构造器中参数的顺序就是实例化的顺序。思考下边的类：

```java
package x.y;

public class ThingOne {

    public ThingOne(ThingTwo thingTwo, ThingThree thingThree) {
        // ...
    }
}
```

假设ThingTwo和ThingThree不存在继承关系，下面的配置将会正常工作，并且不需要在<constructor-arg>元素中显式的指定构造参数索引或类型。

```xml
<beans>
    <bean id="beanOne" class="x.y.ThingOne">
        <constructor-arg ref="beanTwo"/>
        <constructor-arg ref="beanThree"/>
    </bean>

    <bean id="beanTwo" class="x.y.ThingTwo"/>

    <bean id="beanThree" class="x.y.ThingThree"/>
</beans>
```

当引用另一个bean时，类型是已知的，可以进行匹配（如上所示）。当使用基本类型时，直接用<value>元素给出来，例如：<value>true</value>，spring不能识别value的类型，因此无法在没有其他帮助下进行类型匹配。思考下边的类：

```java
package examples;

public class ExampleBean {

    // Number of years to calculate the Ultimate Answer
    private final int years;

    // The Answer to Life, the Universe, and Everything
    private final String ultimateAnswer;

    public ExampleBean(int years, String ultimateAnswer) {
        this.years = years;
        this.ultimateAnswer = ultimateAnswer;
    }
}
```

*构造器参数类型匹配*

在上述场景中，如果使用type属性显式指定构造器参数类型，容器可以使用基本类型来匹配，如下所示：

```xml
<bean id="exampleBean" class="examples.ExampleBean">
    <constructor-arg type="int" value="7500000"/>
    <constructor-arg type="java.lang.String" value="42"/>
</bean>
```

*构造器参数索引*

可以使用index属性来显式的指定构造器参数的索引，如下所示：

```xml
<bean id="exampleBean" class="examples.ExampleBean">
    <constructor-arg index="0" value="7500000"/>
    <constructor-arg index="1" value="42"/>
</bean>
```

index除了可以解决基本类型的不确定外，还可以解决构造器同时具有相同类型参数的不确定性。

> 索引是从0开始的

*构造器参数名*

也可以通过使用构造器参数名来解决不确定性，如下所示：

```xml
<bean id="exampleBean" class="examples.ExampleBean">
    <constructor-arg name="years" value="7500000"/>
    <constructor-arg name="ultimateAnswer" value="42"/>
</bean>
```

请记住，要使用此功能，必须在编译代码时启用调试标志，spring以便可以从构造器中查找参数名称。如果不想带有调试标志，可以使用@ConstructorProperties jdk注解显式的命名构造器参数。如下所示：

```java
package examples;

public class ExampleBean {

    // Fields omitted

    @ConstructorProperties({"years", "ultimateAnswer"})
    public ExampleBean(int years, String ultimateAnswer) {
        this.years = years;
        this.ultimateAnswer = ultimateAnswer;
    }
}
```

#### 基于setter的依赖注入

基于setter的依赖注入是由容器在调用无参默认构造器或无参静态工厂方法实例化bean之后，调用对象中的setter方法来完成的。

下面的例子展示了一个只能通过setter的方式注入依赖项。这个类是传统的Java，不依赖于spring的任何接口、基类和注解。

```java
public class SimpleMovieLister {

    // the SimpleMovieLister has a dependency on the MovieFinder
    private MovieFinder movieFinder;

    // a setter method so that the Spring container can inject a MovieFinder
    public void setMovieFinder(MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }

    // business logic that actually uses the injected MovieFinder is omitted...
}
```

ApplicationContext为其管理的bean，即支持构造器的方式，也支持setter的方式。在已经通过构造器创建的bean对象之后，还支持基于setter的方式。可以支持BeanDefinition的格式配置依赖项，也可以与PropertyEditor实例一起使用，将属性从一种格式转为另一种格式。然而，大多数的开发者并不直接使用这些类，而是使用xml的方式、带注解的组件（使用@Component`, `@Controller等进行注解的类）、或者基于Java的@Configuration注解类中的@Bean方法。然后将这些配置转换为BeanDefinition实例，并用于加载到整个spring ioc容器实例中。

**基于构造器还是setter来实现依赖注入**

由于可以混合使用两种方式，根据经验，对于强制依赖项使用构造器的方式，对于可选的依赖项使用setter的方式来实现。请注意，在setter方法上使用@Required注解可以使该属性变为必须的依赖项。

spring团队通常提倡构造器的方式注入依赖项，因为能够将依赖项变成为不可变的对象，并确保所需的依赖项不为空。此外，构造器注入的组件总是以完全初始化的状态返回给客户端。顺便说一句，大量的使用构造器来实现依赖注入是一个糟糕的实践，这意味着类可能承载着过多的责任，应该进行重构来进行分离。

setter方式主要用于可选的依赖项，这些依赖项可以在类中分配合理的默认值。否则就需要在代码使用的地方需要进行非空检查。setter注入的好处是，setter方法可以在对象创建完成之后重新配置新的依赖项。因此，通过JMX MBeans进行管理是setter注入的一个值得注意的项目。

有时在处理没有源代码的第三方类时，可以给开发者做出更多选择，例如，如果第三方类不公开任何setter方法，那么构造器方式将是合适的方式。

#### 依赖解析的过程

容器解析bean的依赖项过程如下：

- ApplicationContext被初始化和创建，使用的是配置了所有bean的配置元数据。配置元数据可以通过xml、Java代码和Java注解来实现。
- 对于每个bean的依赖关系，以属性、构造器参数、静态工厂方法参数的形式表示，这些依赖项是在实际创建bean的时候提供的。
- 每个属性或构造器参数都要设置实际的定义，或者是对容器中另一个bean的引用。
- 每个属性或构造器参数的值都会被转换为实际的类型。默认情况下，spring可以将字符串类型转换为基本类型。例如：int、long、string、boolen等。

spring容器在创建容器时验证每个bean的配置。但是，直到实际创建bean时才设置bean属性。单例作用域的bean在创建容器时进行预实例化，否则只有在请求bean时才创建他。创建一个bean可能会导致创建其他bean，因为创建并分配了bean的依赖项或者依赖项的依赖项。请注意，这些依赖项匹配错误可能会延迟出现。

------

**循环依赖**

如果主要使用构造器注入依赖，可能会创建一个不可解析的循环依赖场景。

例如：类A通过构造器注入类B的一个实例，而类B通过构造器注入一个类B的一个实例。如果配置了类A和类B相互注入的bean，spring ioc容器将在运行时检测此循环依赖，并抛出BeanCurrentlyInCreationException异常。

一种解决方法是编辑一些类的源代码，把基于构造器引用的依赖改为基于setter的方式来实现。或者避免使用构造器注入，只使用setter的方式注入。换句话说，尽管不推荐这样做，但是可以使用setter的方式来解决循环依赖项。

与典型的情况不同，bean A和bean B之间的循环依赖强制在初始化完成（鸡和蛋的问题）。

------

开发者通常可以相信spring可以做出正确的事情。会在容器加载的过程中进行检查配置中存在的问题，例如，对不存在的bean进行引用，循环依赖项。spring在创建bean时设置属性并解析依赖项。这意味着，如果在创建对象或其依赖项出现问题，正确加载的spring容器稍后可以在请求对象时产生异常。例如，bean由于缺少或无效属性而抛出异常。某些配置问题的可见性可能会延迟，这就是ApplicationContext默认实现对单例bean实现预实例化的原因。在实际需要这些bean之前创建这些bean需要花费一些时间和内存，会在创建ApplicationContext时发现配置中存在的问题，而不是启动之后发现。开发者可以覆盖此行为，让单例延迟初始化，而不是提前初始化。

如果不存在循环依赖项，那么当一个或多个协作的bean注入到bean中时，每个协作bean在注入之前都已经配置好了。这就意味着，如果Bean A依赖与Bean B，spring ioc容器在调用Bean A上的setter方法之前完全配置了Bean B。换句话说，Bean被实例化，他的依赖项被配置，并且相关的生命周期方法也被调用。

#### 依赖注入示例

下面的示例使用的是基于xml的配置元数据，基于setter的依赖注入。spring xml配置文件的一小部分指定了一些bean定义：

```xml
<bean id="exampleBean" class="examples.ExampleBean">
    <!-- setter injection using the nested ref element -->
    <property name="beanOne">
        <ref bean="anotherExampleBean"/>
    </property>

    <!-- setter injection using the neater ref attribute -->
    <property name="beanTwo" ref="yetAnotherBean"/>
    <property name="integerProperty" value="1"/>
</bean>

<bean id="anotherExampleBean" class="examples.AnotherBean"/>
<bean id="yetAnotherBean" class="examples.YetAnotherBean"/>
```

下面展示了对应的ExampleBean类：

```java
public class ExampleBean {

    private AnotherBean beanOne;

    private YetAnotherBean beanTwo;

    private int i;

    public void setBeanOne(AnotherBean beanOne) {
        this.beanOne = beanOne;
    }

    public void setBeanTwo(YetAnotherBean beanTwo) {
        this.beanTwo = beanTwo;
    }

    public void setIntegerProperty(int i) {
        this.i = i;
    }
}
```

下面展示的是基于构造器实现的依赖注入：

```xml
<bean id="exampleBean" class="examples.ExampleBean">
    <!-- constructor injection using the nested ref element -->
    <constructor-arg>
        <ref bean="anotherExampleBean"/>
    </constructor-arg>

    <!-- constructor injection using the neater ref attribute -->
    <constructor-arg ref="yetAnotherBean"/>

    <constructor-arg type="int" value="1"/>
</bean>

<bean id="anotherExampleBean" class="examples.AnotherBean"/>
<bean id="yetAnotherBean" class="examples.YetAnotherBean"/>
```

下面展示了对应的ExampleBean类：

```java
public class ExampleBean {

    private AnotherBean beanOne;

    private YetAnotherBean beanTwo;

    private int i;

    public ExampleBean(
        AnotherBean anotherBean, YetAnotherBean yetAnotherBean, int i) {
        this.beanOne = anotherBean;
        this.beanTwo = yetAnotherBean;
        this.i = i;
    }
}
```

现在思考这个例子的变体，spring被告知调用一个静态工厂的方法返回对象的一个实例，而不是构造器：

```xml
<bean id="exampleBean" class="examples.ExampleBean" factory-method="createInstance">
    <constructor-arg ref="anotherExampleBean"/>
    <constructor-arg ref="yetAnotherBean"/>
    <constructor-arg value="1"/>
</bean>

<bean id="anotherExampleBean" class="examples.AnotherBean"/>
<bean id="yetAnotherBean" class="examples.YetAnotherBean"/>
```

下面展示了对应的ExampleBean类：

```java
public class ExampleBean {

    // a private constructor
    private ExampleBean(...) {
        ...
    }

    // a static factory method; the arguments to this method can be
    // considered the dependencies of the bean that is returned,
    // regardless of how those arguments are actually used.
    public static ExampleBean createInstance (
        AnotherBean anotherBean, YetAnotherBean yetAnotherBean, int i) {

        ExampleBean eb = new ExampleBean (...);
        // some other operations...
        return eb;
    }
}
```

静态工厂方法的参数由<constructor-arg>元素提供，与实际使用的构造器方式完全相同。工厂方法返回的类型不必与包含静态工厂的类型相同。可以以本质上相同的方式使用实例化的工厂方法，因此这里就不讨论这些细节。

### 1.4.2. 详细的依赖和配置

如上一节所述，可以将bean的属性和构造器参数定义为对其他bean（spring管理的bean）的引用或内联定义。为此，spring基于xml的配置元数据，在<property/>和<constructor-arg/>元素中支持子元素类型。

#### 直接值（基本类型、字符串）

<property/>元素的value属性将构造器参数或属性指定为人们可读的字符串形式。spring的转换服务用于将这些值从string类型转换为实际类型。下面的例子展示了这些功能：

```xml
<bean id="myDataSource" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
    <!-- results in a setDriverClassName(String) call -->
    <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
    <property name="url" value="jdbc:mysql://localhost:3306/mydb"/>
    <property name="username" value="root"/>
    <property name="password" value="misterkaoli"/>
</bean>
```

使用p-namespace进行更简洁的xml配置：

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:p="http://www.springframework.org/schema/p"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
    https://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="myDataSource" class="org.apache.commons.dbcp.BasicDataSource"
        destroy-method="close"
        p:driverClassName="com.mysql.jdbc.Driver"
        p:url="jdbc:mysql://localhost:3306/mydb"
        p:username="root"
        p:password="misterkaoli"/>

</beans>
```

但是错别字不是在设计时发现，而是在运行时发现。除非使用支持自动完成属性的ide工具，强烈推荐使用这种ide的帮助。

也可以配置一个java.util.Properties实例，如下所示：

```xml
<bean id="mappings"
    class="org.springframework.context.support.PropertySourcesPlaceholderConfigurer">

    <!-- typed as a java.util.Properties -->
    <property name="properties">
        <value>
            jdbc.driver.className=com.mysql.jdbc.Driver
            jdbc.url=jdbc:mysql://localhost:3306/mydb
        </value>
    </property>
</bean>
```

spring容器通过使用 JavaBeans `PropertyEditor`机制，实现将<value>元素中的值转换为java.util.Properties实例。这是很好的一种方式，也是spring团队喜欢使用的方式。

**元素idref**

idref将容器中bean的id传递给<constructor-arg>或<property>元素的一种防错方法。如下所示：

```xml
<bean id="theTargetBean" class="..."/>

<bean id="theClientBean" class="...">
    <property name="targetName">
        <idref bean="theTargetBean"/>
    </property>
</bean>
```

上面的bean等价与下边的配置：

```xml
<bean id="theTargetBean" class="..." />

<bean id="client" class="...">
    <property name="targetName" value="theTargetBean"/>
</bean>
```

第一种形式比第二种更可取，因为使用idref标记可以让容器部署的时候验证bean是否存在，第二种则没有这个功能，只有在实际实例化客户端bean时才发现错误。如果客户端是一个原型bean，这个错误将会在很久才能发现。

> 在4.1bean中不在支持idref元素的本地属性xsd，因为他不在提供高于常规bean的引用价值。

idref元素在面向aop编程的ProxyFactoryBean拦截器中比较常见。在指定拦截器名称时使用idref元素可以防止拦截器id拼错。

#### 对其他bean的引用（协作者）

ref元素是<constructor-arg>或<property>定义中的最后一个元素。在这里，可以将一个bean的指定属性值设置为对容器中另外一个bean的引用。被引用的bean需要在设置属性之前进行初始化（如果是个单例，可能已经被容器初始化完成）。所有的引用都是对另外一个对象的引用。作用域和校验取决于指定其他对象的id和名称。

通过ref元素标记的bean是通用的形式，他允许在同一个容器或父容器中创建对任何bean的引用，不用管他们是否在同一个文件中。bean的属性值可能与目标bean的id相同，或者与目标bean的name属性相同。如下所示：

```xml
<ref bean="someBean"/>
```

通过parent属性指定目标bean将创建对当前容器父容器中的bean的引用。父bean的属性值可能与目标bean的id、name属性值相同。目标bean必须位于当前bean的父容器中。如下所示：

```xml
<!-- in the parent context -->
<bean id="accountService" class="com.something.SimpleAccountService">
    <!-- insert dependencies as required here -->
</bean>
```

```xml
<!-- in the child (descendant) context -->
<bean id="accountService" <!-- bean name is the same as the parent bean -->
    class="org.springframework.aop.framework.ProxyFactoryBean">
    <property name="target">
        <ref parent="accountService"/> <!-- notice how we refer to the parent bean -->
    </property>
    <!-- insert other configuration and dependencies as required here -->
</bean>
```

> 在4.0bean xsd中不在支持ref元素的本地值，升级到4.0bean时，将现有的ref更改为ref bean。

#### 内部bean

<constructor-arg>或<property>元素中的<bean>元素定义了一个内部bean，如下所示：

```xml
<bean id="outer" class="...">
    <!-- instead of using a reference to a target bean, simply define the target bean inline -->
    <property name="target">
        <bean class="com.example.Person"> <!-- this is the inner bean -->
            <property name="name" value="Fiona Apple"/>
            <property name="age" value="25"/>
        </bean>
    </property>
</bean>
```

内部bean定义不需要定义id或名字，如果指定了，容器也不会使用。容器在创建的时候也会忽略作用域，因为内部bean总是匿名的，总是用外部bean创建的。不会出现单独访问内部bean，也不可能注入到协作bean中。

【省略了部分内容】

内部bean通常只是简单的共享他们的范围属性。

#### 集合

<list/>, <set/>, <map/> 和 <props/>元素分别设置Java集合类型list、set、map和Properties。如下所示：

```xml
<bean id="moreComplexObject" class="example.ComplexObject">
    <!-- results in a setAdminEmails(java.util.Properties) call -->
    <property name="adminEmails">
        <props>
            <prop key="administrator">administrator@example.org</prop>
            <prop key="support">support@example.org</prop>
            <prop key="development">development@example.org</prop>
        </props>
    </property>
    <!-- results in a setSomeList(java.util.List) call -->
    <property name="someList">
        <list>
            <value>a list element followed by a reference</value>
            <ref bean="myDataSource" />
        </list>
    </property>
    <!-- results in a setSomeMap(java.util.Map) call -->
    <property name="someMap">
        <map>
            <entry key="an entry" value="just some string"/>
            <entry key="a ref" value-ref="myDataSource"/>
        </map>
    </property>
    <!-- results in a setSomeSet(java.util.Set) call -->
    <property name="someSet">
        <set>
            <value>just some string</value>
            <ref bean="myDataSource" />
        </set>
    </property>
</bean>
```

map的键和值或set值可以时下列元素中的任何一个：

```xml
bean | ref | idref | list | set | map | props | value | null
```

**集合的合并**

【有待翻译】

```xml
<beans>
    <bean id="parent" abstract="true" class="example.ComplexObject">
        <property name="adminEmails">
            <props>
                <prop key="administrator">administrator@example.com</prop>
                <prop key="support">support@example.com</prop>
            </props>
        </property>
    </bean>
    <bean id="child" parent="parent">
        <property name="adminEmails">
            <!-- the merge is specified on the child collection definition -->
            <props merge="true">
                <prop key="sales">sales@example.com</prop>
                <prop key="support">support@example.co.uk</prop>
            </props>
        </property>
    </bean>
<beans>
```

```
administrator=administrator@example.com
sales=sales@example.com
support=support@example.co.uk
```

**集合合并的限制**

【有待翻译】

**强制类型的集合**

【有待翻译】

```java
public class SomeClass {

    private Map<String, Float> accounts;

    public void setAccounts(Map<String, Float> accounts) {
        this.accounts = accounts;
    }
}
```

```xml
<beans>
    <bean id="something" class="x.y.SomeClass">
        <property name="accounts">
            <map>
                <entry key="one" value="9.99"/>
                <entry key="two" value="2.75"/>
                <entry key="six" value="3.99"/>
            </map>
        </property>
    </bean>
</beans>
```

#### 字符串是空值和null

spring将属性指定为空视为空字符串，如下所示：

```xml
<bean class="ExampleBean">
    <property name="email" value=""/>
</bean>
```

对应的Java代码：

```java
exampleBean.setEmail("");
```

<null>元素视为空值。如下所示：

```xml
<bean class="ExampleBean">
    <property name="email">
        <null/>
    </property>
</bean>
```

对应的Java代码：

```java
exampleBean.setEmail(null);
```

#### xml快捷方式：p-namespace

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:p="http://www.springframework.org/schema/p"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean name="classic" class="com.example.ExampleBean">
        <property name="email" value="someone@somewhere.com"/>
    </bean>

    <bean name="p-namespace" class="com.example.ExampleBean"
        p:email="someone@somewhere.com"/>
</beans>
```

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:p="http://www.springframework.org/schema/p"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean name="john-classic" class="com.example.Person">
        <property name="name" value="John Doe"/>
        <property name="spouse" ref="jane"/>
    </bean>

    <bean name="john-modern"
        class="com.example.Person"
        p:name="John Doe"
        p:spouse-ref="jane"/>

    <bean name="jane" class="com.example.Person">
        <property name="name" value="Jane Doe"/>
    </bean>
</beans>
```



#### xml快捷方式：c-namespace

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:c="http://www.springframework.org/schema/c"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="beanTwo" class="x.y.ThingTwo"/>
    <bean id="beanThree" class="x.y.ThingThree"/>

    <!-- traditional declaration with optional argument names -->
    <bean id="beanOne" class="x.y.ThingOne">
        <constructor-arg name="thingTwo" ref="beanTwo"/>
        <constructor-arg name="thingThree" ref="beanThree"/>
        <constructor-arg name="email" value="something@somewhere.com"/>
    </bean>

    <!-- c-namespace declaration with argument names -->
    <bean id="beanOne" class="x.y.ThingOne" c:thingTwo-ref="beanTwo"
        c:thingThree-ref="beanThree" c:email="something@somewhere.com"/>

</beans>
```

```xml
<!-- c-namespace index declaration -->
<bean id="beanOne" class="x.y.ThingOne" c:_0-ref="beanTwo" c:_1-ref="beanThree"
    c:_2="something@somewhere.com"/>
```

#### 复合属性名

```xml
<bean id="something" class="things.ThingOne">
    <property name="fred.bob.sammy" value="123" />
</bean>
```

### 1.4.3. `depends-on`的使用

【有待翻译】

```xml
<bean id="beanOne" class="ExampleBean" depends-on="manager"/>
<bean id="manager" class="ManagerBean" />
```

```xml
<bean id="beanOne" class="ExampleBean" depends-on="manager,accountDao">
    <property name="manager" ref="manager" />
</bean>

<bean id="manager" class="ManagerBean" />
<bean id="accountDao" class="x.y.jdbc.JdbcAccountDao" />
```



### 1.4.4. 懒加载bean

【有待翻译】

```xml
<bean id="lazy" class="com.something.ExpensiveToCreateBean" lazy-init="true"/>
<bean name="not.lazy" class="com.something.AnotherBean"/>
```

```xml
<beans default-lazy-init="true">
    <!-- no beans will be pre-instantiated... -->
</beans>
```

### 1.4.5. 自动装配协作者

spring容器可以自动装配bean之间的依赖关系，可以通过检查ApplicationContext中的内容，让spring自动为bean解析协作者。自动装配由如下优势：

- 自动装配可以显著减少属性和构造器参数的配置
- 自动装配可以随着bean对象的发展而更新配置。例如：如果开发者需要向类添加依赖项，那么无需修改配置就可以自动装配该依赖项。因此，自动装配在开发过程中特别有用，而不用在代码库变得更加稳定的时候进行显式配置。

当使用基于xml的配置时，可以使用<bean/>元素的autowire来定义指定自动装配模式。自动装配模式有四种，可以为每个bean指定自动装配，这样就可以选择自动装配的bean。下表描述了四种自动装配模式：

| 模式        | 说明                                                         |
| ----------- | ------------------------------------------------------------ |
| no          | （默认）没有自动装配。bean引用必须使用ref元素定义。对于较大的部署，不建议更改默认的配置，因为显式的指定协作者会提供更大的控制和清晰度。在某种程度上来说，记录了系统的结构。 |
| byName      | 通过属性名进行自动装配。spring寻找与需要自动连接的属性同名的bean。例如：如果一个bean定义被设置为按名称自动装配，并且包含一个master属性（也就是说，有一个对应的setter方法），spring会寻找一个名为master的bean定义，并使用他来配置。 |
| byType      | 如果spring容器中一个bean中只有一个，则允许该属性自动连接。如果存在多个，就会抛出一个异常。如果没有找到匹配的bean，则什么都不会发生。 |
| constructor | 类似于byType，只是用于构造器的参数中。如果容器中没有匹配到bean，就会发生错误。 |

使用byType和constructor模式的自动配置，可以连接数组和类型化集合。在这种情况下，将会匹配所有类型的候选者，来满足依赖关系。如果预期的键类型时string，可以自动装配强类型的map实例。一个自动连接的map实例值由匹配预期类型的所有bean实例组成，map类型的键包含相应的bean名称。

#### 自动装配的局限性和缺点

一致使用自动装配效果最好。如果不经常使用自动装配，只是使用它来装配一两个bean，可能会带来更多的困惑。

思考自动装配的局限性和缺点：

- 使用property和constructor-arg显式配置属性，将会覆盖自动装配。对于基本类型、字符串和简单属性的数组，将不会使用自动装配。这种限制是设计出来的。
- 自动装配不如显式装配精确。尽管如此，spring依旧使用自动装配，spring会小心的避免产生意外的结果（在摸棱两可的情况下）。
- 连接信息无法生成对应的文档。

容器中有多个bean可以匹配到属性和构造器参数时，对于单个值的依赖项，spring对于这种不确定将不容易解决，如果没有唯一的bean可用，就会发生异常。

对于后一种情况，可以有如下选择：

- 放弃自动装配，使用显式配置。
- 通过将bean定义的autowire-candidate属性设置为false，不使用自动配置，下一节将会讲述。
- 通过使用bean定义的primary属性设置为true，将某个bean设置为首选配置。
- 基于注解进行更细的粒度来配置。

#### 从自动装配中排除一个bean

在每个bean的基础上，可以从自动装配中排除一个bean。在基于xml的配置中，将bean元素的autowire-candidate属性设置为false，该bean的自动装配机制将会失效（如果使用注解的形式，如@Autowired）。

> autowire-candidate属性只能影响基于类型的自动装配。不能影响按名称的显式引用，即使指定的bean没有标记为自动装配的候选，也会得到解析。因此，如果按名称匹配，依然会注入一个bean对象。

还可以基于bean名称的模式匹配来限制自动装配的候选者。顶层bean元素在其默认的autowire-candidate属性中接受一个或多个模式。例如：将自动装配的候选状态设置为名称以Repository结尾的bean。提供多个模式下，在一个分隔列表中定义他们。bean定义优先设置autowire-candidate属性的值，对于这种bean，模式匹配规则将不适用。

这是不希望将一个bean注入到其他bean中的技术。这并不意味着被排除的bean本身不能使用自动装配机制。

### 1.4.6 方法注入

在大多数的应用场景中，容器中的大多数bean时单例的。当一个单例bean需要另一个单例bean进行协作，或者一个非单例bean需要另外一个非单例bean协作时，通常通过一个bean定义来依赖另外一个bean处理依赖关系。当bean的生命周期（作用范围）不同时，就会出现问题。假设单例bean A需要使用非单例bean B,可能在A的每个方法调用上。容器只创建一个单例bean A，因此只获得一次设置的机会，容器不会在每次需要bean B新实例时为Bean B提供一个。

一个解决方法是放弃ioc控制。可以通过实现ApplicationContextAware接口，来解决上述的场景，如下所示：

```java
// a class that uses a stateful Command-style class to perform some processing
package fiona.apple;

// Spring-API imports
import org.springframework.beans.BeansException;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;

public class CommandManager implements ApplicationContextAware {

    private ApplicationContext applicationContext;

    public Object process(Map commandState) {
        // grab a new instance of the appropriate Command
        Command command = createCommand();
        // set the state on the (hopefully brand new) Command instance
        command.setState(commandState);
        return command.execute();
    }

    protected Command createCommand() {
        // notice the Spring API dependency!
        return this.applicationContext.getBean("command", Command.class);
    }

    public void setApplicationContext(
            ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }
}
```

这种方法是不可取的，因为业务代码与spring框架进行了耦合。方法注入是spring ioc容器的高级特性，可以很好的处理这个问题。

#### 查找方法注入

查找方法注入是在容器管理的bean上的方法中查找另外一个bean的能力，通常涉及一个原型bean，如上一个场景所述。spring框架通过使用从cglib库生成的字节码来动态的生成覆盖该方法的子类来实现这种方法的注入。

> - 为了使用这个动态子类工作，spring bean容器的子类不能是final类，需要重写的方法也不能是final类型。
> - 对于一个具有抽象方法的类进行测试，需要自己创建该类的子类，并提供该方法的具体实现。
> - 组件扫描也需要具体的方法。
> - 一个限制是，查找方法注入不能与工厂方法一起工作，特别是不能和配置类中的@Bean方法一起工作，因为在这样的情况下，容器不负责创建实例，因此不能动态创建运行时生成的子类。

对于前边的代码，对于CommandManager类，spring容器动态创建覆盖createCommand()方法，CommandManager类没有依赖spring，如下所示：

```java
package fiona.apple;

// no more Spring imports!

public abstract class CommandManager {

    public Object process(Object commandState) {
        // grab a new instance of the appropriate Command interface
        Command command = createCommand();
        // set the state on the (hopefully brand new) Command instance
        command.setState(commandState);
        return command.execute();
    }

    // okay... but where is the implementation of this method?
    protected abstract Command createCommand();
}
```

在包含的注入方法客户端（本例如CommandManager）中，要注入的方法需要如下形式的方法签名：

```
<public|protected> [abstract] <return-type> theMethodName(no-arguments);
```

如果方法是抽象的，框架会动态生成子类的实现方法。否则，动态生成的子类将会覆盖原来的方法内容。思考下面的例子：

```xml
<!-- a stateful bean deployed as a prototype (non-singleton) -->
<bean id="myCommand" class="fiona.apple.AsyncCommand" scope="prototype">
    <!-- inject dependencies here as required -->
</bean>

<!-- commandProcessor uses statefulCommandHelper -->
<bean id="commandManager" class="fiona.apple.CommandManager">
    <lookup-method name="createCommand" bean="myCommand"/>
</bean>
```

标识为commandManager的bean在需要myCommand bean的新实例时调用他自己的createCommand（）方法，必须小心的将myCommand配置为prototype，如果是单例，每次都会返回同一个bean实例。

在基于注解的方式中，可以通过@Lookup注解声明一个查找方法，如下所示：

```java
public abstract class CommandManager {

    public Object process(Object commandState) {
        Command command = createCommand();
        command.setState(commandState);
        return command.execute();
    }

    @Lookup("myCommand")
    protected abstract Command createCommand();
}
```

更加通用的做法是，可以依赖于目标bean根据查找方法声明的返回值类型进行解析：

```java
public abstract class CommandManager {

    public Object process(Object commandState) {
        Command command = createCommand();
        command.setState(commandState);
        return command.execute();
    }

    @Lookup
    protected abstract Command createCommand();
}
```

注意，应该使用具体的实现来声明这种带注解的查找方法，以便与spring组件扫描兼容，因为在默认情况下抽象类会被忽略。这种限制不适用于显式的配置。

> 访问作用域的不同的目标bean另一种方法是ObjectFactory`/ `Provider。请参阅相关章节。
>
> 可能发现org.springframework.beans.factory.config.ServiceLocatorFactoryBean被使用。

#### 任意方法替换

与查找方法相比，方法注入的另外一种较小的用处是，能够用另一种方法实现替换托管bean中的任意方法。可以跳过本章节剩余的部分，直到真正使用到在看也行。

对于基于xml的配置，可以使用replaced-method属性将已经部署的bean现有的方法替换为另一种方法。思考下面的类，来重写一下computeValue方法：

```java
public class MyValueCalculator {

    public String computeValue(String input) {
        // some real code...
    }

    // some other methods...
}
```

org.springframework.beans.factory.support.MethodReplacer接口的实现类提供了一个新方法定义，如下所示：

```java
/**
 * meant to be used to override the existing computeValue(String)
 * implementation in MyValueCalculator
 */
public class ReplacementComputeValue implements MethodReplacer {

    public Object reimplement(Object o, Method m, Object[] args) throws Throwable {
        // get the input value, work with it, and return a computed result
        String input = (String) args[0];
        ...
        return ...;
    }
}
```

这个bean指定原始类的方法，如下所示：

```xml
<bean id="myValueCalculator" class="x.y.z.MyValueCalculator">
    <!-- arbitrary method replacement -->
    <replaced-method name="computeValue" replacer="replacementComputeValue">
        <arg-type>String</arg-type>
    </replaced-method>
</bean>

<bean id="replacementComputeValue" class="a.b.c.ReplacementComputeValue"/>
```

可以在replaced-method元素中使用一个或多个arg-type元素来指定被重写方法的参数。为了方便起见，参数的类型字符串可以使用全限定类型。例如：以下匹配java.lang.String类型：

```
java.lang.String
String
Str
```

## 1.5. bean的范围（作用域）

当创建一个bean定义时，对应创建了一系列的配置信息，用于创建该bean的实例。bean定义的一系列配置十分重要，通过这些配置可以创建许多的对象实例。

不仅可以直接控制插入到bean配置中的各种依赖项和配置信息，还可以控制从特定bean定义创建对象的范围。这种方式强大而灵活，开发者可以通过配置选择创建对象的范围，可以将bean定义为部署在多个作用域中的一个。spring框架支持6个作用域，其中4个是在web应用中才能用到。还支持自定义bean的范围。

支持的作用域如下表所示：

| 范围        | 描述                                                     |
| ----------- | -------------------------------------------------------- |
| singlenton  | （默认）将每个spring ioc容器的一个bean定义创建一个对象。 |
| prototype   | 将一个bean定义创建任意多个对象。                         |
| request     | 将一个bean定义作用于单个http请求中。                     |
| session     | 将一个bean定义作用于http的session中。                    |
| application | 将一个bean定义作用于servletContext中。                   |
| websocket   | 将一个bean定义作用于websocket中。                        |

> 从spring3.0开始，添加了线程范围的作用域，在默认情况下没有注册使用。

### 1.5.1. singlenton范围

只管理一个bean的一个共享实例，对bean的所有请求都使用依赖注入或与该bean定义匹配的id将会一个bean实例。

换句话说，当定义一个bean的作用域是单例时，spring ioc容器将会创建一个bean实例，该单一实例将会存储在缓存中，其他地方引用使用这个单一实例。下图展示了单例作用域是如何工作的：

![singleton](singleton.png)

spring中的单例与设计模式中的单例存在差别，设计模式中的单例硬编码为一个类的实例，每个ClassLoader只创建一个实例。spring的单例是在容器中只创建一个实例。这就说明，在一个spring容器中，定义为单例范围的bean只能创建一个实例。单例范围是spring中默认作用域。基于xml的配置中，可以如下做出配置：

```xml
<bean id="accountService" class="com.something.DefaultAccountService"/>

<!-- the following is equivalent, though redundant (singleton scope is the default) -->
<bean id="accountService" class="com.something.DefaultAccountService" scope="singleton"/>
```

### 1.5.2. prototype范围

bean定义为非单例的原型范围，会导致每次对bean的引用都会创建新的实例。也就是说，该bean被注入到另外一个bean中，或者使用getBean（）方法获取他，每次获取的对象都是新创建的。应该对所有有状态的bean使用原型范围，对无状态的bean使用单例范围。

下图对prototype范围的说明：

![prototype](prototype.png)

数据访问对象通常不会设置为原型，因为典型的dao不包含任何状态。使用单例范围更加适合。

下面展示了基于xml的配置：

```xml
<bean id="accountService" class="com.something.DefaultAccountService" scope="prototype"/>
```

与其他作用范围相比，原型范围没有完整的spring bean的生命周期。容器实例化，配置或以其他的方式组装一个原型对象并将给客户端，而不需要该实例进一步记录其他信息。尽管初始化生命周期回调方法会在所有对象上调用，而不用考虑作用域。在原型范围的情况下，配置销毁生命周期回调方法将不会调用。客户端代码必须清理这些原型范围的对象，并释放原型bean持有的内存资源。如果想让spring释放原型bean的实例资源，可以尝试使用自定义bean post-processor，他持有对需要清理bean的引用。

### 1.5.3. 依赖项是原型bean的单例bean

依赖项是原型bean的单例bean时，依赖关系是在实例化时被解析的，如果将原型bean注入到单例bean时，则会实例化一个新的bean，然后将其注入到单例bean中。

但是，如果单例bean在运行时反复获取原型bean实例，这种情况将不会执行，因为注入只在spring容器初始化的时候注入一次。如果想达到自己想要的效果，可以参看方法注入。

### 1.5.4. Request, Session, Application, 和WebSocket 范围

Request, Session, Application, 和WebSocket范围只能在web容器（XmlWebApplicationContext）中才能正常工作，如果在普通的spring ioc容器中，使用这些范围就会抛出异常（IllegalStateException）。

#### 初始化web配置

在使用这些作用域的bean时，需要进行一下web的初始化配置。

如何完成这些配置取决于所在的servlet环境。

如果在spring web mvc中访问这些作用域，不需要进行特殊的配置，DispatcherServlet已经公开了所有的相关状态。

如果使用的是一个servlet 2.5web容器，请求不在spring mvc中，此时需要注册org.springframework.web.context.request.RequestContextListener。ServletRequestListener是对于servlet3.0+，可以通过使用WebApplicationInitializer接口编程完成。对于旧容器，添加以下声明到web应用程序中。

```xml
<web-app>
    ...
    <listener>
        <listener-class>
            org.springframework.web.context.request.RequestContextListener
        </listener-class>
    </listener>
    ...
</web-app>
```

如果监听器存在问题，可以考虑使用spring的RequestContextFilter。过滤器映射取决于web应用程序的配置，所以必须进行适当的修改。下边显式了一个web应用程序过滤器的部分：

```xml
<web-app>
    ...
    <filter>
        <filter-name>requestContextFilter</filter-name>
        <filter-class>org.springframework.web.filter.RequestContextFilter</filter-class>
    </filter>
    <filter-mapping>
        <filter-name>requestContextFilter</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>
    ...
</web-app>
```

`DispatcherServlet`, `RequestContextListener`, 和RequestContextFilter做的是完全相同的事情，即将http请求对象绑定到服务请求的线程。这使得请求和session作用域在调用链的下游可用。

#### request 范围

思考下边基于xml配置的bean定义：

```xml
<bean id="loginAction" class="com.something.LoginAction" scope="request"/>
```

spring 容器通过为每个http请求使用LoginAction的bean定义来创建新实例。也就是说，LoginAction bean的作用域是在http请求范围内的。可以随心所欲的更改实例中的状态，因为普通的LoginAction bean定义看不到这些状态的变化。当请求处理完成之后，将放弃作用域是request 的bean。

当使用基于注解或Java代码配置时，可以使用@RequestScope注解将组件指定为Request范围。如下所示：

```java
@RequestScope
@Component
public class LoginAction {
    // ...
}
```

#### session 范围

思考下边基于xml配置的bean定义：

```xml
<bean id="userPreferences" class="com.something.UserPreferences" scope="session"/>
```

【原理与requset范围相似】

当使用基于注解或Java代码配置时，可以使用@SessionScope注解将组件指定为session范围。如下所示：

```java
@SessionScope
@Component
public class UserPreferences {
    // ...
}
```

#### application 范围

思考下边基于xml配置的bean定义：

```xml
<bean id="appPreferences" class="com.something.AppPreferences" scope="application"/>
```

spring 容器通过为整个web应用程序使用一次AppPreferences的bean定义来创建实例。也就是说，AppPreferences bean的作用域是servletcontext容器中的，并存储为servletcontext的常规属性，有点与singlenton范围比较像，但是有两个方面有所不同，他是servletcontext中的单例，而不是spring容器中的单例，对于每个spring容器，可能存在多个servletcontext,这些都是公开可见的。

当使用基于注解或Java代码配置时，可以使用@ApplicationScope注解将组件指定为application范围。如下所示：

```java
@ApplicationScope
@Component
public class AppPreferences {
    // ...
}
```

#### WebSocket 范围

详情请参考WebSocket 部分。

#### 带有范围的Bean作为依赖项

spring ioc容器不仅能管理bean对象的实例化，而且还能管理协作bean（依赖项）的连接。如果想将一个http请求范围的bean注入到另外一个时间更长的作用域bean中，可以选择使用aop代理来替代原来的bean。换句话说，需要注入一个代理对象，该对象公开与范围对象相同的公共接口，也可以从相关范围内检索到真正的目标对象，并将方法委托给真正对象。

> 还可以在单例作用域的bean之间使用<aop:scoped-proxy>元素，然后引用一个可以序列化的中间代理，然后在反序列化时重新获得目标单例bean。
>
> 当在原型作用域的bean中使用aop:scoped-proxy元素时，共享代理上的每个方法都会创建一个新的实例，然后转发到目标bean。
>
> 范围代理并不是唯一的解决方法，还可以将注入点声明为ObjectFactory<MyTargetBean>，从而允许getObject()调用在每次需要当前实例时根据需要检索当前实例，而不用单独保存他。
>
> 作为扩展变体，可以声明为bjectProvider<MyTargetBean>，提供了几个额外的访问方法，包括getIfAvailable和getIfUnique。
>
> 此功能的 JSR-330 变体是Provider，并与Provider<MyTargetBean>声明和对应的get()方法调用一起用于检索。

下面展示的例子只有一行配置，但是要理解背后的“why”和“how”十分重要。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/aop
        https://www.springframework.org/schema/aop/spring-aop.xsd">

    <!-- an HTTP Session-scoped bean exposed as a proxy -->
    <bean id="userPreferences" class="com.something.UserPreferences" scope="session">
        <!-- instructs the container to proxy the surrounding bean -->
        <aop:scoped-proxy/> ①
    </bean>

    <!-- a singleton-scoped bean injected with a proxy to the above bean -->
    <bean id="userService" class="com.something.SimpleUserService">
        <!-- a reference to the proxied userPreferences bean -->
        <property name="userPreferences" ref="userPreferences"/>
    </bean>
</beans>
```

①定义代理所在的行。

要创建这样的代理，需要将<aop:scoped-proxy>元素插入到有范围的bean定义中。至于为什么这么做，请考虑下边的单例bean定义，进行对比：

```xml
<bean id="userPreferences" class="com.something.UserPreferences" scope="session"/>

<bean id="userManager" class="com.something.UserManager">
    <property name="userPreferences" ref="userPreferences"/>
</bean>
```

在前面的示例中，单例bean（userManager）被注入了session范围的bean（userPreferences）。需要说明的一点是userManager bean是一个单例（在spring容器中只实例化一次），并且它的依赖项也是注入一次。这就意味着userManager bean只操作配置完整的userPreferences对象实例。

将较短范围的bean注入到较长范围的bean中时，这不是想要的。相反，需要一个单例范围的userManager对象，并且在http会话的生命周期内，有一个http会话范围内的userPreferences对象。因此，容器创建一个对象，该对象与userPreferences类有完全相同的公共接口，可以从范围机制中获取真正的userPreferences对象。容器将这个代理对象注入到userManager bean中，他并不知道userPreferences引用的是一个代理。在这个例子中，当userManager对象调用依赖项userPreferences对象上的一个方法时，实际上是一个被代理的方法，然后代理从真是的userPreferences对象中调用真正的方法。

因此，正确的配置如下所示：

```xml
<bean id="userPreferences" class="com.something.UserPreferences" scope="session">
    <aop:scoped-proxy/>
</bean>

<bean id="userManager" class="com.something.UserManager">
    <property name="userPreferences" ref="userPreferences"/>
</bean>
```

**选择要创建的代理类型**

默认情况下，spring容器将为使用<aop:scoped-proxy>元素标记的bean创建代理时，会创建一个基于cglib的类代理。

> cglib代理只拦截公共方法上的调用。

另外，还可以配置spring容器，通过<aop:scoped-proxy>元素的proxy-target-class属性指定为false，为此类的作用域设置为基于Java jdk接口的标准代理。使用基于jdk接口的代理，不需要添加任何其他库，然而，这也意味着要代理bean类需要至少有一个接口。下面展示了一个基于接口的代理配置：

```xml
<!-- DefaultUserPreferences implements the UserPreferences interface -->
<bean id="userPreferences" class="com.stuff.DefaultUserPreferences" scope="session">
    <aop:scoped-proxy proxy-target-class="false"/>
</bean>

<bean id="userManager" class="com.stuff.UserManager">
    <property name="userPreferences" ref="userPreferences"/>
</bean>
```

更多详细信息，请参考代理机制的部分。

### 1.5.5. 自定义范围

bean的范围机制是可以扩展的，可以自定义范围，甚至可以重新定义已经存在的范围，尽管后者被认为是不好的实践，不能够覆盖内置的singleton和prototype范围。

#### 创建一个自定义范围

为了将自定义的范围集成到spring容器中，需要实现org.springframework.beans.factory.config.Scope接口，要想了解更多关于如何实现自定义范围，请参考spring框架中的实现和相关文档，后者更加详细的介绍了如何实现的方法。

scope接口提供了四个方法，用于操作对象，分别是从范围中获取对象，删除销毁对象。

例如，session范围对获取对象的实现。下面展示的是scope接口中的方法定义：

```java
Object get(String name, ObjectFactory<?> objectFactory)
```

例如，session范围实现删除session范围的bean。如果存在对象则返回，如果不存在，则返回空。下面展示的是scope接口中的方法定义：

```java
Object remove(String name)
```

下面的方法注册了一个回调函数，当对象被销毁，就会调用这个回调函数。

```java
void registerDestructionCallback(String name, Runnable destructionCallback)
```

下边的方法是获取作用域的标识符。

```java
String getConversationId()
```

每个作用域的标识符都不一样，对于session作用域来说，此标识符是：session

#### 使用自定义范围

在编写和测试一个或多个自定义范围之后，需要让spring容器知道新的范围。下面展示了如何向spring容器中注册：

```java
void registerScope(String scopeName, Scope scope);
```

此方法是在ConfigurableBeanFactory接口上声明的，该接口可以获得大多数的spring 容器的配置。registerScope方法的第一个参数是与范围名称相关。spring中已经存在的名称有single和prototype。方法的第二个参数是要注册使用的具体的实现类。

假设编写了自定义范围，然后注册他，如下所示：

> 下边的示例是使用SimpleThreadScope，它包含在spring中，但是默认情况下没有注册。对于自定义范围的使用方法相同。

```java
Scope threadScope = new SimpleThreadScope();
beanFactory.registerScope("thread", threadScope);
```

然后，可以创建符合自定义范围的bean定义，如下所示：

```xml
<bean id="..." class="..." scope="thread">
```

使用自定义范围，不仅局限于编程注册的方式，还可以使用CustomScopeConfigurer类以声明的方式进行注册，如下所示：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/aop
        https://www.springframework.org/schema/aop/spring-aop.xsd">

    <bean class="org.springframework.beans.factory.config.CustomScopeConfigurer">
        <property name="scopes">
            <map>
                <entry key="thread">
                    <bean class="org.springframework.context.support.SimpleThreadScope"/>
                </entry>
            </map>
        </property>
    </bean>

    <bean id="thing2" class="x.y.Thing2" scope="thread">
        <property name="name" value="Rick"/>
        <aop:scoped-proxy/>
    </bean>

    <bean id="thing1" class="x.y.Thing1">
        <property name="thing2" ref="thing2"/>
    </bean>

</beans>
```

## 1.6. 自定义bean的性质

spring框架提供了许多接口，可以使用他们自定义bean的性质，本节的组织如下：

- 生命周期回调
- ApplicationContextAware 和`BeanNameAware`

- 其他装配接口

### 1.6.1.生命周期回调

要与**bean生命周期**的容器管理进行交互，可以实现Spring容器的 InitializingBean和DisposableBean接口。容器为前者调用afterPropertiesSet()，为后者调用destroy()，以便让bean在初始化和销毁bean时执行某些操作。

> JSR-250 @PostConstruct和@PreDestroy注释通常被认为是在现代Spring应用程序中接收生命周期回调的最佳实践。使用这些注解意味着bean没有耦合到特定于spring的接口。详细信息请参见使用@PostConstruct和@PreDestroy。
>
> 如果不想使用JSR-250注解，但仍然希望消除耦合，请考虑基于xml配置的init-method和destroy-method bean定义元数据。

在内部，Spring框架使用BeanPostProcessor接口实现来处理，它能找到的任何回调接口并调用适当的方法。如果您需要定制特性或Spring默认不提供的其他生命周期行为，可以自己实现BeanPostProcessor。有关更多信息，请参见容器扩展点。

除了初始化和销毁回调外，spring管理的对象还可以实现Lifecycle 接口，以便那些对象可以参与由**容器自己的生命周期**驱动的启动和关闭过程。

本节将描述生命周期回调接口。

> 笔记
>
> 生命周期回调分为两类，一类是以bean为主的生命周期回调，一类是以spring容器的生命周期回调。

#### 初始化回调

org.springframework.beans.factory.InitializingBean接口允许Bean，在容器中设置了必要的初始化工作。InitializingBean接口指定了一个简单的方法：

```JAVA
void afterPropertiesSet() throws Exception;
```

我们建议不要使用InitializingBean接口，因为它不必要地将代码耦合到Spring。另外，我们建议使用@PostConstruct注解或指定POJO初始化方法。对于基于xml的配置元数据，可以使用init-method属性指定方法名，该方法的签名为void无参数。通过Java配置，您可以使用@Bean的initMethod属性。

思考下边的示例：

```xml
<bean id="exampleInitBean" class="examples.ExampleBean" init-method="init"/>
```

```java
public class ExampleBean {

    public void init() {
        // do some initialization work
    }
}
```

上面的例子与下面的例子(由两个清单组成)几乎具有相同的效果:

```xml
<bean id="exampleInitBean" class="examples.AnotherExampleBean"/>
```

```java
public class AnotherExampleBean implements InitializingBean {

    @Override
    public void afterPropertiesSet() {
        // do some initialization work
    }
}
```

请注意，前面两个示例中的第一个示例并没有将代码耦合到Spring。

#### 销毁回调

实现org.springframework.beans.factory.DisposableBean接口，当包含bean的spring容器被销毁时，让bean获得一个回调函数。DisposableBean接口指定了一个简单的方法：

```java
void destroy() throws Exception;
```

我们建议您不要使用DisposableBean回调接口，因为它不必要地将代码耦合到Spring中。另外，我们建议使用@PreDestroy注释或指定bean定义支持的泛型方法。使用基于xml的配置元数据，您可以在上使用destroy-method属性。通过Java配置，您可以使用@Bean的destroyMethod属性。

思考下边的示例：

```xml
<bean id="exampleInitBean" class="examples.ExampleBean" destroy-method="cleanup"/>
```

```java
public class ExampleBean {

    public void cleanup() {
        // do some destruction work (like releasing pooled connections)
    }
}
```

上面的例子与下面的例子(由两个清单组成)几乎具有相同的效果:

```xml
<bean id="exampleInitBean" class="examples.AnotherExampleBean"/>
```

```java
public class AnotherExampleBean implements DisposableBean {

    @Override
    public void destroy() {
        // do some destruction work (like releasing pooled connections)
    }
}
```

请注意，前面两个示例中的第一个示例并没有将代码耦合到Spring。

> 您可以为<bean>元素的destroy-method属性分配一个特殊的(推断的)值，它指示Spring自动检测特定bean类上的公共close或shutdown方法。(因此，任何实现java.lang.AutoCloseable或java.io.Closeable的类都可以匹配。)您还可以在<beans>元素的default-destroy-method属性上设置这个特殊的(推断的)值，以便将此行为应用于整个bean集(请参阅默认初始化和销毁方法)。注意，这是Java配置的默认行为。

#### 默认的初始化和销毁方法

当您编写初始化和销毁不使用特定于spring的InitializingBean和DisposableBean回调接口的方法回调时，通常会编写具有init()、initialize()、dispose()等名称的方法。理想情况下，这种生命周期回调方法的名称在整个项目中是标准化的，以便所有开发人员使用相同的方法名称，并确保一致性。

您可以配置Spring容器来“查找”每个bean上的命名初始化和销毁回调方法名。这意味着，作为应用程序开发人员，您可以编写应用程序类并使用名为init()的初始化回调，而不必为每个bean定义配置init-method="init"属性。当创建bean时，Spring IoC容器调用该方法(并且按照前面描述的标准生命周期回调契约)。该特性还强制对初始化和销毁方法回调使用一致的命名约定。

假设初始化回调方法名为init()，销毁回调方法名为destroy()。如下所示：

```java
public class DefaultBlogService implements BlogService {

    private BlogDao blogDao;

    public void setBlogDao(BlogDao blogDao) {
        this.blogDao = blogDao;
    }

    // this is (unsurprisingly) the initialization callback method
    public void init() {
        if (this.blogDao == null) {
            throw new IllegalStateException("The [blogDao] property must be set.");
        }
    }
}
```

然后，就可以在类似于以下的bean中使用该类:

```xml
<beans default-init-method="init">

    <bean id="blogService" class="com.something.DefaultBlogService">
        <property name="blogDao" ref="blogDao" />
    </bean>

</beans>
```

在顶层元素属性上的default-init-method属性会导致Spring IoC容器将bean类上的init方法识别为初始化方法回调。在创建和组装bean时，如果bean类具有这样的方法，则会在适当的时候调用它。

通过在顶层元素上使用default-destroy-method属性，可以类似地配置销毁方法回调(即XML)。

如果现有的bean类已经有了命名与约定不一致的回调方法，则可以通过使用本身的init-method和destroy-method属性指定(即在XML中)方法名来覆盖默认值。

Spring容器保证在提供bean的所有依赖项之后立即调用配置好的初始化回调。因此，在原始bean引用上调用初始化回调，这意味着AOP拦截器等还没有应用到bean。首先完全创建一个目标bean，然后应用一个AOP代理(例如)及其拦截器链。如果目标bean和代理是分别定义的，那么您的代码甚至可以绕过代理与原始的目标bean进行交互。因此，将拦截器应用到init方法是不一致的，因为这样做会将目标bean的生命周期与它的代理或拦截器耦合在一起，并在代码直接与原始目标bean交互时留下奇怪的语义。

#### 生命周期机制的结合

在Spring 2.5中，有三个方式来控制bean的生命周期行为:

- InitializingBean和DisposableBean接口的回调
- 自定义init()和destroy()方法（基于xml的配置）
- @PostConstruct和@PreDestroy注解

可以结合这些方式来控制bean。

> 如果为一个bean配置了多个生命周期机制，并且每个机制都配置了不同的方法名，那么每个配置的方法将按照下面列出的顺序运行。但是，如果为多个生命周期机制配置了相同的方法名称(例如，初始化方法为init())，则该方法只运行一次，如上一节所述。

为同一个bean配置具有不同初始化方法的多个生命周期机制，如下顺序所示:

1.方法上的注解@PostConstruct

2.InitializingBean接口实现的afterPropertiesSet()方法

3.基于xml配置的自定义方法init()



Destroy方法的调用顺序相同:

1.方法上的注解@PreDestroy

2.DisposableBean接口实现的destroy()方法

3.基于xml配置的自定义方法destroy()

#### 启动和关闭回调

Lifecycle接口为任何具有自己生命周期需求的对象(例如启动和停止某些后台进程)定义了基本方法:

```java
public interface Lifecycle {

    void start();

    void stop();

    boolean isRunning();
}
```

任何spring管理的对象都可以实现Lifecycle接口。然后，当ApplicationContext本身接收到启动和停止信号(例如，对于运行时的停止/重启场景)时，它将这些调用级联到该上下文中定义的所有Lifecycle接口的实现。它通过委托给LifecycleProcessor来做到这一点，如下面的清单所示:

```java
public interface LifecycleProcessor extends Lifecycle {

    void onRefresh();

    void onClose();
}
```

注意，LifecycleProcessor接口本身是Lifecycle接口的扩展。它还添加了另外两个方法，用于对刷新和关闭的上下文做出响应。

> 注意，常规的org.springframework.context.Lifecycle接口是用于显式启动和停止通知的普通契约，并不意味着在上下文刷新时自动启动。要对特定bean的自动启动进行细粒度控制(包括启动阶段)，请考虑实现org.springframework.context.SmartLifecycle。
>
> 另外，请注意，停止通知不保证在销毁之前到来。在常规关闭时，所有Lifecycle bean首先会在传播常规销毁回调之前收到停止通知。然而，在上下文生命周期中的热刷新或停止刷新尝试时，只调用destroy方法。

启动和关闭调用的顺序可能很重要。如果任何两个对象之间存在“依赖”关系，则依赖方在其依赖方之后开始，在其依赖方之前停止。然而，有时直接依赖关系是未知的。您可能只知道某一类型的对象应该先于另一类型的对象启动。在这些情况下，SmartLifecycle接口定义了另一个选项，即getPhase()方法，就像它的超级接口phase中定义的那样。如下所示：

```java
public interface Phased {

    int getPhase();
}
```

下面展示的是SmartLifecycle接口的定义：

```java
public interface SmartLifecycle extends Lifecycle, Phased {

    boolean isAutoStartup();

    void stop(Runnable callback);
}
```

启动时，相位最低的对象先启动。当停止时，遵循相反的顺序。因此，实现SmartLifecycle的对象，其getPhase()方法返回Integer。MIN_VALUE将是第一个启动和最后一个停止的。在频谱的另一端，相位值为Integer。MAX_VALUE将指示该对象应该最后启动并首先停止(可能是因为它依赖于正在运行的其他进程)。在考虑阶段值时，还必须知道没有实现SmartLifecycle的任何“正常”生命周期对象的默认阶段为0。因此，任何负的相位值都表明对象应该在那些标准组件之前启动(并在它们之后停止)。相反，对于任何正相位值都是如此。

SmartLifecycle定义的stop方法接受回调。任何实现都必须在该实现的关闭过程完成后调用该回调的run()方法。因为LifecycleProcessor接口的默认实现DefaultLifecycleProcessor等待每个阶段中的对象组的超时值，以调用回调，所以在必要的地方启用异步关闭。默认的每阶段超时时间是30秒。您可以通过在上下文中定义一个名为lifecycleProcessor的bean来覆盖默认的生命周期处理器实例。如果你只想修改超时时间，定义以下代码就足够了:

```xml
<bean id="lifecycleProcessor" class="org.springframework.context.support.DefaultLifecycleProcessor">
    <!-- timeout value in milliseconds -->
    <property name="timeoutPerShutdownPhase" value="10000"/>
</bean>
```

如前所述，LifecycleProcessor接口还定义了刷新和关闭上下文的回调方法。后者驱动关闭过程，就像显式调用了stop()一样，但它发生在上下文关闭时。另一方面，'refresh'回调启用了SmartLifecycle bean的另一个特性。当刷新上下文时(在所有对象都已实例化和初始化之后)，将调用该回调。这时，默认生命周期处理器检查每个SmartLifecycle对象的isAutoStartup()方法返回的布尔值。如果为true，该对象将在此时启动，而不是等待显式调用上下文的或它自己的start()方法(与上下文刷新不同，对于标准上下文实现，上下文启动不会自动发生)。如前所述，阶段值和任何“依赖”关系决定启动顺序。

#### 在非web应用程序中优雅地关闭Spring IoC容器

> 本节仅适用于非web应用。Spring基于web的ApplicationContext实现已经有适当的代码，可以在相关web应用程序关闭时优雅地关闭Spring IoC容器。

如果您在非web应用程序环境中使用Spring的IoC容器(例如，在富客户端桌面环境中)，请向JVM注册一个关闭钩子。这样做可以确保良好的关闭，并在单例bean上调用相关的destroy方法，以便释放所有资源。您仍然必须正确地配置和实现这些销毁回调。

要注册一个关机钩子，调用在ConfigurableApplicationContext接口上声明的registerShutdownHook()方法，如下例所示:

```java
import org.springframework.context.ConfigurableApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public final class Boot {

    public static void main(final String[] args) throws Exception {
        ConfigurableApplicationContext ctx = new ClassPathXmlApplicationContext("beans.xml");

        // add a shutdown hook for the above context...
        ctx.registerShutdownHook();

        // app runs here...

        // main method exits, hook is called prior to the app shutting down...
    }
}
```

### 1.6.2. `ApplicationContextAware` 和`BeanNameAware`

当一个ApplicationContext创建了一个实现了org.springframework.context.ApplicationContextAware接口的对象实例时，该实例会被提供一个对该ApplicationContext的引用。下面的清单显示了ApplicationContextAware接口的定义:

```java
public interface ApplicationContextAware {

    void setApplicationContext(ApplicationContext applicationContext) throws BeansException;
}
```

因此，bean可以通过ApplicationContext接口或将引用转换为该接口的已知子类(例如ConfigurableApplicationContext，它公开了额外的功能)，以编程方式操作创建它们的ApplicationContext。其中一个用途是对其他bean进行编程检索。有时这个功能是有用的。但是，一般情况下，您应该避免它，因为它将代码耦合到Spring，并且不遵循控制反转风格，在这种风格中，协作者作为属性提供给bean。ApplicationContext的其他方法提供对文件资源、发布应用程序事件和访问MessageSource的访问。这些附加特性在ApplicationContext的附加功能中有描述。

自动装配是获取对ApplicationContext的引用的另一种选择。传统的构造函数和byType自动装配模式(如自动装配合作者中所述)可以分别为构造函数参数或setter方法参数提供类型为ApplicationContext的依赖项。要获得更大的灵活性，包括自动装配字段和多个参数方法的能力，请使用基于注释的自动装配特性。如果这样做，则ApplicationContext将自动连接到一个字段、构造函数参数或方法参数中，如果有关的字段、构造函数或方法携带@Autowired注释，则该参数期望ApplicationContext类型。有关更多信息，请参见使用@Autowired。

当ApplicationContext创建一个实现了org.springframework.bean .factory. beannameaware接口的类时，该类提供了一个对其关联对象定义中定义的名称的引用。下面的清单显示了BeanNameAware接口的定义:

```java
public interface BeanNameAware {

    void setBeanName(String name) throws BeansException;
}
```

在填充普通bean属性之后调用回调，但在初始化回调(如InitializingBean.afterPropertiesSet())或自定义初始化方法之前调用回调。

### 1.6.3. 其他的aware接口

除了applicationcontexttaware和BeanNameAware(前面讨论过)之外，Spring还提供了广泛的Aware回调接口，允许bean向容器指示它们需要某种基础设施依赖。作为一般规则，名称表示依赖项类型。下表总结了最重要的Aware接口:

| 名字                           | 注入依赖项                                                   | 所在章节                                                     |
| ------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| ApplicationContextAware        | 声明ApplicationContext                                       | [`ApplicationContextAware` and `BeanNameAware`](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-factory-aware) |
| ApplicationEventPublisherAware | 封闭的ApplicationContext的事件发布者。                       | [Additional Capabilities of the `ApplicationContext`](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#context-introduction) |
| BeanClassLoaderAware           | 用于装入bean类的类装入器。                                   | [Instantiating Beans](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-factory-class) |
| BeanFactoryAware               | 声明BeanFactory                                              | [The `BeanFactory` API](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-beanfactory) |
| BeanNameAware                  | 声明bean的名称。                                             | [`ApplicationContextAware` and `BeanNameAware`](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-factory-aware) |
| LoadTimeWeaverAware            | 已定义编织器，用于在加载时处理类定义。                       | [Load-time Weaving with AspectJ in the Spring Framework](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#aop-aj-ltw) |
| MessageSourceAware             | 为解析消息配置的策略(支持参数化和国际化)。                   | [Additional Capabilities of the `ApplicationContext`](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#context-introduction) |
| NotificationPublisherAware     | Spring JMX通知发布器。                                       | [Notifications](https://docs.spring.io/spring-framework/docs/current/reference/html/integration.html#jmx-notifications) |
| ResourceLoaderAware            | 为对资源的低级访问配置了加载器。                             | [Resources](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#resources) |
| ServletConfigAware             | 容器运行的当前ServletConfig。只在web感知的Spring ApplicationContext中有效。 | [Spring MVC](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc) |
| ServletContextAware            | 容器运行的当前ServletContext。只在web感知的Spring ApplicationContext中有效。 | [Spring MVC](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc) |

再次注意，使用这些接口将您的代码绑定到Spring API，并且不遵循控制反转风格。因此，我们建议将它们用于需要对容器进行编程访问的基础设施bean。

## 1.7. Bean 定义继承

bean定义可以包含大量配置信息，包括构造函数参数、属性值和特定于容器的信息，如初始化方法、静态工厂方法名等。子bean定义从父bean定义继承配置数据。子定义可以覆盖一些值或根据需要添加其他值。使用父bean和子bean定义可以节省大量输入。实际上，这是模板的一种形式。

如果您以编程方式使用ApplicationContext接口，则子bean定义由ChildBeanDefinition类表示。大多数用户不会在这个级别上使用它们。相反，它们在类(如ClassPathXmlApplicationContext)中以声明的方式配置bean定义。当您使用基于xml的配置元数据时，您可以通过使用父属性指定父bean作为该属性的值来指示子bean定义。下面的例子展示了如何做到这一点:

```xml
<bean id="inheritedTestBean" abstract="true"
        class="org.springframework.beans.TestBean">
    <property name="name" value="parent"/>
    <property name="age" value="1"/>
</bean>

<bean id="inheritsWithDifferentClass"
        class="org.springframework.beans.DerivedTestBean"
        parent="inheritedTestBean" init-method="initialize"> ① 
    <property name="name" value="override"/>
    <!-- the age property value of 1 will be inherited from parent -->
</bean>
```

①注意parent属性

如果没有指定，则子bean定义使用来自父定义的bean类，但也可以覆盖它。在后一种情况下，子bean类必须与父类兼容(也就是说，它必须接受父类的属性值)。

子bean定义从父bean继承范围、构造函数参数值、属性值和方法重写，并可以添加新值。您指定的任何范围、初始化方法、销毁方法或静态工厂方法设置都会覆盖相应的父设置。

其余的设置总是取自子定义:依赖、自动连接模式、依赖项检查、单例和延迟初始化。

前面的示例通过使用抽象属性显式地将父bean定义标记为抽象。如果父定义没有指定一个类，则需要显式地将父bean定义标记为抽象，如下面的示例所示:

```xml
<bean id="inheritedTestBeanWithoutClass" abstract="true">
    <property name="name" value="parent"/>
    <property name="age" value="1"/>
</bean>

<bean id="inheritsWithClass" class="org.springframework.beans.DerivedTestBean"
        parent="inheritedTestBeanWithoutClass" init-method="initialize">
    <property name="name" value="override"/>
    <!-- age will inherit the value of 1 from the parent bean definition-->
</bean>
```

父bean不能单独实例化，因为它是不完整的，而且它还被显式地标记为抽象。当定义是抽象的时，它只能作为纯模板bean定义使用，作为子定义的父定义。尝试单独使用这样一个抽象的父bean，将其作为另一个bean的ref属性引用，或使用父bean ID显式地执行getBean()调用，将返回错误。类似地，容器的内部preInstantiateSingletons()方法忽略定义为抽象的bean定义。

> ApplicationContext默认预实例化所有单例。因此，重要的是(至少对于单例bean)，如果您有一个仅打算用作模板的(父)bean定义，并且该定义指定了一个类，那么您必须确保将抽象属性设置为true，否则应用程序上下文将实际(尝试)预实例化抽象bean。

## 1.8. 容器扩展点

