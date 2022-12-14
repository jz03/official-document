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

spring配置中，ioc容器必须管理至少一个bean（通常不止一个）。基于xml的配置将这些bean配置在<bean>的元素中。基于Java的配置通常在@Configuration注解的类中使用@Bean注解的方法。

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

> JSR-250 @PostConstruct和@PreDestroy注解通常被认为是在现代Spring应用程序中接收生命周期回调的最佳实践。使用这些注解意味着bean没有耦合到特定于spring的接口。详细信息请参见使用@PostConstruct和@PreDestroy。
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

我们建议您不要使用DisposableBean回调接口，因为它不必要地将代码耦合到Spring中。另外，我们建议使用@PreDestroy注解或指定bean定义支持的泛型方法。使用基于xml的配置元数据，您可以在上使用destroy-method属性。通过Java配置，您可以使用@Bean的destroyMethod属性。

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

自动装配是获取对ApplicationContext的引用的另一种选择。传统的构造函数和byType自动装配模式(如自动装配合作者中所述)可以分别为构造函数参数或setter方法参数提供类型为ApplicationContext的依赖项。要获得更大的灵活性，包括自动装配字段和多个参数方法的能力，请使用基于注解的自动装配特性。如果这样做，则ApplicationContext将自动连接到一个字段、构造函数参数或方法参数中，如果有关的字段、构造函数或方法携带@Autowired注解，则该参数期望ApplicationContext类型。有关更多信息，请参见使用@Autowired。

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

通常，应用程序开发人员不需要继承ApplicationContext实现类来进行扩展。相反，Spring IoC容器提供了一些特殊的集成接口来实现扩展。接下来的几节将描述这些集成接口。

### 1.8.1. 使用BeanPostProcessor自定义bean

BeanPostProcessor接口定义了您可以实现的回调方法，以提供您自己的(或覆盖容器的默认)**实例化逻辑、依赖关系解析逻辑**等等。如果您想在Spring容器完成bean的实例化、配置和初始化之后实现一些自定义逻辑，也可以插入一个或多个自定义BeanPostProcessor实现。

BeanPostProcessor接口定义了您可以实现的回调方法，以提供您自己的(或覆盖容器的默认)实例化逻辑、依赖关系解析逻辑等等。如果您想在Spring容器完成bean的实例化、配置和初始化之后实现一些自定义逻辑，那么您可以插入一个或多个自定义BeanPostProcessor实现。

您可以配置多个BeanPostProcessor实例，并且可以通过设置order属性来控制这些BeanPostProcessor实例运行的顺序。只有当BeanPostProcessor实现Ordered接口时，才能设置此属性。如果编写自己的BeanPostProcessor，也应该考虑实现Ordered接口。有关更多详细信息，请参阅BeanPostProcessor和Ordered接口的javadoc。另请参见关于BeanPostProcessor实例的编程注册的说明。

> 扩展信息
>
> BeanPostProcessor实例对bean(或对象)实例进行操作。也就是说，Spring IoC容器实例化一个bean实例，然后BeanPostProcessor实例完成它们的工作。
>
> BeanPostProcessor实例的作用域是每个容器。这只与使用容器层次结构有关。如果在一个容器中定义BeanPostProcessor，则后处理器只处理该容器中的bean。换句话说，定义在一个容器中的bean不会被定义在另一个容器中的BeanPostProcessor后处理，即使两个容器都是相同层次结构的一部分。
>
> 要更改实际的bean定义(即定义bean的蓝图)，您需要使用BeanFactoryPostProcessor，如使用BeanFactoryPostProcessor自定义配置元数据中所述。

org.springframework.beans.factory.config.BeanPostProcessor接口恰好包含两个回调方法。当这样的类被注册为容器的后处理器时，对于容器创建的每个bean实例，后处理器在调用容器初始化方法(如InitializingBean.afterPropertiesSet()或任何声明的初始化方法)之前和在任何bean初始化回调之后从容器获得回调。后处理器可以对bean实例采取任何操作，包括完全忽略回调。bean后处理器通常检查回调接口，或者用代理包装bean。一些Spring AOP基础结构类被实现为bean后处理器，以便提供代理包装逻辑。

ApplicationContext自动检测在实现BeanPostProcessor接口的配置元数据中定义的任何bean。ApplicationContext将这些bean注册为后处理器，以便稍后在创建bean时调用它们。Bean后处理器可以以与任何其他Bean相同的方式部署在容器中。

注意，当通过在配置类上使用@Bean工厂方法来声明BeanPostProcessor时，工厂方法的返回类型应该是实现类本身，或者至少是org.springframe.beans.factory .config.BeanPostProcessor接口，这清楚地表明了该bean的后处理器性质。否则，ApplicationContext不能在完全创建它之前根据类型自动检测它。由于BeanPostProcessor需要尽早实例化，以便应用于上下文中其他bean的初始化，因此这种早期的类型检测是至关重要的。

> 以编程方式注册BeanPostProcessor实例
> 虽然BeanPostProcessor注册的推荐方法是通过ApplicationContext自动检测(如前所述)，但您可以通过使用addBeanPostProcessor方法以编程方式向ConfigurableBeanFactory注册它们。当您需要在注册之前评估条件逻辑，或者甚至在层次结构的上下文中复制bean后处理器时，这非常有用。但是请注意，以编程方式添加的BeanPostProcessor实例不尊守Ordered接口。在这里，登记的顺序决定了执行的顺序。还要注意，**以编程方式注册的BeanPostProcessor实例总是在通过自动检测注册的实例之前处理，而不考虑任何显式的顺序。**

> BeanPostProcessor实例和AOP自动代理
> 实现BeanPostProcessor接口的类是特殊的，容器以不同的方式对待它们。它们直接引用的所有BeanPostProcessor实例和bean都在启动时实例化，作为ApplicationContext的特殊启动阶段的一部分。接下来，以排序的方式注册所有BeanPostProcessor实例，并将其应用于容器中的所有其他bean。因为AOP自动代理是作为BeanPostProcessor本身实现的，所以无论是BeanPostProcessor实例还是它们直接引用的bean都不适合进行自动代理，因此，它们没有嵌入方面。
>
> 对于任何这样的bean，您应该看到一条信息日志消息:`Bean someBean is not eligible for getting processed by all BeanPostProcessor interfaces (for example: not eligible for auto-proxying)`.
>
> 如果通过使用自动装配或@Resource(可能会退回到自动装配)将bean连接到BeanPostProcessor中，那么在搜索类型匹配依赖项候选项时，Spring可能会访问意外bean，从而使它们不符合自动代理或其他类型的bean后处理的条件。例如，如果您有一个用@Resource注解的依赖项，其中字段或setter名称不直接对应于bean声明的名称，并且没有使用name属性，那么Spring将访问其他bean以按类型匹配它们。

下面的示例演示如何在ApplicationContext中编写、注册和使用BeanPostProcessor实例。

#### 示例：Hello World, `BeanPostProcessor`-style

第一个示例演示了基本用法。该示例显示了一个自定义BeanPostProcessor实现，该实现在容器创建每个bean时调用其toString()方法，并将结果字符串打印到系统控制台。

下面的清单显示了自定义BeanPostProcessor实现类定义:

```java
package scripting;

import org.springframework.beans.factory.config.BeanPostProcessor;

public class InstantiationTracingBeanPostProcessor implements BeanPostProcessor {

    // simply return the instantiated bean as-is
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        return bean; // we could potentially return any object reference here...
    }

    public Object postProcessAfterInitialization(Object bean, String beanName) {
        System.out.println("Bean '" + beanName + "' created : " + bean.toString());
        return bean;
    }
}
```

下面的bean元素使用了InstantiationTracingBeanPostProcessor:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:lang="http://www.springframework.org/schema/lang"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/lang
        https://www.springframework.org/schema/lang/spring-lang.xsd">
    <!-- 使用groovy语言实现的一个bean定义-->
    <lang:groovy id="messenger"
            script-source="classpath:org/springframework/scripting/groovy/Messenger.groovy">
        <lang:property name="message" value="Fiona Apple Is Just So Dreamy."/>
    </lang:groovy>
    <!--
    when the above bean (messenger) is instantiated, this custom
    BeanPostProcessor implementation will output the fact to the system console
    -->
    <bean class="scripting.InstantiationTracingBeanPostProcessor"/>

</beans>
```

注意如何定义InstantiationTracingBeanPostProcessor。它甚至没有名称，而且因为它是bean，所以可以像其他bean一样对它进行依赖注入。

以下Java应用程序运行上述代码和配置:

```java
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;
import org.springframework.scripting.Messenger;

public final class Boot {

    public static void main(final String[] args) throws Exception {
        ApplicationContext ctx = new ClassPathXmlApplicationContext("scripting/beans.xml");
        Messenger messenger = ctx.getBean("messenger", Messenger.class);
        System.out.println(messenger);
    }

}
```

上述应用程序的输出如下所示:

```
Bean 'messenger' created : org.springframework.scripting.groovy.GroovyMessenger@272961
org.springframework.scripting.groovy.GroovyMessenger@272961
```

#### 示例：`AutowiredAnnotationBeanPostProcessor`

将回调接口或注解与自定义BeanPostProcessor实现结合使用是扩展Spring IoC容器的常用方法。一个例子是Spring的AutowiredAnnotationBeanPostProcessor—一个BeanPostProcessor实现，它附带Spring发行版和自动连接注解字段、setter方法和任意配置方法。

### 1.8.2. 使用BeanFactoryPostProcessor自定义配置元数据

我们查看的下一个扩展点是org.springframework.beans.factory.config.BeanFactoryPostProcessor。该接口的语义与BeanPostProcessor的语义相似，有一个主要区别:BeanFactoryPostProcessor操作bean配置元数据。也就是说，Spring IoC容器允许BeanFactoryPostProcessor读取配置元数据，并可能在容器实例化BeanFactoryPostProcessor实例以外的任何bean之前更改配置元数据。

您可以配置多个BeanFactoryPostProcessor实例，并且可以通过设置order属性来控制这些BeanFactoryPostProcessor实例的运行顺序。但是，如果BeanFactoryPostProcessor实现了Ordered接口，则只能设置此属性。如果编写自己的BeanFactoryPostProcessor，也应该考虑实现Ordered接口。有关更多详细信息，请参阅BeanFactoryPostProcessor和Ordered接口的javadoc。

> 扩展信息
>
> 如果您想要更改实际的bean实例(即从配置元数据创建的对象)，那么您需要使用BeanPostProcessor(前面在使用BeanPostProcessor定制bean中描述过)。虽然在技术上可以在BeanFactoryPostProcessor中使用bean实例(例如，通过使用BeanFactory.getBean())，但这样做会导致过早的bean实例化，违反标准的容器生命周期。这可能会导致负面的副作用，比如绕过bean后处理。
>
> 此外，BeanFactoryPostProcessor实例的作用域是每个容器。这只与使用容器层次结构有关。如果您在一个容器中定义了BeanFactoryPostProcessor，那么它只应用于该容器中的bean定义。一个容器中的Bean定义不会被另一个容器中的BeanFactoryPostProcessor实例后处理，即使两个容器属于同一个层次结构的一部分。

当在ApplicationContext中声明bean工厂后处理器时，将自动运行它，以便对定义容器的配置元数据应用更改。Spring包括许多预定义的bean工厂后处理器，如PropertyOverrideConfigurer和PropertySourcesPlaceholderConfigurer。您还可以使用自定义BeanFactoryPostProcessor—例如，注册自定义属性编辑器。

ApplicationContext自动检测部署到其中的实现BeanFactoryPostProcessor接口的任何bean。它在适当的时候使用这些bean作为bean工厂的后处理器。您可以像部署其他bean一样部署这些后处理器bean。

> 扩展信息
>
> 与BeanPostProcessors一样，通常不希望为延迟初始化配置BeanFactoryPostProcessors。如果没有其他bean引用bean(工厂)后处理器，则该后处理器根本不会被实例化。因此，将其标记为延迟初始化将被忽略，并且即使在<beans></beans>元素的声明中将default-lazy-init属性设置为true, Bean(Factory)PostProcessor也将被急切地实例化。

#### 示例：类名替换`PropertySourcesPlaceholderConfigurer`

您可以使用PropertySourcesPlaceholderConfigurer，通过使用标准Java Properties格式将bean定义中的属性值外部化到单独的文件中。这样做使部署应用程序的人员可以自定义特定于环境的属性，如数据库url和密码，而不必修改容器的主要XML定义文件的复杂性或风险。

考虑以下基于xml的配置元数据片段，其中定义了一个带占位符值的DataSource:

```xml
<bean class="org.springframework.context.support.PropertySourcesPlaceholderConfigurer">
    <property name="locations" value="classpath:com/something/jdbc.properties"/>
</bean>

<bean id="dataSource" destroy-method="close"
        class="org.apache.commons.dbcp.BasicDataSource">
    <property name="driverClassName" value="${jdbc.driverClassName}"/>
    <property name="url" value="${jdbc.url}"/>
    <property name="username" value="${jdbc.username}"/>
    <property name="password" value="${jdbc.password}"/>
</bean>
```

该示例显示了从外部properties文件配置的属性。在运行时，PropertySourcesPlaceholderConfigurer被应用到替换数据源的某些属性的元数据上。要替换的值被指定为表单${property-name}的占位符，它遵循Ant、log4j和JSP EL风格。

实际值来自另一个标准Java属性格式的文件:

```
jdbc.driverClassName=org.hsqldb.jdbcDriver
jdbc.url=jdbc:hsqldb:hsql://production:9002
jdbc.username=sa
jdbc.password=root
```

因此,${jdbc.username}字符串在运行时用值'sa'替换，同样适用于属性文件中匹配键的其他占位符值。PropertySourcesPlaceholderConfigurer检查bean定义的大多数属性和属性中的占位符。此外，还可以自定义占位符前缀和后缀。

通过在Spring 2.5中引入的上下文名称空间，您可以使用专用的配置元素配置属性占位符。你可以在location属性中以逗号分隔的列表形式提供一个或多个位置，如下例所示:

```xml
<context:property-placeholder location="classpath:com/something/jdbc.properties"/>
```

PropertySourcesPlaceholderConfigurer不仅查找您指定的properties文件中的属性。默认情况下，如果在指定的属性文件中找不到属性，它将检查Spring环境属性和常规Java系统属性。

> 说明补充
>
> 您可以使用PropertySourcesPlaceholderConfigurer来替换类名，当您必须在运行时选择特定的实现类时，这有时很有用。下面的例子展示了如何做到这一点:
>
> ```xml
> <bean class="org.springframework.beans.factory.config.PropertySourcesPlaceholderConfigurer">
>     <property name="locations">
>         <value>classpath:com/something/strategy.properties</value>
>     </property>
>     <property name="properties">
>         <value>custom.strategy.class=com.something.DefaultStrategy</value>
>     </property>
> </bean>
> 
> <bean id="serviceStrategy" class="${custom.strategy.class}"/>
> ```
>
> 如果不能在运行时将类解析为有效的类，则在即将创建bean时解析失败，此时是非惰性初始化bean的ApplicationContext的preInstantiateSingletons()阶段。

#### 示例：`PropertyOverrideConfigurer`

PropertyOverrideConfigurer是另一个bean工厂后处理器，它类似于PropertySourcesPlaceholderConfigurer，但与后者不同的是，原始定义可以有bean属性的默认值，或者根本没有值。如果覆盖的Properties文件没有特定bean属性的条目，则使用默认的上下文定义。

请注意，bean定义并不知道正在被覆盖，因此不能立即从XML定义文件中看出正在使用覆盖配置器。如果多个PropertyOverrideConfigurer实例为同一个bean属性定义了不同的值，由于覆盖机制，最后一个会胜出。

属性文件配置行采用以下格式:

```
beanName.property=value
```

下面的清单显示了该格式的一个示例:

```
dataSource.driverClassName=com.mysql.jdbc.Driver
dataSource.url=jdbc:mysql:mydb
```

此示例文件可以与容器定义一起使用，该容器定义包含一个名为dataSource的bean，该bean具有驱动程序和url属性。

复合属性名也被支持，只要路径的每个组件(除了被覆盖的最终属性)都是非空的(大概是由构造函数初始化的)。在下面的例子中，tom bean的bob属性的fred属性的sammy属性被设置为标量值123:

```
tom.fred.bob.sammy=123
```

> 补充说明
>
> 指定的覆盖值总是文字值。它们没有被翻译成bean引用。当XML bean定义中的原始值指定一个bean引用时，此约定也适用。

通过在Spring 2.5中引入的上下文名称空间，可以使用专用的配置元素配置属性覆盖，如下面的示例所示:

```xml
<context:property-override location="classpath:override.properties"/>
```

### 1.8.3. 使用FactoryBean自定义实例化逻辑

您可以为本身就是工厂的对象实现org.springframework.beans.factory.FactoryBean接口。

FactoryBean接口是可插入到Spring IoC容器的实例化逻辑中的一个点。如果您有更适合用Java表示的复杂初始化代码，而不是(可能)冗长的XML，那么您可以创建自己的FactoryBean，在该类中编写复杂的初始化，然后将定制的FactoryBean插入容器中。

FactoryBean<t>接口提供了三个方法:

- `T getObject()`:返回此工厂创建的对象的实例。实例可能是共享的，这取决于该工厂返回的是单例还是原型。
- `boolean isSingleton()`: 如果FactoryBean返回单例则返回true，否则返回false。此方法的默认实现返回true。
- `Class<?> getObjectType()`: 返回getObject()方法返回的对象类型，如果类型事先不知道，则返回null。

FactoryBean的概念和接口在Spring框架中的许多地方都有使用。spring本身附带了FactoryBean接口的50多个实现。

当您需要向容器请求实际的FactoryBean实例本身，而不是它所产生的bean时，在调用ApplicationContext的getBean()方法时，将与符号(&)作为bean id的前缀。因此，对于一个id为myBean的给定FactoryBean，在容器上调用getBean("myBean")将返回FactoryBean的产品，而调用getBean("&myBean")将返回FactoryBean实例本身。

## 1.9. 基于注解的容器配置

> 对于spring配置，注解比xml更好吗？
>
> 基于注解的配置，提出了比xml配置更好的问题。简短的回答是“视情况而定。”较长的答案是每种方法都有其优缺点，通常由开发人员决定哪种策略更适合他们。由于定义方式的不同，**注解在声明中提供了context的大量信息，从而实现更短、更简洁的配置。然而，XML擅长于在不接触组件源代码或重新编译的情况下连接组件**。一些开发人员更喜欢让连接接近源代码，而另一些开发人员则认为带注解的类不再是pojo，而且配置变得分散且更难控制。3
>
> 无论选择什么，Spring都可以适应这两种风格，甚至可以将它们混合在一起。值得指出的是，通过JavaConfig选项，Spring允许以一种非侵入式的方式使用注解，而不涉及目标组件源代码，而且，就工具而言，所有配置样式都由用于Eclipse的Spring Tools支持。

XML设置的另一种选择是基于注解的配置，它依赖于字节码元数据来连接组件，而不是尖括号声明。开发人员不使用XML描述bean连接，而是通过在相关的类、方法或字段声明上使用注解，将配置移动到组件类本身。正如在示例:AutowiredAnnotationBeanPostProcessor中提到的，**将BeanPostProcessor与注解结合使用是扩展Spring IoC容器的常见方法**。例如，Spring 2.0引入了通过@Required注解强制执行必需属性的可能性。Spring 2.5使得遵循同样的一般方法来驱动Spring的依赖注入成为可能。本质上，**@Autowired注解提供了与自动装配合作者中描述的相同的功能，但具有更细粒度的控制和更广泛的适用性。**Spring 2.5还增加了对JSR-250注解的支持，比如@PostConstruct和@PreDestroy。Spring 3.0增加了对javax中包含的JSR-330 (Java依赖注入)注解的支持。注入包，例如@Inject和@Named。有关这些注解的详细信息可以在相关部分中找到。

> 扩展信息
>
> 注解的注入在XML注入之前执行。因此，XML配置将覆盖通过两种方法连接的属性的注解。

和往常一样，您可以将后处理器注册为单独的bean定义，但也可以通过在基于xml的Spring配置中包含以下标记来隐式注册它们(注意上下文名称空间的包含):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        https://www.springframework.org/schema/context/spring-context.xsd">

    <context:annotation-config/>

</beans>
```

<context:annotation-config>元素隐式注册以下后处理器:

- [`ConfigurationClassPostProcessor`](https://docs.spring.io/spring-framework/docs/5.3.23/javadoc-api/org/springframework/context/annotation/ConfigurationClassPostProcessor.html)
- [`AutowiredAnnotationBeanPostProcessor`](https://docs.spring.io/spring-framework/docs/5.3.23/javadoc-api/org/springframework/beans/factory/annotation/AutowiredAnnotationBeanPostProcessor.html)
- [`CommonAnnotationBeanPostProcessor`](https://docs.spring.io/spring-framework/docs/5.3.23/javadoc-api/org/springframework/context/annotation/CommonAnnotationBeanPostProcessor.html)
- [`PersistenceAnnotationBeanPostProcessor`](https://docs.spring.io/spring-framework/docs/5.3.23/javadoc-api/org/springframework/orm/jpa/support/PersistenceAnnotationBeanPostProcessor.html)
- [`EventListenerMethodProcessor`](https://docs.spring.io/spring-framework/docs/5.3.23/javadoc-api/org/springframework/context/event/EventListenerMethodProcessor.html)

> 扩展信息
>
> <context:annotation-config>只在定义它的应用程序context中查找bean上的注解。这意味着，如果您将放在DispatcherServlet的WebApplicationContext中，它只检查controller中的@Autowired bean，而不检查service。有关更多信息，请参阅DispatcherServlet。

### 1.9.1. @Required

@Required注解适用于bean属性setter方法，如下例所示:

```java
public class SimpleMovieLister {

    private MovieFinder movieFinder;

    @Required
    public void setMovieFinder(MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }

    // ...
}
```

此注解指示必须在配置时通过bean定义中的显式属性值或通过自动装配填充受影响的bean属性。如果未填充受影响的bean属性，容器将抛出异常。这允许出现急切的和显式的失败，避免以后出现NullPointerException实例或类似的情况。我们仍然建议您将断言放入bean类本身中(例如，放入init方法中)。这样做会强制执行那些所需的引用和值，即使在容器外部使用该类也是如此。

> 扩展信息
>
> RequiredAnnotationBeanPostProcessor必须注册为bean，以启用对@Required注解的支持。

> 在Spring框架5.1中，**@Required注解和RequiredAnnotationBeanPostProcessor已经正式弃用**，而支持使用构造函数注入进行必要的设置(或者使用InitializingBean.afterPropertiesSet()的自定义实现，或者使用自定义@PostConstruct方法以及bean属性setter方法)。

### 1.9.2. `@Autowired`的使用

> 扩展信息
>
> 在本节包含的例子中，JSR 330的@Inject注解可以用来代替Spring的@Autowired注解。查看这里了解更多细节。

你可以将@Autowired注解应用到构造函数中，如下面的例子所示:

```java
public class MovieRecommender {

    private final CustomerPreferenceDao customerPreferenceDao;

    @Autowired
    public MovieRecommender(CustomerPreferenceDao customerPreferenceDao) {
        this.customerPreferenceDao = customerPreferenceDao;
    }

    // ...
}
```

> 扩展信息
>
> 从Spring Framework 4.3开始，如果目标bean一开始只定义了一个构造函数，那么就不再需要在这样的构造函数上使用@Autowired注解了。但是，如果有几个可用的构造函数，并且没有主/默认构造函数，那么至少必须用@Autowired注解其中一个构造函数，以便指示容器使用哪个构造函数。有关详细信息，请参阅关于构造函数解析的讨论。

你也可以将@Autowired注解应用到传统的setter方法中，如下面的例子所示:

```java
public class SimpleMovieLister {

    private MovieFinder movieFinder;

    @Autowired
    public void setMovieFinder(MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }

    // ...
}
```

您还可以将注解应用于具有任意名称和多个参数的方法，如下例所示:

```java
public class MovieRecommender {

    private MovieCatalog movieCatalog;

    private CustomerPreferenceDao customerPreferenceDao;

    @Autowired
    public void prepare(MovieCatalog movieCatalog,
            CustomerPreferenceDao customerPreferenceDao) {
        this.movieCatalog = movieCatalog;
        this.customerPreferenceDao = customerPreferenceDao;
    }

    // ...
}
```

你也可以将@Autowired应用到字段中，甚至将它与构造函数混合使用，如下面的例子所示:

```java
public class MovieRecommender {

    private final CustomerPreferenceDao customerPreferenceDao;

    @Autowired
    private MovieCatalog movieCatalog;

    @Autowired
    public MovieRecommender(CustomerPreferenceDao customerPreferenceDao) {
        this.customerPreferenceDao = customerPreferenceDao;
    }

    // ...
}
```

> 说明
>
> 确保您的目标组件(例如，MovieCatalog或CustomerPreferenceDao)由用于@ autowired注解注入点的类型一致地声明。否则，由于在运行时出现“未找到类型匹配”错误，注入可能会失败。
>
> 对于通过类路径扫描找到的xml定义的bean或组件类，容器通常预先知道具体类型。但是，对于@Bean工厂方法，您需要确保声明的返回类型具有足够的表达性。对于实现多个接口的组件，或者对于可能由其实现类型引用的组件，请考虑在工厂方法上声明最具体的返回类型(至少与引用bean的注入点所要求的一样具体)。

您还可以通过向期望该类型数组的字段或方法添加@Autowired注解来指示Spring从ApplicationContext中提供所有特定类型的bean，如下面的示例所示:

```java
public class MovieRecommender {

    @Autowired
    private MovieCatalog[] movieCatalogs;

    // ...
}
```

这同样适用于类型化集合，如下例所示:

```java
public class MovieRecommender {

    private Set<MovieCatalog> movieCatalogs;

    @Autowired
    public void setMovieCatalogs(Set<MovieCatalog> movieCatalogs) {
        this.movieCatalogs = movieCatalogs;
    }

    // ...
}
```

> 说明
>
> 目标bean可以实现org.springframework.core.Ordered接口，如果希望数组或列表中的项按特定顺序排序，则可以使用@Order或标准的@Priority注解。否则，它们的顺序将遵循容器中相应目标bean定义的注册顺序。
>
> 您可以在目标类级别和@Bean方法上声明@Order注解，可能针对的是单个bean定义(如果多个定义使用同一个bean类)。@Order值可能会影响注入点的优先级，但请注意，它们不会影响单例启动顺序，这是由依赖关系和@DependsOn声明确定的正交关注。
>
> 注意，标准的javax.annotation.Priority注解在@Bean级别不可用，因为它不能在方法上声明。它的语义可以通过每种类型的单个bean上的@Order值与@Primary相结合来建模。

即使是类型化的Map实例，只要预期的键类型是String，也可以自动连接。映射值包含所有预期类型的bean，键包含相应的bean名称，如下面的示例所示:

```java
public class MovieRecommender {

    private Map<String, MovieCatalog> movieCatalogs;

    @Autowired
    public void setMovieCatalogs(Map<String, MovieCatalog> movieCatalogs) {
        this.movieCatalogs = movieCatalogs;
    }

    // ...
}
```

默认情况下，当给定的注入点没有匹配的候选bean可用时，自动装配失败。在声明数组、集合或映射的情况下，至少需要一个匹配元素。

默认的行为是将带注解的方法和字段视为指示所需的依赖项。你可以改变这种行为，如下例所示，使框架通过将一个不满意的注入点标记为非必需的来跳过它(例如，通过将@Autowired中的required属性设置为false):

```java
public class SimpleMovieLister {

    private MovieFinder movieFinder;

    @Autowired(required = false)
    public void setMovieFinder(MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }

    // ...
}
```

如果一个非必需的方法的依赖项(或者它的一个依赖项，如果有多个参数的话)不可用，则根本不会被调用。在这种情况下，不需要填充非必需字段，只保留其默认值。

注入构造函数和工厂方法参数是一种特殊情况，因为由于Spring的构造函数解析算法可能会处理多个构造函数，@Autowired中的所需属性有一些不同的含义。构造函数和工厂方法参数在默认情况下是有效的，但在单个构造函数场景中有一些特殊的规则，如果没有匹配的bean可用，则解析为空实例的多元素注入点(数组、集合、映射)。这允许一种通用的实现模式，其中所有依赖关系都可以在唯一的多参数构造函数中声明——例如，声明为单个公共构造函数而不使用@Autowired注解。

> 扩展信息
>
> 任何给定bean类中只有一个构造函数可以声明@Autowired，并将所需的属性设置为true，指示构造函数在用作Spring bean时自动连接。因此，如果required属性保留默认值true，则只能用@Autowired注解单个构造函数。如果多个构造函数声明注解，它们都必须声明required=false，以便被视为自动装配的候选对象(类似于XML中的autowire=constructor)。通过匹配Spring容器中的bean可以满足的依赖项数量最多的构造函数将被选择。如果没有一个候选构造函数可以满足，那么将使用主/默认构造函数(如果存在)。类似地，如果一个类声明了多个构造函数，但没有一个用@Autowired注解，那么将使用主/默认构造函数(如果存在)。如果一个类在开始时只声明一个构造函数，那么它将始终被使用，即使没有注解。注意，带注解的构造函数不一定是公共的。
>
> 推荐使用@Autowired的required属性，而不是setter方法上已弃用的@Required注解。将required属性设置为false表示为了自动连接目的不需要该属性，如果不能自动连接该属性，则忽略该属性。另一方面，@Required更强，因为它强制通过容器支持的任何方式设置属性，如果没有定义值，则会引发相应的异常。

另外，您也可以通过Java 8的java.util.Optional来表示特定依赖项的非必需性质。如下例所示:

```java
public class SimpleMovieLister {

    @Autowired
    public void setMovieFinder(Optional<MovieFinder> movieFinder) {
        ...
    }
}
```

从Spring Framework 5.0开始，你也可以使用@Nullable注解(任何包中的任何类型的注解-例如，来自JSR-305的javax.annotation.Nullable):

```java
public class SimpleMovieLister {

    @Autowired
    public void setMovieFinder(@Nullable MovieFinder movieFinder) {
        ...
    }
}
```

您还可以对众所周知的可解析依赖的接口使用@Autowired: BeanFactory、ApplicationContext、Environment、ResourceLoader、ApplicationEventPublisher和MessageSource。这些接口及其扩展接口(如ConfigurableApplicationContext或ResourcePatternResolver)是自动解析的，不需要进行特殊设置。下面的例子自动连接了一个ApplicationContext对象:

```java
public class MovieRecommender {

    @Autowired
    private ApplicationContext context;

    public MovieRecommender() {
    }

    // ...
}
```

> 扩展信息
>
> @Autowired、@Inject、@Value和@Resource注解由Spring BeanPostProcessor实现处理。这意味着您不能在自己的BeanPostProcessor或BeanFactoryPostProcessor类型(如果有的话)中应用这些注解。这些类型必须通过使用XML或Spring @Bean方法显式地“连接”起来。

### 1.9.3. 使用@Primary微调基于注解的自动装配

因为按类型自动装配可能会导致多个候选人，所以通常有必要对选择过程有更多的控制。实现这一点的一种方法是使用Spring的@Primary注解。@Primary表示当多个bean都是自动连接到单值依赖项的候选bean时，应该优先考虑特定的bean。如果在候选bean中只存在一个主bean，则它将成为自动连接的值。

考虑下面的配置，它将firstMovieCatalog定义为主MovieCatalog:

```java
@Configuration
public class MovieConfiguration {

    @Bean
    @Primary
    public MovieCatalog firstMovieCatalog() { ... }

    @Bean
    public MovieCatalog secondMovieCatalog() { ... }

    // ...
}
```

通过前面的配置，下面的MovieRecommender会自动连接firstMovieCatalog:

```java
public class MovieRecommender {

    @Autowired
    private MovieCatalog movieCatalog;

    // ...
}
```

相应的bean定义如下:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        https://www.springframework.org/schema/context/spring-context.xsd">

    <context:annotation-config/>

    <bean class="example.SimpleMovieCatalog" primary="true">
        <!-- inject any dependencies required by this bean -->
    </bean>

    <bean class="example.SimpleMovieCatalog">
        <!-- inject any dependencies required by this bean -->
    </bean>

    <bean id="movieRecommender" class="example.MovieRecommender"/>

</beans>
```

### 1.9.4. 使用@Qualifiers微调基于注解的自动装配

@Primar可以确定一个主候选对象时y是一种有效的方法，可以对多个实例使用按类型自动装配。当您需要对选择过程进行更多控制时，您可以使用Spring的@Qualifier注解。您可以将限定符值与特定的参数关联起来，缩小类型匹配集，以便为每个参数选择特定的bean。在最简单的情况下，这可以是一个简单的描述性值，如下例所示:

```java
public class MovieRecommender {

    @Autowired
    @Qualifier("main")
    private MovieCatalog movieCatalog;

    // ...
}
```

您还可以在单个构造函数参数或方法参数上指定@Qualifier注解，如下例所示:

```java
public class MovieRecommender {

    private MovieCatalog movieCatalog;

    private CustomerPreferenceDao customerPreferenceDao;

    @Autowired
    public void prepare(@Qualifier("main") MovieCatalog movieCatalog,
            CustomerPreferenceDao customerPreferenceDao) {
        this.movieCatalog = movieCatalog;
        this.customerPreferenceDao = customerPreferenceDao;
    }

    // ...
}
```

下面的示例显示了相应的bean定义。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        https://www.springframework.org/schema/context/spring-context.xsd">

    <context:annotation-config/>

    <bean class="example.SimpleMovieCatalog">
        <qualifier value="main"/> ①

        <!-- inject any dependencies required by this bean -->
    </bean>

    <bean class="example.SimpleMovieCatalog">
        <qualifier value="action"/> ②

        <!-- inject any dependencies required by this bean -->
    </bean>

    <bean id="movieRecommender" class="example.MovieRecommender"/>

</beans>
```

①：带有main限定符值的bean与构造函数参数连接具有相同的值。

②：带有action限定符值的bean与构造函数参数连接具有相同的值。

对于备用匹配，bean名称被视为默认限定符值。因此，您可以使用main作为id来定义bean，而不是使用嵌套的限定符元素，从而得到相同的匹配结果。然而，尽管您可以使用这种约定通过名称引用特定bean，但是@Autowired基本上是关于带有可选语义限定符的类型驱动注入。这意味着限定符值，即使使用bean名称备用，**在类型匹配集中也总是具有缩小语义**。它们不从语义上表示对唯一bean id的引用。好的限定符值是main、EMEA、persistent，表示独立于bean id的特定组件的特征，在使用匿名bean定义(如前面示例中的bean)的情况下，bean的id可能是自动生成的。

限定符也适用于类型化集合，如前所述—例如，适用于Set。在本例中，根据声明的限定符，所有匹配的bean都作为集合注入。这意味着限定符不必是唯一的。相反，它们构成了过滤标准。例如，您可以定义多个具有相同限定符值“action”的MovieCatalog bean，所有这些bean都被注入到带有@Qualifier(“action”)注解的Set中。

> 说明
>
> 在类型匹配候选对象中，让限定符值针对目标bean名称进行选择，不需要在注入点上使用@Qualifier注解。如果没有其他解析指示符(例如限定符或主标记)，对于非惟一的依赖关系情况，Spring将根据目标bean名称匹配注入点名称(即字段名称或参数名称)，并选择同名的候选名称(如果有的话)。

也就是说，如果您打算通过名称来表示注解驱动的注入，那么不要主要使用@Autowired，即使它能够在类型匹配候选对象中通过bean名称进行选择。相反，应该使用JSR-250 @Resource注解，该注解在语义上定义为通过惟一名称标识特定的目标组件，声明的类型与匹配过程无关。@Autowired具有截然不同的语义:**在按类型选择候选bean之后，指定的String限定符值只在那些类型选择的候选bean中考虑**(例如，将account限定符与使用相同限定符标签标记的bean相匹配)。

对于本身被定义为集合、Map或数组类型的bean， @Resource是一个很好的解决方案，它通过惟一的名称引用特定的集合或数组bean。也就是说，从4.3开始，您还可以通过Spring的@Autowired类型匹配算法来匹配集合、Map和数组类型，只要元素类型信息保存在@Bean返回类型签名或集合继承层次结构中。在这种情况下，可以使用限定符值在相同类型的集合之间进行选择，如上一段所述。

从4.3开始，@Autowired还考虑了注入的自我引用(也就是说，引用回当前注入的bean)。注意，自我注入是一个备用方案。对其他组件的常规依赖总是具有优先级。从这个意义上说，自我参考不参与常规的候选人选择，因此特别不是主要的。相反，它们总是以最低优先级结束。在实践中，您应该将自引用仅作为最后的手段(例如，通过bean的事务代理调用同一实例上的其他方法)。在这样的场景中，考虑将受影响的方法分解到单独的委托bean中。或者，您可以使用@Resource，它可以通过其惟一的名称获得返回当前bean的代理。

> 尝试在相同的配置类上注入来自@Bean方法的结果实际上也是一个自引用场景。要么在实际需要的方法签名中惰性解析此类引用(与配置类中的自动连接字段相反)，要么将受影响的@Bean方法声明为静态的，将它们与包含的配置类实例及其生命周期解耦。否则，只在备用阶段考虑此类bean，而选择其他配置类上的匹配bean作为主要候选bean(如果可用)。

@Autowired适用于字段、构造函数和多参数方法，允许在参数级别通过限定符注解进行缩小。相比之下，@Resource只支持具有单个参数的字段和bean属性setter方法。因此，如果注入目标是构造函数或多参数方法，则应该坚持使用限定符。

您可以创建自己的自定义限定符注解。为此，定义一个注解并在定义中提供@Qualifier注解，如下例所示:

```java
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface Genre {

    String value();
}
```

然后你可以在自动连接的字段和参数上提供自定义限定符，如下面的例子所示:

```java
public class MovieRecommender {

    @Autowired
    @Genre("Action")
    private MovieCatalog actionCatalog;

    private MovieCatalog comedyCatalog;

    @Autowired
    public void setComedyCatalog(@Genre("Comedy") MovieCatalog comedyCatalog) {
        this.comedyCatalog = comedyCatalog;
    }

    // ...
}
```

接下来，您可以为候选bean定义提供信息。您可以添加<qualifier>标记作为<bean>标记的子元素，然后指定类型和值以匹配您的自定义限定符注解。类型与注解的完全限定类名匹配。另外，如果不存在名称冲突的风险，为了方便，您可以使用短类名。下面的例子演示了这两种方法:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        https://www.springframework.org/schema/context/spring-context.xsd">

    <context:annotation-config/>

    <bean class="example.SimpleMovieCatalog">
        <qualifier type="Genre" value="Action"/>
        <!-- inject any dependencies required by this bean -->
    </bean>

    <bean class="example.SimpleMovieCatalog">
        <qualifier type="example.Genre" value="Comedy"/>
        <!-- inject any dependencies required by this bean -->
    </bean>

    <bean id="movieRecommender" class="example.MovieRecommender"/>

</beans>
```

在类路径扫描和托管组件中，您可以看到以XML提供限定符元数据的基于注解的替代方案。具体地说，请参见使用注解提供限定符元数据。

在某些情况下，使用不带值的注解可能就足够了。当注解服务于更通用的目的，并且可以跨几种不同类型的依赖关系应用时，这可能很有用。例如，您可以提供一个离线目录，在没有可用的Internet连接时可以进行搜索。首先，定义简单注解，如下例所示:

```java
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface Offline {

}
```

然后将注解添加到要自动连接的字段或属性中，如下例所示:

```java
public class MovieRecommender {

    @Autowired
    @Offline ①
    private MovieCatalog offlineCatalog;

    // ...
}
```

① 这一行添加了@Offline注解。

现在，bean定义只需要一个限定符类型，如下例所示:

```xml
<bean class="example.SimpleMovieCatalog">
    <qualifier type="Offline"/> ①
    <!-- inject any dependencies required by this bean -->
</bean>
```

① 该元素指定限定符。

您还可以定义自定义限定符注解，这些注解接受命名属性或取代简单值属性。如果在要自动连接的字段或参数上指定了多个属性值，则bean定义必须匹配所有这样的属性值，才能被视为自动连接候选对象。例如，考虑以下注解定义:

```java
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface MovieQualifier {

    String genre();

    Format format();
}
```

在本例中，Format是一个enum，定义如下:

```java
public enum Format {
    VHS, DVD, BLURAY
}
```

自动连接的字段使用自定义限定符进行注解，并包含两个属性的值:genre和format，如下例所示:

```java
public class MovieRecommender {

    @Autowired
    @MovieQualifier(format=Format.VHS, genre="Action")
    private MovieCatalog actionVhsCatalog;

    @Autowired
    @MovieQualifier(format=Format.VHS, genre="Comedy")
    private MovieCatalog comedyVhsCatalog;

    @Autowired
    @MovieQualifier(format=Format.DVD, genre="Action")
    private MovieCatalog actionDvdCatalog;

    @Autowired
    @MovieQualifier(format=Format.BLURAY, genre="Comedy")
    private MovieCatalog comedyBluRayCatalog;

    // ...
}
```

最后，bean定义应该包含匹配的限定符值。这个例子还演示了您可以使用bean元属性来代替<qualifier>元素。如果可用，<qualifier>元素及其属性优先，但如果没有这样的限定符，自动装配机制将退回到<meta>标记中提供的值，如下例中的最后两个bean定义所示:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        https://www.springframework.org/schema/context/spring-context.xsd">

    <context:annotation-config/>

    <bean class="example.SimpleMovieCatalog">
        <qualifier type="MovieQualifier">
            <attribute key="format" value="VHS"/>
            <attribute key="genre" value="Action"/>
        </qualifier>
        <!-- inject any dependencies required by this bean -->
    </bean>

    <bean class="example.SimpleMovieCatalog">
        <qualifier type="MovieQualifier">
            <attribute key="format" value="VHS"/>
            <attribute key="genre" value="Comedy"/>
        </qualifier>
        <!-- inject any dependencies required by this bean -->
    </bean>

    <bean class="example.SimpleMovieCatalog">
        <meta key="format" value="DVD"/>
        <meta key="genre" value="Action"/>
        <!-- inject any dependencies required by this bean -->
    </bean>

    <bean class="example.SimpleMovieCatalog">
        <meta key="format" value="BLURAY"/>
        <meta key="genre" value="Comedy"/>
        <!-- inject any dependencies required by this bean -->
    </bean>

</beans>
```

### 1.9.5. 使用泛型作为自动装配限定符

除了@Qualifier注解之外，还可以使用Java泛型类型作为一种隐式的限定形式。例如，假设您有以下配置:

```java
@Configuration
public class MyConfiguration {

    @Bean
    public StringStore stringStore() {
        return new StringStore();
    }

    @Bean
    public IntegerStore integerStore() {
        return new IntegerStore();
    }
}
```

假设前面的bean实现了一个泛型接口(即Store<string>和Store<integer>)，您可以@Autowire  Store接口并使用泛型作为限定符，如下面的示例所示:

```java
@Autowired
private Store<String> s1; // <String> qualifier, injects the stringStore bean

@Autowired
private Store<Integer> s2; // <Integer> qualifier, injects the integerStore bean
```

泛型限定符也应用于自动装配列表、Map实例和数组时。下面的例子自动连接了一个泛型List:

```java
// Inject all Store beans as long as they have an <Integer> generic
// Store<String> beans will not appear in this list
@Autowired
private List<Store<Integer>> s;
```

### 1.9.6. 使用`CustomAutowireConfigurer`

CustomAutowireConfigurer是一个BeanFactoryPostProcessor，它允许您注册自己的自定义限定注解类型，即使它们没有使用Spring的@Qualifier进行注解。下面的例子展示了如何使用CustomAutowireConfigurer:

```xml
<bean id="customAutowireConfigurer"
        class="org.springframework.beans.factory.annotation.CustomAutowireConfigurer">
    <property name="customQualifierTypes">
        <set>
            <value>example.CustomQualifier</value>
        </set>
    </property>
</bean>
```

AutowireCandidateResolver通过以下方法确定自动装配候选:

- 每个bean定义的autowire-candidate值
- <beans>元素上可用的任何默认自动连接候选模式
- @Qualifier注解和任何向CustomAutowireConfigurer注册的自定注解的存在

当多个bean有资格成为自动连接候选bean时，“主”的确定如下:如果候选bean中只有一个bean定义的primary 属性设置为true，则选择它。

### 1.9.7. `@Resource`注入

Spring还通过在字段或bean属性setter方法上使用JSR-250 @Resource注解(javax.annotation.Resource)支持注入。这是Java EE中的一种常见模式:例如，在jsf管理的bean和JAX-WS端点中。Spring对Spring管理的对象也支持这种模式。

@Resource接受一个name属性。默认情况下，Spring将该值解释为要注入的bean名称。换句话说，它遵循by-name语义，如下例所示:

```java
public class SimpleMovieLister {

    private MovieFinder movieFinder;

    @Resource(name="myMovieFinder") ①
    public void setMovieFinder(MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }
}
```

①这一行注入一个@Resource。

如果没有显式指定名称，则默认名称从字段name或setter方法派生而来。对于字段，它接受字段名。对于setter方法，它接受bean属性名。下面的例子将把名为movieFinder的bean注入到它的setter方法中:

```JAVA
public class SimpleMovieLister {

    private MovieFinder movieFinder;

    @Resource
    public void setMovieFinder(MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }
}
```

> 扩展信息
>
> 与注解一起提供的名称被ApplicationContext解析为bean名称，CommonAnnotationBeanPostProcessor知道这个名称。如果显式配置Spring的SimpleJndiBeanFactory，则可以通过JNDI解析名称。但是，我们建议您依赖默认行为并使用Spring的JNDI查找功能来保持间接级别。

在没有指定显式名称的@Resource使用的唯一情况下，并且类似于@Autowired， @Resource找到一个主要类型匹配，而不是特定的命名bean，并解决众所周知的可解析依赖关系:BeanFactory、ApplicationContext、ResourceLoader、ApplicationEventPublisher和MessageSource接口。

因此，在下面的例子中，customerPreferenceDao字段首先查找一个名为“customerPreferenceDao”的bean，然后退回到类型customerPreferenceDao的主类型匹配:

```java
public class MovieRecommender {

    @Resource
    private CustomerPreferenceDao customerPreferenceDao;

    @Resource
    private ApplicationContext context; ①

    public MovieRecommender() {
    }

    // ...
}
```

① context字段是基于已知的可解析依赖类型注入的:ApplicationContext。

### 1.9.8. 使用@Value

@Value通常用于注入外部属性:

```java
@Component
public class MovieRecommender {

    private final String catalog;

    public MovieRecommender(@Value("${catalog.name}") String catalog) {
        this.catalog = catalog;
    }
}
```

使用以下配置:

```java
@Configuration
@PropertySource("classpath:application.properties")
public class AppConfig { }
```

以及下面的应用application.properties文件:

```
catalog.name=MovieCatalog
```

在这种情况下，catalog参数和字段将等于MovieCatalog值。

Spring提供了一个默认的宽宏大量的嵌入式值解析器。它将尝试解析属性值，如果无法解析，则将属性名(例如${catalog.name})作为值注入。如果你想对不存在的值保持严格的控制，你应该声明一个PropertySourcesPlaceholderConfigurer bean，如下面的例子所示:

```java
@Configuration
public class AppConfig {

    @Bean
    public static PropertySourcesPlaceholderConfigurer propertyPlaceholderConfigurer() {
        return new PropertySourcesPlaceholderConfigurer();
    }
}
```

> 扩展信息
>
> 当使用JavaConfig配置PropertySourcesPlaceholderConfigurer时，@Bean方法必须是静态的。

如果任何${}占位符无法解析，使用上述配置可确保Spring初始化失败。也可以使用setPlaceholderPrefix、setPlaceholderSuffix或setValueSeparator等方法来定制占位符。

> 扩展信息
>
> Spring Boot默认配置一个PropertySourcesPlaceholderConfigurer bean，它将从application.properties和application.yml文件中获取属性。

Spring提供的内置转换器支持允许自动处理简单的类型转换(例如Integer或int)。多个逗号分隔的值可以自动转换为String数组，无需额外的工作。

可以提供如下的默认值:

```java
@Component
public class MovieRecommender {

    private final String catalog;

    public MovieRecommender(@Value("${catalog.name:defaultCatalog}") String catalog) {
        this.catalog = catalog;
    }
}
```

Spring BeanPostProcessor在幕后使用ConversionService来处理将@Value中的String值转换为目标类型的过程。如果你想为你自己的自定义类型提供转换支持，你可以提供你自己的ConversionService bean实例，如下例所示:

```java
@Configuration
public class AppConfig {

    @Bean
    public ConversionService conversionService() {
        DefaultFormattingConversionService conversionService = new DefaultFormattingConversionService();
        conversionService.addConverter(new MyCustomConverter());
        return conversionService;
    }
}
```

当@Value包含SpEL表达式时，该值将在运行时动态计算，如下例所示:

```java
@Component
public class MovieRecommender {

    private final String catalog;

    public MovieRecommender(@Value("#{systemProperties['user.catalog'] + 'Catalog' }") String catalog) {
        this.catalog = catalog;
    }
}
```

SpEL还支持使用更复杂的数据结构:

```java
@Component
public class MovieRecommender {

    private final Map<String, Integer> countOfMoviesPerCatalog;

    public MovieRecommender(
            @Value("#{{'Thriller': 100, 'Comedy': 300}}") Map<String, Integer> countOfMoviesPerCatalog) {
        this.countOfMoviesPerCatalog = countOfMoviesPerCatalog;
    }
}
```

### 1.9.9. 使用`@PostConstruct` 和 `@PreDestroy`

CommonAnnotationBeanPostProcessor不仅可以识别@Resource注解，还可以识别JSR-250生命周期注解:javax.annotation.PostConstruct和javax.annotation.PreDestroy。在Spring 2.5中引入了对这些注解的支持，为初始化回调和销毁回调中描述的生命周期回调机制提供了另一种选择。假设CommonAnnotationBeanPostProcessor是在Spring ApplicationContext中注册的，那么一个携带这些注解的方法将在生命周期中与相应的Spring生命周期接口方法或显式声明的回调方法相同的位置被调用。在下面的例子中，缓存在初始化时被预填充，在销毁时被清除:

```java
public class CachingMovieLister {

    @PostConstruct
    public void populateMovieCache() {
        // populates the movie cache upon initialization...
    }

    @PreDestroy
    public void clearMovieCache() {
        // clears the movie cache upon destruction...
    }
}
```

有关组合各种生命周期机制的效果的详细信息，请参见组合生命周期机制。

> 扩展信息
>
> 像@Resource一样，@PostConstruct和@PreDestroy注解类型也是JDK 6到8标准Java库的一部分。然而，整个javax.annotation包从JDK 9的核心Java模块中分离出来，最终在JDK 11中被移除。如果需要javax.annotation-api。现在需要通过Maven Central获得工件，只需像其他库一样将其添加到应用程序的类路径中即可。

## 1.10. classpath扫描与组件管理

本章中的大多数例子都使用XML来指定配置元数据，这些元数据在Spring容器中生成每个BeanDefinition。上一节(基于注解的容器配置)演示了如何通过源级注解提供大量配置元数据。然而，即使在这些示例中，“基本”bean定义也显式地定义在XML文件中，而注解仅驱动依赖项注入。本节描述通过扫描类路径隐式检测候选组件的选项。候选组件是与筛选条件匹配的类，并且在容器中注册了相应的bean定义。这样就不需要使用XML来执行bean注册。相反，您可以使用注解(例如，@Component)、AspectJ类型表达式或您自己的自定义筛选条件来选择哪些类具有向容器注册的bean定义。

> 扩展信息
>
> 从Spring 3.0开始，Spring JavaConfig项目提供的许多特性都是Spring框架核心的一部分。这允许您使用Java而不是使用传统的XML文件定义bean。查看@Configuration、@Bean、@Import和@DependsOn注解，了解如何使用这些新特性的示例。

### 1.10.1. `@Component`与进一步构造型注解

@Repository注解是满足存储库角色或原型的任何类的标记(也称为数据访问对象或DAO)。该标记的用途之一是异常的自动翻译，如异常翻译中所述。

Spring提供了更多的原型注解:@Component， @Service和@Controller。@Component是任何spring管理组件的通用原型。@Repository、@Service和@Controller是@Component的特殊化，用于更具体的用例(分别在持久层、服务层和表示层)。因此，您可以使用@Component来注解组件类，但是，通过使用@Repository、@Service或@Controller来注解它们，您的类更适合由工具处理或与方面关联。例如，这些原型注解是切入点的理想目标。@Repository， @Service和@Controller也可以在Spring框架的未来版本中携带额外的语义。因此，如果要为服务层选择使用@Component还是@Service， @Service显然是更好的选择。类似地，如前所述，已经支持@Repository作为持久性层中自动异常转换的标记。

### 1.10.2. 使用元注解与组合注解

Spring提供的许多注解都可以在您自己的代码中用作元注解。元注解是可以应用于另一个注解的注解。例如，前面提到的@Service注解是用@Component进行元注解的，如下面的示例所示:

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component ①
public @interface Service {

    // ...
}
```

① @Component会导致@Service被以与@Component相同的方式对待。

您还可以组合元注解来创建“组合注解”。例如，来自Spring MVC的@RestController注解由@Controller和@ResponseBody组成。

此外，组合注解可以选择性地重新声明元注解的属性，以允许自定义。当您只想公开元注解属性的一个子集时，这可能特别有用。例如，Spring的@SessionScope注解将作用域名称硬编码到会话，但仍然允许自定义proxyMode。下面的清单显示了SessionScope注解的定义:

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Scope(WebApplicationContext.SCOPE_SESSION)
public @interface SessionScope {

    /**
     * Alias for {@link Scope#proxyMode}.
     * <p>Defaults to {@link ScopedProxyMode#TARGET_CLASS}.
     */
    @AliasFor(annotation = Scope.class)
    ScopedProxyMode proxyMode() default ScopedProxyMode.TARGET_CLASS;

}
```

然后你可以使用@SessionScope而不声明proxyMode，如下所示:

```java
@Service
@SessionScope
public class SessionScopedService {
    // ...
}
```

你也可以覆盖proxyMode的值，如下面的例子所示:

```java
@Service
@SessionScope(proxyMode = ScopedProxyMode.INTERFACES)
public class SessionScopedUserService implements UserService {
    // ...
}
```

要了解更多细节，请参见Spring注解编程模型wiki页面。

### 1.10.3. 自动检测类与注册Bean定义

Spring可以自动检测构造型类，并使用ApplicationContext注册相应的BeanDefinition实例。例如，以下两个类有资格进行这种自动检测:

```java
@Service
public class SimpleMovieLister {

    private MovieFinder movieFinder;

    public SimpleMovieLister(MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }
}
```

```java
@Repository
public class JpaMovieFinder implements MovieFinder {
    // 为了清晰起见，省略了实现
}
```

要自动检测这些类并注册相应的bean，您需要将@ComponentScan添加到@Configuration类中，其中basePackages属性是两个类的公共父包。(或者，您可以指定一个逗号或分号或空格分隔的列表，其中包括每个类的父包。)

```java
@Configuration
@ComponentScan(basePackages = "org.example")
public class AppConfig  {
    // ...
}
```

> 扩展信息
>
> 为了简单起见，前面的示例可以使用注解的值属性(即@ComponentScan("org.example"))。

以下是使用XML的替代方案:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        https://www.springframework.org/schema/context/spring-context.xsd">

    <context:component-scan base-package="org.example"/>

</beans>
```

> 说明
>
> 使用<context:component-scan>隐式启用了<context:annotation-config>的功能。在使用<context:component-scan>时，通常不需要包含<context:annotation-config>元素。

> 扩展信息
>
> 扫描类路径包需要在类路径中存在相应的目录条目。当您使用Ant构建JAR时，请确保您没有激活JAR任务的仅文件开关。另外，在某些环境中，类路径目录可能不会根据安全策略公开——例如，JDK 1.7.0_45及更高版本上的独立应用程序(这需要在清单中设置“托管库”——请参阅https://stackoverflow.com/questions/19394570/java-jre-7u45-breaks-classloader-getresources)。
>
> 在JDK 9的模块路径(Jigsaw)上，Spring的类路径扫描通常按预期工作。但是，要确保组件类是在模块信息描述符中导出的。如果您希望Spring调用类的非公共成员，请确保它们是“开放的”(也就是说，它们在模块信息描述符中使用了开放声明而不是导出声明)。

此外，在使用组件扫描元素时，AutowiredAnnotationBeanPostProcessor和CommonAnnotationBeanPostProcessor都是隐式包含的。这意味着两个组件被自动检测并连接在一起—所有这些都不需要XML中提供任何bean配置元数据。

> 扩展信息
>
> 您可以通过包含带有false值的注解配置属性来禁用AutowiredAnnotationBeanPostProcessor和CommonAnnotationBeanPostProcessor的注册。

### 1.10.4. 使用过滤器来进行自定义扫描

默认情况下，使用@Component、@Repository、@Service、@Controller、@Configuration注释的类，或者本身使用@Component注释的自定义注释是唯一被检测到的候选组件。但是，您可以通过应用自定义过滤器来修改和扩展此行为。将它们添加为@ComponentScan注释的includeFilters或excludeFilters属性(或作为XML配置中<context:component-scan>元素的<context:include-filter>或<context:exclude-filter>子元素)。每个筛选器元素都需要类型和表达式属性。过滤选项如下表所示:

| 过滤器类型           | 表达式示例                 | 说明                                              |
| -------------------- | -------------------------- | ------------------------------------------------- |
| annotation (default) | org.example.SomeAnnotation | 在目标组件的类型级别上显示或元显示的注释。        |
| assignable           | org.example.SomeClass      | 目标组件可分配给(扩展或实现)的类(或接口)。        |
| aspectj              | org.example..*Service+     | 目标组件要匹配的AspectJ类型表达式。               |
| regex                | org\.example\.Default.*    | 由目标组件的类名匹配的正则表达式。                |
| custom               | org.example.MyTypeFilter   | 实现org.springframework.core.type.TypeFilter 接口 |

下面的例子展示了忽略所有@Repository注释并使用“stub”repositories 的配置:

```java
@Configuration
@ComponentScan(basePackages = "org.example",
        includeFilters = @Filter(type = FilterType.REGEX, pattern = ".*Stub.*Repository"),
        excludeFilters = @Filter(Repository.class))
public class AppConfig {
    // ...
}
```

下面的清单显示了等效的XML:

```xml
<beans>
    <context:component-scan base-package="org.example">
        <context:include-filter type="regex"
                expression=".*Stub.*Repository"/>
        <context:exclude-filter type="annotation"
                expression="org.springframework.stereotype.Repository"/>
    </context:component-scan>
</beans>
```

> 扩展信息
>
> 您还可以通过在注释上设置useDefaultFilters=false或提供use-default-filters="false"作为<component-scan>元素的属性来禁用默认过滤器。这有效地禁用了使用@Component、@Repository、@Service、@Controller、@RestController或@Configuration注释或元注释的类的自动检测。

### 1.10.5. 在组件中定义Bean元数据

Spring组件还可以向容器提供bean定义元数据。您可以使用与在@Configuration注释类中定义bean元数据相同的@Bean注释来完成此操作。下面的例子展示了如何做到这一点:

```java
@Component
public class FactoryMethodComponent {

    @Bean
    @Qualifier("public")
    public TestBean publicInstance() {
        return new TestBean("publicInstance");
    }

    public void doWork() {
        // Component method implementation omitted
    }
}
```

前面的类是一个Spring组件，它的doWork()方法中有特定于应用程序的代码。但是，它还提供了一个bean定义，其中包含一个引用方法publicInstance()的工厂方法。@Bean注释标识工厂方法和其他bean定义属性，例如通过@Qualifier注释标识的限定符值。其他可以指定的方法级注释有@Scope、@Lazy和自定义限定符注释。

> 说明
>
> 除了组件初始化的作用外，您还可以将@Lazy注释放在标记为@Autowired或@Inject的注入点上。在这种情况下，它会导致插入惰性解析代理。然而，这种代理方法是相当有限的。对于复杂的惰性交互，特别是与可选依赖项相结合时，我们建议改用ObjectProvider<mytargetbean>。

如前所述，支持自动连接字段和方法，还支持@Bean方法的自动装配。下面的例子展示了如何做到这一点:

```java
@Component
public class FactoryMethodComponent {

    private static int i;

    @Bean
    @Qualifier("public")
    public TestBean publicInstance() {
        return new TestBean("publicInstance");
    }

    // use of a custom qualifier and autowiring of method parameters
    @Bean
    protected TestBean protectedInstance(
            @Qualifier("public") TestBean spouse,
            @Value("#{privateInstance.age}") String country) {
        TestBean tb = new TestBean("protectedInstance", 1);
        tb.setSpouse(spouse);
        tb.setCountry(country);
        return tb;
    }

    @Bean
    private TestBean privateInstance() {
        return new TestBean("privateInstance", i++);
    }

    @Bean
    @RequestScope
    public TestBean requestScopedInstance() {
        return new TestBean("requestScopedInstance", 3);
    }
}
```

该示例将String方法参数country自动连接到另一个名为privateInstance的bean上的age属性的值。Spring Expression Language元素通过符号#{< Expression >}定义属性的值。对于@Value注释，表达式解析器被预先配置为在解析表达式文本时查找bean名称。

从Spring Framework 4.3开始，您还可以声明InjectionPoint类型的工厂方法参数(或其更具体的子类:DependencyDescriptor)来访问触发当前bean创建的请求注入点。注意，这只适用于bean实例的实际创建，而不适用于现有实例的注入。因此，这个特性对于原型范围的bean最有意义。对于其他作用域，工厂方法只能看到在给定作用域中触发新bean实例创建的注入点(例如，触发惰性单例bean创建的依赖项)。在这种情况下，您可以使用提供的注入点元数据，并注意语义。下面的例子展示了如何使用InjectionPoint:

```java
@Component
public class FactoryMethodComponent {

    @Bean @Scope("prototype")
    public TestBean prototypeInstance(InjectionPoint injectionPoint) {
        return new TestBean("prototypeInstance for " + injectionPoint.getMember());
    }
}
```

常规Spring组件中的@Bean方法与Spring @Configuration类中的@Bean方法处理方式不同。不同之处在于@Component类没有使用CGLIB进行增强，以拦截方法和字段的调用。CGLIB代理是一种调用@Configuration类中的@Bean方法中的方法或字段来创建对协作对象的bean元数据引用的方法。这些方法不是用正常的Java语义调用的，而是通过容器进行调用，以便提供Spring bean的通常生命周期管理和代理，甚至在通过对@Bean方法的编程调用引用其他bean时也是如此。相比之下，在普通的@Component类中调用@Bean方法中的方法或字段具有标准的Java语义，不应用特殊的CGLIB处理或其他约束。

> 扩展信息
>
> 您可以将@Bean方法声明为静态的，允许在不创建包含它们的配置类作为实例的情况下调用它们。这在定义后处理器bean(例如，类型为BeanFactoryPostProcessor或BeanPostProcessor)时特别有意义，因为这样的bean在容器生命周期的早期被初始化，并且应该避免在那个时候触发配置的其他部分。
>
> 对静态@Bean方法的调用永远不会被容器拦截，甚至在@Configuration类中也不会(如本节前面所述)，这是由于技术限制:CGLIB子类只能覆盖非静态方法。因此，直接调用另一个@Bean方法具有标准的Java语义，导致直接从工厂方法本身返回一个独立的实例。
>
> @Bean方法的Java语言可见性不会对Spring容器中生成的bean定义产生直接影响。您可以自由地在non-@Configuration类中声明您认为合适的工厂方法，也可以在任何地方声明静态方法。然而，@Configuration类中的常规@Bean方法需要被重写——也就是说，它们不能被声明为private或final。
>
> @Bean方法也可以在给定组件或配置类的基类中发现，也可以在Java 8中在组件或配置类实现的接口中声明的默认方法中发现。这允许在组合复杂配置安排时具有很大的灵活性，甚至可以通过Spring 4.2的Java 8默认方法实现多个继承。
>
> 最后，单个类可以为同一个bean保存多个@Bean方法，这是在运行时根据可用的依赖性使用多个工厂方法的安排。这与在其他配置场景中选择“最贪婪的”构造函数或工厂方法的算法相同:在构造时选择具有最多可满足依赖关系的变量，类似于容器如何在多个@Autowired构造函数之间进行选择。

### 1.10.6. 使用名称自动检测组件

当一个组件作为扫描过程的一部分被自动检测到时，它的bean名称是由该扫描器所知道的BeanNameGenerator策略生成的。默认情况下，任何包含名称值的Spring原型注释(@Component， @Repository， @Service，和@Controller)都会为相应的bean定义提供名称。

如果这样的注释不包含名称值，或者不包含任何其他检测到的组件(例如由自定义过滤器发现的组件)，则默认bean名称生成器返回不大写的非限定类名。例如，如果检测到以下组件类，名称将是myMovieLister和movieFinderImpl:

```java
@Service("myMovieLister")
public class SimpleMovieLister {
    // ...
}
```

```java
@Repository
public class MovieFinderImpl implements MovieFinder {
    // ...
}
```

如果不希望依赖默认的bean命名策略，可以提供自定义的bean命名策略。首先，实现BeanNameGenerator接口，并确保包含默认的无参数构造函数。然后，在配置扫描器时提供完全限定的类名，如下面的示例注释和bean定义所示。

> 说明
>
> 如果由于多个自动检测组件具有相同的非限定类名(例如，具有相同名称但驻留在不同包中的类)而导致命名冲突，则可能需要为生成的bean名配置一个默认为完全限定类名的BeanNameGenerator。在Spring Framework 5.2.3中，位于org.springframework.context.annotation包中的fullqualifiedannotationbeannamegenerator可以用于这些目的。

```java
@Configuration
@ComponentScan(basePackages = "org.example", nameGenerator = MyNameGenerator.class)
public class AppConfig {
    // ...
}
```

```xml
<beans>
    <context:component-scan base-package="org.example"
        name-generator="org.example.MyNameGenerator" />
</beans>
```

作为一般规则，当其他组件可能显式引用该名称时，考虑使用注释指定名称。另一方面，当容器负责连接时，自动生成的名称就足够了。

### 1.10.7. 为自动检测组件提供作用域

与spring管理的组件一样，自动检测组件的默认和最常见的作用域是单例的。但是，有时您需要一个不同的范围，可以由@Scope注释指定。您可以在注释中提供作用域的名称，如下面的示例所示:

```java
@Scope("prototype")
@Repository
public class MovieFinderImpl implements MovieFinder {
    // ...
}
```

> 扩展信息
>
> @Scope注释只在具体的bean类(对于带注释的组件)或工厂方法(对于@Bean方法)上进行内省。与XML bean定义相比，没有bean定义继承的概念，类级别的继承层次结构与元数据的目的无关。

有关web特定作用域的详细信息，如Spring上下文中的“请求”或“会话”，请参见Request, Session, Application和WebSocket作用域。与为那些作用域预先构建的注释一样，您也可以使用Spring的元注释方法来组成自己的作用域注释:例如，使用@Scope(“prototype”)进行元注释的自定义注释，也可能声明自定义作用域代理模式。

> 扩展信息
>
> 要为范围解析提供自定义策略，而不是依赖于基于注释的方法，您可以实现ScopeMetadataResolver接口。确保包含一个默认的无参数构造函数。然后，您可以在配置扫描器时提供完全限定的类名，如下面的注释和bean定义示例所示:

```java
@Configuration
@ComponentScan(basePackages = "org.example", scopeResolver = MyScopeResolver.class)
public class AppConfig {
    // ...
}
```

```xml
<beans>
    <context:component-scan base-package="org.example" scope-resolver="org.example.MyScopeResolver"/>
</beans>
```

在使用某些非单例作用域时，可能需要为作用域对象生成代理。推理在作为依赖的作用域bean中进行了描述。为此，可以在组件扫描元素上使用作用域代理属性。取值为no、interfaces和targetClass。例如，在标准JDK动态代理中，如下配置的结果:

```java
@Configuration
@ComponentScan(basePackages = "org.example", scopedProxy = ScopedProxyMode.INTERFACES)
public class AppConfig {
    // ...
}
```

```xml
<beans>
    <context:component-scan base-package="org.example" scoped-proxy="interfaces"/>
</beans>
```

### 1.10.8. 用注解提供限定符元数据

@Qualifier注释将在基于注释的基于qualifier的自动装配中进行微调。那一节中的示例演示了如何使用@Qualifier注释和自定义限定符注释来在解析自动装配候选项时提供细粒度控制。因为这些示例基于XML bean定义，所以通过使用XML中bean元素的限定符或元元素，在候选bean定义上提供了限定符元数据。当依赖于类路径扫描来自动检测组件时，您可以在候选类上提供带有类型级别注释的限定符元数据。下面三个例子演示了这种技术:

```java
@Component
@Qualifier("Action")
public class ActionMovieCatalog implements MovieCatalog {
    // ...
}
```

```java
@Component
@Genre("Action")
public class ActionMovieCatalog implements MovieCatalog {
    // ...
}
```

```java
@Component
@Offline
public class CachingMovieCatalog implements MovieCatalog {
    // ...
}
```

> 扩展信息
>
> 与大多数基于注释的替代方案一样，请记住注释元数据绑定到类定义本身，而XML的使用允许相同类型的多个bean在其限定符元数据中提供变体，因为元数据是按实例而不是按类提供的。

### 1.10.9. 生成候选组件的索引

虽然类路径扫描非常快，但是可以通过在编译时创建静态候选列表来提高大型应用程序的启动性能。在这种模式下，所有作为组件扫描目标的模块都必须使用这种机制。

> 扩展信息
>
> 您现有的@ComponentScan或<context:component-scan>指令必须保持不变，以请求上下文扫描某些包中的候选程序。当ApplicationContext检测到这样一个索引时，它会自动使用它，而不是扫描类路径。

要生成索引，向每个包含组件扫描指令目标组件的模块添加额外的依赖项。下面的例子展示了如何使用Maven实现这一点:

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context-indexer</artifactId>
        <version>5.3.23</version>
        <optional>true</optional>
    </dependency>
</dependencies>
```

spring-context-indexer工件生成一个包含在jar文件中的META-INF/spring.components文件。

> 扩展信息
>
> 在IDE中使用这种模式时，必须将spring-context-indexer注册为注释处理器，以确保在更新候选组件时索引是最新的。

> 说明
>
> 当在类路径上找到META-INF/spring.components文件时，索引会自动启用。如果一个索引对于某些库(或用例)是部分可用的，但不能为整个应用程序构建，你可以通过将spring.index.ignore设置为true(作为JVM系统属性或通过SpringProperties机制)，回到常规的类路径安排(就好像根本不存在索引一样)。

## 1.11. 使用JSR 330标准注解（不常用）

从Spring 3.0开始，Spring提供了对JSR-330标准注释(依赖注入)的支持。这些注释以与Spring注释相同的方式扫描。要使用它们，您需要在类路径中有相关的jar。

> 补充信息
>
> 如果使用maven，可以通过如下方式引入`javax.inject`构件到pom文件中。
>
> ```xml
> <dependency>
>     <groupId>javax.inject</groupId>
>     <artifactId>javax.inject</artifactId>
>     <version>1</version>
> </dependency>
> ```

### 1.11.1. 使用@Inject和@Named注解进行依赖注入

可以使用`@javax.inject.Inject`注解来替代@Autowired注解，如下所示：

```java
import javax.inject.Inject;

public class SimpleMovieLister {

    private MovieFinder movieFinder;

    @Inject
    public void setMovieFinder(MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }

    public void listMovies() {
        this.movieFinder.findMovies(...);
        // ...
    }
}
```

和@Autowired一样，可以在字段级、方法级和构造函数参数级使用@Inject。此外，您可以将注入点声明为提供者，允许按需访问范围较短的bean或通过Provider.get()调用延迟访问其他bean。下面的例子是上一个例子的变体:

```java
import javax.inject.Inject;
import javax.inject.Provider;

public class SimpleMovieLister {

    private Provider<MovieFinder> movieFinder;

    @Inject
    public void setMovieFinder(Provider<MovieFinder> movieFinder) {
        this.movieFinder = movieFinder;
    }

    public void listMovies() {
        this.movieFinder.get().findMovies(...);
        // ...
    }
}
```

如果你想为要注入的依赖项使用限定名，你应该使用@Named注释，如下面的例子所示:

```java
import javax.inject.Inject;
import javax.inject.Named;

public class SimpleMovieLister {

    private MovieFinder movieFinder;

    @Inject
    public void setMovieFinder(@Named("main") MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }

    // ...
}
```

与@Autowired一样，@Inject也可以与java.util.Optional或@Nullable一起使用。这在这里更适用，因为@Inject没有必需的属性。下面的两个例子展示了如何使用@Inject和@Nullable:

```java
public class SimpleMovieLister {

    @Inject
    public void setMovieFinder(Optional<MovieFinder> movieFinder) {
        // ...
    }
}
```

```java
public class SimpleMovieLister {

    @Inject
    public void setMovieFinder(@Nullable MovieFinder movieFinder) {
        // ...
    }
}
```

### 1.11.2. `@Named`与`@ManagedBean`：与`@Component`注解相同

使用`@Named`或`@ManagedBean`可以替换`@Component`注解，如下所示：

```java
import javax.inject.Inject;
import javax.inject.Named;

@Named("movieListener")  // @ManagedBean("movieListener") could be used as well
public class SimpleMovieLister {

    private MovieFinder movieFinder;

    @Inject
    public void setMovieFinder(MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }

    // ...
}
```

使用@Component而不为组件指定名称是很常见的。@Named也可以以类似的方式使用，如下例所示:

```java
import javax.inject.Inject;
import javax.inject.Named;

@Named
public class SimpleMovieLister {

    private MovieFinder movieFinder;

    @Inject
    public void setMovieFinder(MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }

    // ...
}
```

当您使用@Named或@ManagedBean时，您可以使用与使用Spring注释完全相同的方式使用组件扫描，如下面的示例所示:

```java
@Configuration
@ComponentScan(basePackages = "org.example")
public class AppConfig  {
    // ...
}
```

> 补充信息
>
> 与@Component相比，JSR-330 @Named和JSR-250 @ManagedBean注释是不可组合的。您应该使用Spring的原型模型来构建自定义组件注释。

### 1.11.3. JSR-330标准注解的局限性

在使用标准注解时，您应该知道一些重要的特性是不可用的，如下表所示:

| spring              | JSR-330               | 说明                                                         |
| ------------------- | --------------------- | ------------------------------------------------------------ |
| @Autowired          | @Inject               | @Inject没有'required'属性。可以与Java 8的Optional一起使用。  |
| @Component          | @Named / @ManagedBean | JSR-330不提供可组合模型，只提供一种识别命名组件的方法。      |
| @Scope("singleton") | @Singleton            | JSR-330的默认作用域类似于Spring的原型。然而，为了使它与Spring的一般默认值保持一致，在Spring容器中声明的JSR-330 bean在默认情况下是单例的。为了使用作用域而不是单例，你应该使用Spring的@Scope注释。javax。inject还提供了@Scope注释。然而，这个选项仅用于创建您自己的注释。 |
| @Qualifier          | @Qualifier / @Named   | “javax.inject.Qualifier '只是一个用于构建自定义限定符的元注释。具体的字符串限定符(像Spring的带有值的@Qualifier)可以通过javax.inject.Named进行关联。 |
| @Value              |                       |                                                              |
| @Required           |                       |                                                              |
| @Lazy               |                       |                                                              |
| ObjectFactory       | Provider              | javax.inject.Provider是Spring的ObjectFactory的直接替代品，只是有一个更短的get()方法名。它还可以与Spring的@Autowired或与无注释的构造函数和setter方法结合使用。 |

## 1.12. 基于Java的容器配置

本节介绍如何在Java代码中使用注释来配置Spring容器。

### 1.12.1. 基本概念：`@Bean`与 `@Configuration`

Spring新的java配置支持中的核心构件是@ configuration注释类和@ bean注释方法。

@Bean注释用于指示一个方法实例化、配置和初始化一个由Spring IoC容器管理的新对象。对于使用spring的<bean>的xml配置，@Bean注解与<bean>元素有相同的作用。您可以对任何Spring @Component使用@ bean注释的方法。但是，它们最常与@Configuration注解的bean一起使用。

用@Configuration注释一个类表明它的主要用途是作为bean定义的来源。此外，@Configuration类允许通过调用同一个类中的其他@Bean方法来定义bean间的依赖关系。最简单的@Configuration类如下所示:

```java
@Configuration
public class AppConfig {

    @Bean
    public MyService myService() {
        return new MyServiceImpl();
    }
}
```

对等于下边的xml配置

```xml
<beans>
    <bean id="myService" class="com.acme.services.MyServiceImpl"/>
</beans>
```

> 补充
>
> 完整的@Configuration  VS 精简的@Bean 模式？
>
> 当@Bean方法在没有使用@Configuration注释的类中声明时，它们被称为以“lite”模式处理。在@Component中声明的Bean方法，甚至在普通的老类中声明的Bean方法都被认为是“lite”的，包含类的主要目的不同，@Bean方法在那里是一种额外的好处。例如，服务组件可以通过每个适用组件类上的附加@Bean方法向容器公开管理视图。在这种场景中，@Bean方法是一种通用的工厂方法机制。
>
> 与完整的@Configuration不同，lite @Bean方法不能声明bean间的依赖关系。相反，它们对包含它们的组件的内部状态进行操作，并可选地对它们可能声明的参数进行操作。因此，这样的@Bean方法不应该调用其他@Bean方法。每个这样的方法实际上只是特定bean引用的工厂方法，没有任何特殊的运行时语义。这里的积极的副作用是没有CGLIB子类必须在运行时应用，所以在类设计方面没有限制(也就是说，包含的类可能是final类等等)。
>
> 在常见的场景中，@Bean方法要在@Configuration类中声明，以确保始终使用“完整”模式，并且跨方法引用因此被重定向到容器的生命周期管理。这可以防止通过常规Java调用意外地调用相同的@Bean方法，这有助于减少在“精简”模式下操作时难以跟踪的微妙错误。

下面几节将深入讨论@Bean和@Configuration注释。不过，我们首先介绍通过使用基于java的配置创建spring容器的各种方法。

### 1.12.2. 通过`AnnotationConfigApplicationContext`配置spring容器

下面的章节介绍Spring的AnnotationConfigApplicationContext，它是在Spring 3.0中引入的。这个多功能的ApplicationContext实现不仅能够接受@Configuration类作为输入，还能够接受普通的@Component类和用JSR-330元数据注释的类。

当@Configuration类作为输入提供时，@Configuration类本身被注册为bean定义，并且类中所有声明的@Bean方法也被注册为bean定义。

当提供@Component和JSR-330类时，它们被注册为bean定义，并假定在必要时在这些类中使用@Autowired或@Inject等DI元数据。

#### 简单的结构

与实例化ClassPathXmlApplicationContext时使用Spring XML文件作为输入的方式大致相同，您也可以在实例化AnnotationConfigApplicationContext时使用@Configuration类作为输入。这允许Spring容器完全无xml的使用，如下例所示:

```java
public static void main(String[] args) {
    ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
    MyService myService = ctx.getBean(MyService.class);
    myService.doStuff();
}
```

如前所述，AnnotationConfigApplicationContext并不局限于仅使用@Configuration类。任何@Component或JSR-330注释类都可以作为构造函数的输入提供，如下例所示:

```java
public static void main(String[] args) {
    ApplicationContext ctx = new AnnotationConfigApplicationContext(MyServiceImpl.class, Dependency1.class, Dependency2.class);
    MyService myService = ctx.getBean(MyService.class);
    myService.doStuff();
}
```

前面的例子假设MyServiceImpl、Dependency1和Dependency2使用Spring依赖注入注释，例如@Autowired。

#### 通过`register(Class<?>…)`以编程的方式构建容器

您可以使用无参数构造函数实例化AnnotationConfigApplicationContext，然后使用register()方法配置它。当以编程方式构建AnnotationConfigApplicationContext时，这种方法特别有用。下面的例子展示了如何做到这一点:

```java
public static void main(String[] args) {
    AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
    ctx.register(AppConfig.class, OtherConfig.class);
    ctx.register(AdditionalConfig.class);
    ctx.refresh();
    MyService myService = ctx.getBean(MyService.class);
    myService.doStuff();
}
```

#### 使用`scan(String…)`进行组件扫描

要启用组件扫描，你可以像下面这样注释@Configuration类:

```java
@Configuration
@ComponentScan(basePackages = "com.acme") 
public class AppConfig  {
    // ...
}
```

> 说明
>
> 有经验的Spring用户可能熟悉Spring的context:命名空间中的XML声明等价物，如下例所示:
>
> ```xml
> <beans>
>     <context:component-scan base-package="com.acme"/>
> </beans>
> ```

在上面的例子中，com.acme包被扫描以查找任何@ component注释的类，并且这些类被注册为容器中的Spring bean定义。AnnotationConfigApplicationContext公开了scan(String…)方法来允许相同的组件扫描功能，如下面的示例所示:

```java
public static void main(String[] args) {
    AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
    ctx.scan("com.acme");
    ctx.refresh();
    MyService myService = ctx.getBean(MyService.class);
}
```

> 补充信息
>
> 记住，@Configuration类是用@Component进行元注释的，因此它们是组件扫描的候选对象。在前面的例子中，假设AppConfig在com. Acme包中声明(或下面的任何包)，它是在调用scan()时拾取的。在refresh()时，它的所有@Bean方法都被处理并注册为容器中的bean定义。

#### 通过`AnnotationConfigWebApplicationContext`支持web应用

AnnotationConfigApplicationContext的WebApplicationContext变体可通过AnnotationConfigWebApplicationContext获得。您可以在配置Spring ContextLoaderListener servlet监听器、Spring MVC DispatcherServlet等时使用此实现。以下web.xml代码片段配置了一个典型的Spring MVC web应用程序(注意使用了contextClass 、context-param和init-param):

```xml
<web-app>
    <!-- Configure ContextLoaderListener to use AnnotationConfigWebApplicationContext
        instead of the default XmlWebApplicationContext -->
    <context-param>
        <param-name>contextClass</param-name>
        <param-value>
            org.springframework.web.context.support.AnnotationConfigWebApplicationContext
        </param-value>
    </context-param>

    <!-- Configuration locations must consist of one or more comma- or space-delimited
        fully-qualified @Configuration classes. Fully-qualified packages may also be
        specified for component-scanning -->
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>com.acme.AppConfig</param-value>
    </context-param>

    <!-- Bootstrap the root application context as usual using ContextLoaderListener -->
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

    <!-- Declare a Spring MVC DispatcherServlet as usual -->
    <servlet>
        <servlet-name>dispatcher</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <!-- Configure DispatcherServlet to use AnnotationConfigWebApplicationContext
            instead of the default XmlWebApplicationContext -->
        <init-param>
            <param-name>contextClass</param-name>
            <param-value>
                org.springframework.web.context.support.AnnotationConfigWebApplicationContext
            </param-value>
        </init-param>
        <!-- Again, config locations must consist of one or more comma- or space-delimited
            and fully-qualified @Configuration classes -->
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>com.acme.web.MvcConfig</param-value>
        </init-param>
    </servlet>

    <!-- map all requests for /app/* to the dispatcher servlet -->
    <servlet-mapping>
        <servlet-name>dispatcher</servlet-name>
        <url-pattern>/app/*</url-pattern>
    </servlet-mapping>
</web-app>
```

> 补充信息
>
> 对于编程用例，可以使用GenericWebApplicationContext替代AnnotationConfigWebApplicationContext。详细信息请参见GenericWebApplicationContext javadoc。

### 1.12.3. 使用@Bean注解

@Bean是一个方法级注释，是XML bean元素的直接类比。注释支持bean提供的一些属性，例如:

- init-method
- destroy-method
- autowiring
- name

您可以在@ configuration注释类或@ component注释类中使用@Bean注释。

#### 声明一个bean

要声明bean，可以使用@Bean注释方法。您可以使用此方法在指定为方法返回值的类型向ApplicationContext容器中注册bean定义。默认情况下，bean名称与方法名称相同。下面的例子展示了@Bean方法声明:

```java
@Configuration
public class AppConfig {

    @Bean
    public TransferServiceImpl transferService() {
        return new TransferServiceImpl();
    }
}
```

上面的配置与下面的Spring XML完全等价:

```xml
<beans>
    <bean id="transferService" class="com.acme.TransferServiceImpl"/>
</beans>
```

这两个声明使一个名为transferService的bean在ApplicationContext中可用，绑定到TransferServiceImpl类型的对象实例，如下面的文本图像所示:

```
transferService -> com.acme.TransferServiceImpl
```

您还可以使用默认方法来定义bean。这允许通过在默认方法上实现带有bean定义的接口来组合bean配置。如下所示：

```java
public interface BaseConfig {

    @Bean
    default TransferServiceImpl transferService() {
        return new TransferServiceImpl();
    }
}

@Configuration
public class AppConfig implements BaseConfig {

}
```

你也可以用接口(或基类)返回类型来声明@Bean方法，如下例所示:

```java
@Configuration
public class AppConfig {

    @Bean
    public TransferService transferService() {
        return new TransferServiceImpl();
    }
}
```

但是，这将预先类型预测的可见性限制为指定的接口类型(TransferService)。然后，只有在实例化了受影响的单例bean之后，容器才知道完整的类型(TransferServiceImpl)。非惰性单例bean根据它们的声明顺序进行实例化，因此您可能会看到不同的类型匹配结果，这取决于另一个组件何时尝试通过非声明类型进行匹配(例如@Autowired TransferServiceImpl，它只在transferService bean被实例化之后进行解析)。

> 说明
>
> 如果您始终通过声明的服务接口引用您的类型，那么您的@Bean返回类型可以安全地加入到设计决策中。但是，对于实现多个接口的组件，或者对于可能由其实现类型引用的组件，声明最具体的返回类型(至少与引用您的bean的注入点所要求的一样具体)更安全。

#### bean之间的依赖

带有@ bean注释的方法可以具有任意数量的参数，用于描述构建该bean所需的依赖项。例如，如果我们的TransferService需要一个AccountRepository，我们可以用一个方法参数具体化这个依赖，如下面的例子所示:

```java
@Configuration
public class AppConfig {

    @Bean
    public TransferService transferService(AccountRepository accountRepository) {
        return new TransferServiceImpl(accountRepository);
    }
}
```

解析机制与基于构造函数的依赖注入非常相似。有关更多细节，请参阅相关部分。

#### 接受声明周期回调

任何用@Bean注释定义的类都支持常规生命周期回调，并且可以使用JSR-250中的@PostConstruct和@PreDestroy注释。更多细节请参见JSR-250注释。

常规的Spring生命周期回调也得到了充分的支持。如果bean实现了InitializingBean、DisposableBean或Lifecycle，容器将调用它们各自的方法。

也完全支持*Aware接口的标准集合(例如BeanFactoryAware, BeanNameAware, MessageSourceAware, ApplicationContextAware，等等)。

@Bean注释支持指定任意初始化和销毁回调方法，很像Spring XML在bean元素上的init-method和destroy-method属性，如下例所示:

```java
public class BeanOne {

    public void init() {
        // initialization logic
    }
}

public class BeanTwo {

    public void cleanup() {
        // destruction logic
    }
}

@Configuration
public class AppConfig {

    @Bean(initMethod = "init")
    public BeanOne beanOne() {
        return new BeanOne();
    }

    @Bean(destroyMethod = "cleanup")
    public BeanTwo beanTwo() {
        return new BeanTwo();
    }
}
```

> 补充信息
>
> 默认情况下，使用Java配置定义的具有公共关闭或关闭方法的bean会自动使用销毁回调。如果您有一个公共关闭或关闭方法，并且您不希望在容器关闭时调用它，那么您可以在bean定义中添加@Bean(destroyMethod="")，以禁用默认(推断)模式。
>
> 默认情况下，您可能希望对使用JNDI获取的资源这样做，因为它的生命周期是在应用程序之外管理的。特别是，要确保总是为DataSource这样做，因为众所周知，在Java EE应用程序服务器上这是有问题的。
>
> 下面的例子展示了如何防止DataSource的自动销毁回调:
>
> ```java
> @Bean(destroyMethod="")
> public DataSource dataSource() throws NamingException {
>     return (DataSource) jndiTemplate.lookup("MyDS");
> }
> ```
>
> 同样，对于@Bean方法，您通常使用编程的JNDI查找，要么使用Spring的JndiTemplate或JndiLocatorDelegate帮助程序，要么直接使用JNDI InitialContext，但不使用JndiObjectFactoryBean变量(这将迫使您将返回类型声明为FactoryBean类型，而不是实际的目标类型，使它在其他打算引用此处提供的资源的@Bean方法中难以使用交叉引用调用)。

在上述例子中的BeanOne中，在构造过程中直接调用init()方法同样有效，如下例所示:

```java
@Configuration
public class AppConfig {

    @Bean
    public BeanOne beanOne() {
        BeanOne beanOne = new BeanOne();
        beanOne.init();
        return beanOne;
    }

    // ...
}
```

当您直接使用Java工作时，您可以对对象做任何您想做的事情，而不总是需要依赖容器生命周期。

#### 指定bean的作用域

Spring包含@Scope注释，以便您可以指定bean的范围。

##### 使用@Scope注解

您可以指定使用@Bean注释定义的bean应该具有特定的作用域。您可以使用Bean作用域一节中指定的任何标准作用域。

默认作用域是单例的，但是你可以用@Scope注释来覆盖它，如下面的例子所示:

```java
@Configuration
public class MyConfiguration {

    @Bean
    @Scope("prototype")
    public Encryptor encryptor() {
        // ...
    }
}
```

##### @Scope与scoped-proxy

Spring提供了一种通过作用域代理处理作用域依赖项的方便方法。在使用XML配置时创建这种代理的最简单方法是<aop:scoped-proxy/>元素。在Java中使用@Scope注释配置bean提供了与proxyMode属性相同的支持。默认是ScopedProxyMode.DEFAULT，它通常表示不应创建作用域代理，除非在组件扫描指令级别配置了不同的默认值。您可以指定`ScopedProxyMode.TARGET_CLASS`, `ScopedProxyMode.INTERFACES` 或 `ScopedProxyMode.NO`.

如果您使用Java将作用域代理示例从XML参考文档(参见作用域代理)移植到我们的@Bean，它类似于以下内容:

```java
// an HTTP Session-scoped bean exposed as a proxy
@Bean
@SessionScope
public UserPreferences userPreferences() {
    return new UserPreferences();
}

@Bean
public Service userService() {
    UserService service = new SimpleUserService();
    // a reference to the proxied userPreferences bean
    service.setUserPreferences(userPreferences());
    return service;
}
```

##### bean的别名

正如在命名bean中所讨论的，有时需要为单个bean提供多个名称，否则称为bean别名。为此，@Bean注释的name属性接受一个String数组。下面的例子展示了如何为一个bean设置多个别名:

```java
@Configuration
public class AppConfig {

    @Bean({"dataSource", "subsystemA-dataSource", "subsystemB-dataSource"})
    public DataSource dataSource() {
        // instantiate, configure and return DataSource bean...
    }
}
```

##### bean的描述

有时，提供一个bean的更详细的文本描述是有帮助的。当为了监视目的而公开bean(可能通过JMX)时，这可能特别有用。

要向@Bean添加描述，可以使用@Description注释，如下面的示例所示:

```java
@Configuration
public class AppConfig {

    @Bean
    @Description("Provides a basic example of a bean")
    public Thing thing() {
        return new Thing();
    }
}
```

### 1.12.4. 使用`@Configuration`注解

@Configuration是一个类级注释，指示对象是bean定义的源。@Configuration类通过@ bean注释的方法声明bean。对@Configuration类上的@Bean方法的调用也可以用来定义bean间的依赖关系。有关基础概念的介绍，请参阅:@Bean和@Configuration。

#### 注入 inter-bean 依赖

当bean之间存在依赖关系时，表达这种依赖关系就像让一个bean方法调用另一个bean方法一样简单，如下面的示例所示:

```java
@Configuration
public class AppConfig {

    @Bean
    public BeanOne beanOne() {
        return new BeanOne(beanTwo());
    }

    @Bean
    public BeanTwo beanTwo() {
        return new BeanTwo();
    }
}
```

在前面的示例中，beanOne通过构造函数注入接收到对beanTwo的引用。

> 补充信息
>
> 这种声明bean之间依赖关系的方法只有在@Configuration类中声明@Bean方法时才有效。您不能通过使用普通的@Component类来声明bean间的依赖关系。

#### Lookup方法注入

如前所述，查找方法注入是一种高级特性，应该很少使用。当单个作用域的bean依赖于原型作用域的bean时，它非常有用。使用Java进行这种类型的配置为实现这种模式提供了一种自然的方法。下面的例子展示了如何使用查找方法注入:

```java
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

通过使用Java配置，您可以创建CommandManager的一个子类，其中抽象的createCommand()方法以查找新的(原型)命令对象的方式被重写。下面的例子展示了如何做到这一点:

```java
@Bean
@Scope("prototype")
public AsyncCommand asyncCommand() {
    AsyncCommand command = new AsyncCommand();
    // inject dependencies here as required
    return command;
}

@Bean
public CommandManager commandManager() {
    // return new anonymous implementation of CommandManager with createCommand()
    // overridden to return a new prototype Command object
    return new CommandManager() {
        protected Command createCommand() {
            return asyncCommand();
        }
    }
}
```

#### 关于基于java的配置如何在内部工作的进一步信息

思考下面的例子，它显示了一个@Bean注释的方法被调用了两次:

```java
// 两次调用同一个方法，产生的对象是同一个。cglib进行了中间代理。
//如果没有cglib的代理，将会产生两个不同的对象。
@Configuration
public class AppConfig {

    @Bean
    public ClientService clientService1() {
        ClientServiceImpl clientService = new ClientServiceImpl();
        clientService.setClientDao(clientDao());
        return clientService;
    }

    @Bean
    public ClientService clientService2() {
        ClientServiceImpl clientService = new ClientServiceImpl();
        clientService.setClientDao(clientDao());
        return clientService;
    }

    @Bean
    public ClientDao clientDao() {
        return new ClientDaoImpl();
    }
}
```

clientDao()在clientService1()和clientService2()中分别被调用了一次。由于该方法创建ClientDaoImpl的一个新实例并返回它，所以您通常期望有两个实例(每个服务一个)。这肯定会有问题:在Spring中，实例化的bean默认有一个单例作用域。这就是神奇的地方:所有的@Configuration类都在启动时用CGLIB进行子类化。在子类中，在调用父方法并创建新实例之前，子方法首先检查容器中是否有任何缓存的(限定作用域的)bean。

> 补充信息
>
> 根据bean的范围，行为可能不同。我们在这里谈论的是单例。

> 在Spring 3.2中，不再需要将CGLIB添加到你的类路径中，因为CGLIB类已经被重新打包到org.springframework.cglib下，并直接包含在Spring -core JAR中。

> 说明
>
> 由于CGLIB在启动时动态添加特性，因此有一些限制。特别是，配置类不能是final。然而，从4.3开始，配置类上允许使用任何构造函数，包括使用@Autowired或用于默认注入的单个非默认构造函数声明。
>
> 如果您希望避免任何cglib强加的限制，可以考虑在non-@Configuration类上声明@Bean方法(例如，在普通的@Component类上声明)。@Bean方法之间的跨方法调用不会被拦截，因此您必须在构造函数或方法级别上专门依赖依赖注入。

### 1.12.5. 编写基于java的配置

Spring基于java的配置特性允许您编写注释，这可以降低配置的复杂性。

#### 使用@`Import` 注解

就像在Spring XML文件中使用import元素来帮助模块化配置一样，@Import注释允许从另一个配置类加载@Bean定义，如下面的示例所示:

```java
@Configuration
public class ConfigA {

    @Bean
    public A a() {
        return new A();
    }
}

@Configuration
@Import(ConfigA.class)
public class ConfigB {

    @Bean
    public B b() {
        return new B();
    }
}
```

现在，在实例化上下文时不需要同时指定ConfigA.class和ConfigB.class，只需要显式提供ConfigB，如下例所示:

```java
public static void main(String[] args) {
    ApplicationContext ctx = new AnnotationConfigApplicationContext(ConfigB.class);

    // now both beans A and B will be available...
    A a = ctx.getBean(A.class);
    B b = ctx.getBean(B.class);
}
```

这种方法简化了容器实例化，因为只需要处理一个类，而不需要在构造过程中记住可能大量的@Configuration类。

> 说明
>
> 在Spring Framework 4.2中，@Import也支持对常规组件类的引用，类似于AnnotationConfigApplicationContext.register方法。如果您想要避免组件扫描，通过使用一些配置类作为入口点显式定义所有组件，这尤其有用。

##### 在导入的@Bean定义上注入依赖项

前面的示例可以工作，但过于简单。在大多数实际场景中，bean之间存在跨配置类的依赖关系。当使用XML时，这不是问题，因为不涉及编译器，并且您可以声明ref="someBean"，并相信Spring会在容器初始化期间解决它。当使用@Configuration类时，Java编译器会对配置模型施加约束，因为对其他bean的引用必须是有效的Java语法。

幸运的是，解决这个问题很简单。正如我们已经讨论过的，@Bean方法可以有任意数量的参数来描述bean依赖关系。考虑以下包含几个@Configuration类的更真实的场景，每个类都取决于在其他类中声明的bean:

```java
@Configuration
public class ServiceConfig {

    @Bean
    public TransferService transferService(AccountRepository accountRepository) {
        return new TransferServiceImpl(accountRepository);
    }
}

@Configuration
public class RepositoryConfig {

    @Bean
    public AccountRepository accountRepository(DataSource dataSource) {
        return new JdbcAccountRepository(dataSource);
    }
}

@Configuration
@Import({ServiceConfig.class, RepositoryConfig.class})
public class SystemTestConfig {

    @Bean
    public DataSource dataSource() {
        // return new DataSource
    }
}

public static void main(String[] args) {
    ApplicationContext ctx = new AnnotationConfigApplicationContext(SystemTestConfig.class);
    // everything wires up across configuration classes...
    TransferService transferService = ctx.getBean(TransferService.class);
    transferService.transfer(100.00, "A123", "C456");
}
```

还有一种方法可以达到同样的效果。请记住，@Configuration类最终只是容器中的另一个bean:这意味着它们可以像任何其他bean一样利用@Autowired和@Value 注入以及其他特性。

> 注意
>
> 确保您以这种方式注入的依赖项只是最简单的类型。@Configuration类在上下文初始化过程中很早就被处理了，以这种方式强制注入依赖可能会导致意外的早期初始化。尽可能使用基于参数的注入，如前面的示例所示。
>
> 另外，要特别注意通过@Bean定义BeanPostProcessor和BeanFactoryPostProcessor。这些方法通常应该声明为静态@Bean方法，而不是触发包含它们的配置类的实例化。否则，@Autowired和@Value可能无法在配置类本身上工作，因为可以在AutowiredAnnotationBeanPostProcessor之前将其创建为bean实例。

下面的例子展示了如何将一个bean自动连接到另一个bean:

```java
@Configuration
public class ServiceConfig {

    @Autowired
    private AccountRepository accountRepository;

    @Bean
    public TransferService transferService() {
        return new TransferServiceImpl(accountRepository);
    }
}

@Configuration
public class RepositoryConfig {

    private final DataSource dataSource;

    public RepositoryConfig(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    @Bean
    public AccountRepository accountRepository() {
        return new JdbcAccountRepository(dataSource);
    }
}

@Configuration
@Import({ServiceConfig.class, RepositoryConfig.class})
public class SystemTestConfig {

    @Bean
    public DataSource dataSource() {
        // return new DataSource
    }
}

public static void main(String[] args) {
    ApplicationContext ctx = new AnnotationConfigApplicationContext(SystemTestConfig.class);
    // everything wires up across configuration classes...
    TransferService transferService = ctx.getBean(TransferService.class);
    transferService.transfer(100.00, "A123", "C456");
}
```

> 说明
>
> @Configuration类中的构造函数注入只在Spring Framework 4.3中被支持。还要注意，如果目标bean只定义了一个构造函数，则不需要指定@Autowired。

##### 完全合格的导入bean，便于导航

在前面的场景中，使用@Autowired工作良好，并提供了所需的模块化，但是确定自动连接bean定义的确切声明位置仍然有些模糊。例如，作为查看ServiceConfig的开发人员，您如何确切地知道@Autowired AccountRepository bean是在哪里声明的?它在代码中不是显式的，这可能就可以了。记住，用于Eclipse的Spring Tools提供的工具可以呈现显示一切如何连接的图表，这可能就是您所需要的全部。而且，您的Java IDE可以轻松地找到AccountRepository类型的所有声明和使用，并快速地向您显示返回该类型的@Bean方法的位置。

如果这种歧义是不可接受的，并且您希望在IDE中从一个@Configuration类直接导航到另一个@Configuration类，那么可以考虑自动装配配置类本身。下面的例子展示了如何做到这一点:

```java
@Configuration
public class ServiceConfig {

    @Autowired
    private RepositoryConfig repositoryConfig;

    @Bean
    public TransferService transferService() {
        // navigate 'through' the config class to the @Bean method!
        return new TransferServiceImpl(repositoryConfig.accountRepository());
    }
}
```

在前面的情况中，定义AccountRepository的位置是完全显式的。但是，ServiceConfig现在与RepositoryConfig紧密耦合。这就是权衡。通过使用基于接口或基于抽象类的@Configuration类，可以在一定程度上缓解这种紧密耦合。考虑下面的例子:

```java
@Configuration
public class ServiceConfig {

    @Autowired
    private RepositoryConfig repositoryConfig;

    @Bean
    public TransferService transferService() {
        return new TransferServiceImpl(repositoryConfig.accountRepository());
    }
}

@Configuration
public interface RepositoryConfig {

    @Bean
    AccountRepository accountRepository();
}

@Configuration
public class DefaultRepositoryConfig implements RepositoryConfig {

    @Bean
    public AccountRepository accountRepository() {
        return new JdbcAccountRepository(...);
    }
}

@Configuration
@Import({ServiceConfig.class, DefaultRepositoryConfig.class})  // import the concrete config!
public class SystemTestConfig {

    @Bean
    public DataSource dataSource() {
        // return DataSource
    }

}

public static void main(String[] args) {
    ApplicationContext ctx = new AnnotationConfigApplicationContext(SystemTestConfig.class);
    TransferService transferService = ctx.getBean(TransferService.class);
    transferService.transfer(100.00, "A123", "C456");
}
```

现在ServiceConfig与具体的DefaultRepositoryConfig松散耦合，内置的IDE工具仍然很有用:您可以很容易地获得RepositoryConfig实现的类型层次结构。通过这种方式，导航@Configuration类及其依赖项与导航基于接口的代码的通常过程没有什么不同。

> 说明
>
> 如果您想要影响某些bean的启动创建顺序，可以考虑将其中一些bean声明为@Lazy(用于在第一次访问时而不是在启动时创建)或@DependsOn(确保在当前bean之前创建特定的其他bean，而不考虑后者的直接依赖关系)。

#### 有条件地包含@Configuration类或@Bean方法

根据某些任意的系统状态，有条件地启用或禁用一个完整的@Configuration类甚至个别的@Bean方法通常是有用的。一个常见的例子是，只有在Spring环境中启用了特定的概要文件时，才使用@Profile注释来激活Bean(详细信息请参阅Bean定义概要文件)。

@Profile注释实际上是通过使用一个更灵活的注释@Conditional实现的。@Conditional注释指示了在注册@Bean之前应该咨询的特定的org.springframework.context.annotation.Condition实现。

Condition接口的实现提供了一个matches(…)方法，该方法返回true或false。例如，下面的清单显示了@Profile使用的实际条件实现:

```java
@Override
public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
    // Read the @Profile annotation attributes
    MultiValueMap<String, Object> attrs = metadata.getAllAnnotationAttributes(Profile.class.getName());
    if (attrs != null) {
        for (Object value : attrs.get("value")) {
            if (context.getEnvironment().acceptsProfiles(((String[]) value))) {
                return true;
            }
        }
        return false;
    }
    return true;
}
```

#### 结合Java和XML配置

Spring的@Configuration类支持并不打算100%完全取代Spring XML。一些工具，如Spring XML名称空间，仍然是配置容器的理想方式。在XML方便或必要的情况下，您有一个选择:用“以XML为中心”的方式实例化容器，例如使用ClassPathXmlApplicationContext，或者用“以java为中心”的方式实例化它，使用AnnotationConfigApplicationContext和@ImportResource注释，根据需要导入XML。

##### 以xml为中心使用**@Configuration**注解的类

最好是从XML引导Spring容器，并以一种特殊的方式包含@Configuration类。例如，在使用Spring XML的大型现有代码库中，更容易根据需要创建@Configuration类，并从现有XML文件中包含它们。在本节的后面，我们将介绍在这种“以xml为中心”的情况下使用@Configuration类的情况。

*将@Configuration类声明为普通Spring bean元素*

记住，@Configuration类最终是容器中的bean定义。在本系列示例中，我们创建了一个名为AppConfig的@Configuration类，并将其作为bean定义包含在system-test-config.xml中。因为' context:annotation-config '被打开了，容器识别@Configuration注释并正确地处理AppConfig中声明的@Bean方法。

下面的例子展示了Java中的一个普通配置类:

```java
@Configuration
public class AppConfig {

    @Autowired
    private DataSource dataSource;

    @Bean
    public AccountRepository accountRepository() {
        return new JdbcAccountRepository(dataSource);
    }

    @Bean
    public TransferService transferService() {
        return new TransferService(accountRepository());
    }
}
```

下面的示例显示了system-test-config.xml示例文件的一部分:

```xml
<beans>
    <!-- enable processing of annotations such as @Autowired and @Configuration -->
    <context:annotation-config/>
    <context:property-placeholder location="classpath:/com/acme/jdbc.properties"/>

    <bean class="com.acme.AppConfig"/>

    <bean class="org.springframework.jdbc.datasource.DriverManagerDataSource">
        <property name="url" value="${jdbc.url}"/>
        <property name="username" value="${jdbc.username}"/>
        <property name="password" value="${jdbc.password}"/>
    </bean>
</beans>
```

下边展示了可能存在的 jdbc.properties配置文件：

```properties
jdbc.url=jdbc:hsqldb:hsql://localhost/xdb
jdbc.username=sa
jdbc.password=
```

```java
public static void main(String[] args) {
    ApplicationContext ctx = new ClassPathXmlApplicationContext("classpath:/com/acme/system-test-config.xml");
    TransferService transferService = ctx.getBean(TransferService.class);
    // ...
}
```

> 补充信息
>
> 在system-test-config.xml文件中，AppConfig bean不声明id元素。虽然这样做是可以接受的，但这是不必要的，因为没有其他bean引用它，而且不太可能根据名称显式地从容器中获取它。类似地，DataSource bean只按类型自动连接，因此并不严格要求显式的bean id。

*用**<context:component-scan/>**组件获取@Configuration的配置类*

因为@Configuration是用@Component进行元注释的，所以@Configuration注释类自动成为组件扫描的候选者。使用与前面示例中描述的相同的场景，我们可以重新定义system-test-config.xml，以利用组件扫描。注意，在这种情况下，我们不需要显式地声明' context:annotation-config '，因为' context:component-scan '启用了相同的功能。

修改后的system-test-config.xml文件示例如下:

```xml
<beans>
    <!-- picks up and registers AppConfig as a bean definition -->
    <context:component-scan base-package="com.acme"/>
    <context:property-placeholder location="classpath:/com/acme/jdbc.properties"/>

    <bean class="org.springframework.jdbc.datasource.DriverManagerDataSource">
        <property name="url" value="${jdbc.url}"/>
        <property name="username" value="${jdbc.username}"/>
        <property name="password" value="${jdbc.password}"/>
    </bean>
</beans>
```

##### 以**@Configuration**类为核心使用**@ImportResource**来引用xml配置

在@Configuration类是配置容器的主要机制的应用程序中，仍然可能至少需要使用一些XML。在这些场景中，您可以使用@ImportResource并只定义所需的XML。这样做可以实现一种“以java为中心”的方法来配置容器，并将XML保持在最低限度。下面的示例(包括一个配置类、一个定义bean的XML文件、一个属性文件和主类)展示了如何使用@ImportResource注释来实现“以java为中心”的配置，并根据需要使用XML:

```java
@Configuration
@ImportResource("classpath:/com/acme/properties-config.xml")
public class AppConfig {

    @Value("${jdbc.url}")
    private String url;

    @Value("${jdbc.username}")
    private String username;

    @Value("${jdbc.password}")
    private String password;

    @Bean
    public DataSource dataSource() {
        return new DriverManagerDataSource(url, username, password);
    }
}
```

```xml
properties-config.xml
<beans>
    <context:property-placeholder location="classpath:/com/acme/jdbc.properties"/>
</beans>
```

```properties
jdbc.properties
jdbc.url=jdbc:hsqldb:hsql://localhost/xdb
jdbc.username=sa
jdbc.password=
```

```java
public static void main(String[] args) {
    ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
    TransferService transferService = ctx.getBean(TransferService.class);
    // ...
}
```

## 1.13. 环境抽象

Environment接口是一个集成在容器中的抽象，它对应用程序环境的两个关键方面建模:概要文件和属性。

概要文件是一个命名的、逻辑的bean定义组，仅当给定的概要文件是活动的时才向容器注册。可以将bean分配给配置文件，无论是用XML定义的还是用注释定义的。与概要文件相关的Environment对象的作用是确定哪些概要文件(如果有的话)当前是活动的，以及哪些概要文件(如果有的话)在默认情况下应该是活动的。

属性在几乎所有应用程序中都扮演着重要的角色，并且可能来自各种来源:属性文件、JVM系统属性、系统环境变量、JNDI、servlet上下文参数、特殊的Properties对象、Map对象等等。与属性相关的Environment对象的作用是为用户提供一个方便的服务接口，用于配置属性源并从中解析属性。

### 1.13.1. bean定义的概要文件（Profiles）

Bean定义概要文件在核心容器中提供了一种机制，允许在不同环境中注册不同的Bean。“环境”这个词对不同的用户有不同的含义，这个功能可以帮助解决很多用例，包括:

- 在开发中使用内存中的数据源，而在QA或生产中从JNDI中查找相同的数据源。

- 只有在将应用程序部署到性能环境时才注册监视基础设施。

- 为客户A和客户B部署注册定制的bean实现。

考虑一个需要DataSource的实际应用程序中的第一个用例。在测试环境中，配置可能类似于以下:

```java
@Bean
public DataSource dataSource() {
    return new EmbeddedDatabaseBuilder()
        .setType(EmbeddedDatabaseType.HSQL)
        .addScript("my-schema.sql")
        .addScript("my-test-data.sql")
        .build();
}
```

现在考虑如何将该应用程序部署到QA或生产环境中，假设应用程序的数据源已注册到生产应用程序服务器的JNDI目录中。我们的dataSource bean现在看起来像下面的清单:

```java
@Bean(destroyMethod="")
public DataSource dataSource() throws Exception {
    Context ctx = new InitialContext();
    return (DataSource) ctx.lookup("java:comp/env/jdbc/datasource");
}
```

问题是如何根据当前环境在使用这两种变体之间切换。随着时间的推移，Spring用户设计了许多方法来实现这一点，通常依赖于系统环境变量和包含${placeholder}令牌的XML“import”语句的组合，该语句根据环境变量的值解析为正确的配置文件路径。Bean定义概要文件是为这个问题提供解决方案的核心容器特性。

如果我们推广前面特定于环境的bean定义示例中所示的用例，我们最终需要在某些上下文中注册某些bean定义，而在其他上下文中不注册。可以说，您希望在情况a中注册bean定义的某个概要文件，在情况b中注册不同的概要文件。我们首先更新配置以反映这种需求。

#### 使用`@Profile`

@Profile注释允许您在一个或多个指定的概要文件处于活动状态时指示组件有资格注册。使用前面的例子，我们可以像下面这样重写dataSource配置:

```java
@Configuration
@Profile("development")
public class StandaloneDataConfig {

    @Bean
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.HSQL)
            .addScript("classpath:com/bank/config/sql/schema.sql")
            .addScript("classpath:com/bank/config/sql/test-data.sql")
            .build();
    }
}
```

```java
@Configuration
@Profile("production")
public class JndiDataConfig {

    @Bean(destroyMethod="")
    public DataSource dataSource() throws Exception {
        Context ctx = new InitialContext();
        return (DataSource) ctx.lookup("java:comp/env/jdbc/datasource");
    }
}
```

> 补充信息
>
> 正如前面提到的，对于@Bean方法，您通常选择使用编程的JNDI查找，通过使用Spring的JndiTemplate/JndiLocatorDelegate帮助程序或前面显示的直接JNDI InitialContext用法，而不是使用JndiObjectFactoryBean变量，这将迫使您将返回类型声明为FactoryBean类型。

概要字符串可以包含一个简单的概要名称(例如，production)或一个概要表达式。概要表达式允许表达更复杂的概要逻辑(例如，production & us-east)。在配置文件表达式中支持以下操作符:

- :配置文件的逻辑“不”

- &:配置文件的逻辑“和”

- |:概要文件的逻辑“或”

> 补充信息
>
> 如果不使用括号，就不能混合使用&和|操作符。例如，production & us-east | eu-central不是一个有效的表达。它必须表示为production &(us-east | eu-central)。

为了创建自定义组合注释，可以使用@Profile作为元注释。下面的例子定义了一个自定义的@Production注释，你可以用它替换@Profile("production"):

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Profile("production")
public @interface Production {
}
```

> 说明
>
> 如果一个@Configuration类用@Profile标记，那么所有与这个类关联的@Bean方法和@Import注释都将被绕过，除非一个或多个指定的概要文件是活动的。如果一个@Component或@Configuration类被标记为@Profile({"p1"， "p2"})，这个类不会被注册或处理，除非配置文件'p1'或'p2'已经被激活。如果给定的概要文件以NOT操作符(!)为前缀，则只有在概要文件未激活时才会注册带注释的元素。例如，给定@Profile({"p1"， "!p2"})，如果概要文件'p1'是活动的，或者概要文件'p2'不是活动的，那么就会发生注册。

@Profile也可以在方法级别声明，以只包含配置类的一个特定bean(例如，特定bean的可选变体)，如下例所示:

```java
@Configuration
public class AppConfig {

    @Bean("dataSource")
    @Profile("development") 
    public DataSource standaloneDataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.HSQL)
            .addScript("classpath:com/bank/config/sql/schema.sql")
            .addScript("classpath:com/bank/config/sql/test-data.sql")
            .build();
    }

    @Bean("dataSource")
    @Profile("production") 
    public DataSource jndiDataSource() throws Exception {
        Context ctx = new InitialContext();
        return (DataSource) ctx.lookup("java:comp/env/jdbc/datasource");
    }
}
```

> 补充信息
>
> 在@Bean方法上使用@Profile，可能会应用一个特殊的场景:在具有相同Java方法名的重载@Bean方法的情况下(类似于构造函数重载)，需要在所有重载方法上一致地声明@Profile条件。如果条件不一致，则重载方法中只有第一个声明上的条件有意义。因此，@Profile不能用于选择具有特定参数签名的重载方法。同一bean的所有工厂方法之间的解析在创建时遵循Spring的构造函数解析算法。
>
> 如果您想定义具有不同概要文件条件的替代bean，可以使用不同的Java方法名，通过使用@Bean name属性指向相同的bean名，如前面的示例所示。如果参数签名都是相同的(例如，所有变量都有无参数的工厂方法)，那么这是首先在有效Java类中表示这种安排的唯一方法(因为一个特定名称和参数签名只能有一个方法)。

#### xml配置bean的概要文件

XML对应的是beans元素的概要文件属性。我们前面的示例配置可以在两个XML文件中重写，如下所示:

```xml
<beans profile="development"
    xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:jdbc="http://www.springframework.org/schema/jdbc"
    xsi:schemaLocation="...">

    <jdbc:embedded-database id="dataSource">
        <jdbc:script location="classpath:com/bank/config/sql/schema.sql"/>
        <jdbc:script location="classpath:com/bank/config/sql/test-data.sql"/>
    </jdbc:embedded-database>
</beans>
```

```xml
<beans profile="production"
    xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:jee="http://www.springframework.org/schema/jee"
    xsi:schemaLocation="...">

    <jee:jndi-lookup id="dataSource" jndi-name="java:comp/env/jdbc/datasource"/>
</beans>
```

也可以避免在同一个文件中拆分和嵌套bean元素，如下例所示:

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:jdbc="http://www.springframework.org/schema/jdbc"
    xmlns:jee="http://www.springframework.org/schema/jee"
    xsi:schemaLocation="...">

    <!-- other bean definitions -->

    <beans profile="development">
        <jdbc:embedded-database id="dataSource">
            <jdbc:script location="classpath:com/bank/config/sql/schema.sql"/>
            <jdbc:script location="classpath:com/bank/config/sql/test-data.sql"/>
        </jdbc:embedded-database>
    </beans>

    <beans profile="production">
        <jee:jndi-lookup id="dataSource" jndi-name="java:comp/env/jdbc/datasource"/>
    </beans>
</beans>
```

`spring bean.Xsd `已经被限制为只允许文件中的最后一个元素。这应该有助于提供灵活性，而不会在XML文件中造成混乱。

> 补充信息
>
> XML对应程序不支持前面描述的概要文件表达式。但是，可以通过使用!操作符。也可以通过嵌套配置文件来应用逻辑上的“and”，如下例所示:
>
> ```xml
> <beans xmlns="http://www.springframework.org/schema/beans"
>     xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
>     xmlns:jdbc="http://www.springframework.org/schema/jdbc"
>     xmlns:jee="http://www.springframework.org/schema/jee"
>     xsi:schemaLocation="...">
> 
>     <!-- other bean definitions -->
> 
>     <beans profile="production">
>         <beans profile="us-east">
>             <jee:jndi-lookup id="dataSource" jndi-name="java:comp/env/jdbc/datasource"/>
>         </beans>
>     </beans>
> </beans>
> ```
>
> 在前面的示例中，如果生产和us-east概要文件都是活动的，那么dataSource bean就会公开。

#### 激活一个概要文件

现在我们已经更新了我们的配置，我们仍然需要告诉Spring哪个概要文件是活动的。如果我们现在启动示例应用程序，我们将看到抛出NoSuchBeanDefinitionException，因为容器找不到名为dataSource的Spring bean。

激活概要文件可以通过几种方式完成，但最直接的方法是通过通过ApplicationContext可用的环境API以编程方式完成。下面的例子展示了如何做到这一点:

```java
AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
ctx.getEnvironment().setActiveProfiles("development");
ctx.register(SomeConfig.class, StandaloneDataConfig.class, JndiDataConfig.class);
ctx.refresh();
```

此外，您还可以通过spring.profiles.active属性声明式地激活概要文件，该属性可以通过系统环境变量、JVM系统属性、web.xml中的servlet上下文参数指定，甚至可以作为JNDI中的一个条目指定(参见PropertySource抽象)。在集成测试中，可以通过使用spring-test模块中的@ActiveProfiles注释来声明活动概要文件(请参阅环境概要文件的上下文配置)。

注意，概要文件不是一个非此即彼的命题。您可以同时激活多个配置文件。通过编程，您可以向setActiveProfiles()方法提供多个配置文件名称，该方法接受String…下面的示例激活多个配置文件:

```java
ctx.getEnvironment().setActiveProfiles("profile1", "profile2");
```

声明性地说，spring.profiles.active可以接受一个以逗号分隔的配置文件名称列表，如下例所示:

```properties
    -Dspring.profiles.active="profile1,profile2"
```

#### 默认概要配置文件

默认配置文件表示默认启用的配置文件。考虑下面的例子:

```java
@Configuration
@Profile("default")
public class DefaultDataConfig {

    @Bean
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.HSQL)
            .addScript("classpath:com/bank/config/sql/schema.sql")
            .build();
    }
}
```

如果没有活动的概要文件，则创建dataSource。您可以将此视为为一个或多个bean提供默认定义的方法。如果启用了任何配置文件，则不应用默认配置文件。

您可以通过在环境上使用setDefaultProfiles()来更改默认概要文件的名称，或者通过使用spring.profiles.default属性来声明。

### 1.13.2.`PropertySource` 抽象

Spring的Environment抽象在可配置的属性源层次结构上提供搜索操作。考虑以下清单:

```java
ApplicationContext ctx = new GenericApplicationContext();
Environment env = ctx.getEnvironment();
boolean containsMyProperty = env.containsProperty("my-property");
System.out.println("Does my environment contain the 'my-property' property? " + containsMyProperty);
```

在前面的代码片段中，我们看到了询问Spring my-property属性是否为当前环境定义的高级方法。要回答这个问题，Environment对象对一组PropertySource对象执行搜索。PropertySource是任何键-值对源的简单抽象，Spring的StandardEnvironment配置了两个PropertySource对象——一个表示JVM系统属性集(system . getproperties())，另一个表示系统环境变量集(system .getenv())。

> 补充信息
>
> 这些默认的属性源提供给StandardEnvironment，以便在独立的应用程序中使用。StandardServletEnvironment中填充了额外的默认属性源，包括servlet配置、servlet上下文参数和JndiPropertySource(如果JNDI可用的话)。

具体地说，当您使用StandardEnvironment时，如果my-property系统属性或my-property环境变量在运行时存在，则调用env.containsProperty("my-property")将返回true。

> 说明
>
> 执行的搜索是分层的。默认情况下，系统属性优先于环境变量。因此，如果my-property属性碰巧在调用env.getProperty("my-property")期间在这两个地方都设置了，则系统属性值“胜出”并返回。注意，属性值不会被合并，而是被前面的条目完全覆盖。
>
> 对于一个通用的StandardServletEnvironment，完整的层次结构如下所示，优先级最高的条目位于顶部:
>
> - ServletConfig参数(如果适用——例如，DispatcherServlet上下文)
>
> - ServletContext参数(web.xml上下文参数条目)
>
> - JNDI环境变量(java:comp/env/ entries)
>
> - JVM系统属性(-D命令行参数)
>
> - JVM系统环境(操作系统环境变量)

最重要的是，整个机制是可配置的。也许您有一个想要集成到此搜索中的自定义属性源。为此，实现并实例化您自己的PropertySource，并将其添加到当前环境的PropertySource集合中。下面的例子展示了如何做到这一点:

```java
ConfigurableApplicationContext ctx = new GenericApplicationContext();
MutablePropertySources sources = ctx.getEnvironment().getPropertySources();
sources.addFirst(new MyPropertySource());
```

在前面的代码中，在搜索中以最高优先级添加了MyPropertySource。如果它包含一个my-property属性，则检测并返回该属性，有利于任何其他propertsource中的任何my-property属性。MutablePropertySources API公开了许多允许对属性源集进行精确操作的方法。

### 1.13.3. 使用`@PropertySource`注解

@PropertySource注释为向Spring的环境中添加PropertySource提供了一种方便的声明性机制。

给定一个名为app.properties的文件，其中包含键值对testbean.name=myTestBean，下面的@Configuration类使用@PropertySource的方式是调用testBean.getName()返回myTestBean:

```java
@Configuration
@PropertySource("classpath:/com/myco/app.properties")
public class AppConfig {

    @Autowired
    Environment env;

    @Bean
    public TestBean testBean() {
        TestBean testBean = new TestBean();
        testBean.setName(env.getProperty("testbean.name"));
        return testBean;
    }
}
```

@PropertySource资源位置中的任何${…}占位符都将根据已经在环境中注册的属性源集进行解析，如下面的示例所示:

```java
@Configuration
@PropertySource("classpath:/com/${my.placeholder:default/path}/app.properties")
public class AppConfig {

    @Autowired
    Environment env;

    @Bean
    public TestBean testBean() {
        TestBean testBean = new TestBean();
        testBean.setName(env.getProperty("testbean.name"));
        return testBean;
    }
}
```

假设`my.placeholder`出现在已经注册的属性源中(例如，系统属性或环境变量)，占位符将解析为相应的值。如果不是，则使用default/path作为默认值。如果没有指定默认值且属性无法解析，则抛出IllegalArgumentException异常。

> 补充信息
>
> 根据Java 8约定，@PropertySource注释是可重复的。但是，所有这样的@PropertySource注释都需要在同一层声明，要么直接在配置类上声明，要么作为同一自定义注释中的元注释声明。不建议混合使用直接注释和元注释，因为直接注释有效地覆盖了元注释。

### 1.13.4. 语句中的占位符解析

过去，元素中的占位符的值只能根据JVM系统属性或环境变量进行解析。现在情况已经不同了。因为环境抽象集成在整个容器中，所以很容易通过它来路由占位符的解析。这意味着您可以以任何喜欢的方式配置解析过程。您可以更改搜索系统属性和环境变量的优先级，或者完全删除它们。您还可以根据需要将自己的属性源添加到混合中。

具体来说，下面的语句不管客户属性是在哪里定义的，只要它在Environment中可用就可以工作:

```xml
<beans>
    <import resource="com/bank/service/${customer}-config.xml"/>
</beans>
```

## 1.14. 注册一个LoadTimeWeaver

当类加载到Java虚拟机(JVM)中时，Spring使用LoadTimeWeaver动态地转换类。

要启动加载时编织，可以将@EnableLoadTimeWeaving添加到一个@Configuration类中，如下例所示:

```java
@Configuration
@EnableLoadTimeWeaving
public class AppConfig {
}
```

或者，对于XML配置，你可以使用context:load-time-weaver元素:

```xml
<beans>
    <context:load-time-weaver/>
</beans>
```

一旦为ApplicationContext配置好，该ApplicationContext中的任何bean都可以实现LoadTimeWeaverAware，从而接收到加载时编织器实例的引用。这在与Spring的JPA支持结合时特别有用，因为在Spring的JPA类转换中可能需要加载时编织。更多细节请参考LocalContainerEntityManagerFactoryBean javadoc。有关AspectJ加载时编织的更多信息，请参见Spring框架中使用AspectJ的加载时编织。

## 1.15. ApplicationContext的附加功能

正如在本章的介绍中所讨论的，org.springframework.beans.factory包提供了管理和操作bean的基本功能，包括以编程的方式。org.springframework.context包添加了ApplicationContext接口，它扩展了BeanFactory接口，此外还扩展了其他接口，以更面向应用程序框架的风格提供额外的功能。许多人以一种完全声明式的方式使用ApplicationContext，甚至不以编程方式创建它，而是依靠支持类，如ContextLoader自动实例化ApplicationContext，作为Java EE web应用程序正常启动过程的一部分。

为了以更面向框架的风格增强BeanFactory功能，context 包还提供了以下功能:

- 通过MessageSource接口访问i18n风格的消息。

- 通过ResourceLoader接口访问资源，例如url和文件。

- 事件发布，即通过使用ApplicationEventPublisher接口发布到实现ApplicationListener接口的bean。

- 加载多个(分层的)上下文，通过HierarchicalBeanFactory接口将每个上下文集中在一个特定的层上，例如应用程序的web层。

### 1.15.1. 使用MessageSource接口国际化

ApplicationContext接口扩展了一个名为MessageSource的接口，因此提供了国际化(“i18n”)功能。Spring还提供了HierarchicalMessageSource接口，该接口可以分层地解析消息。这些接口共同提供了Spring影响消息解析的基础。在这些接口上定义的方法包括:

- String getMessage(String code, Object[] args, String default, Locale loc):用于从MessageSource检索消息的基本方法。如果没有找到针对指定地区的消息，则使用默认消息。通过使用标准库提供的MessageFormat功能，传入的任何参数都将成为替换值。

- String getMessage(String code, Object[] args, Locale loc):本质上与前面的方法相同，但有一点不同:不能指定缺省消息。如果无法找到该消息，则抛出NoSuchMessageException。

- String getMessage(MessageSourceResolvable resolvable, Locale locale):前面方法中使用的所有属性也被包装在一个名为MessageSourceResolvable的类中，您可以在此方法中使用该类。

加载ApplicationContext时，它会自动搜索上下文中定义的MessageSource bean。bean必须具有messageSource的名称。如果找到了这样的bean，那么对上述方法的所有调用都将委托给消息源。如果没有找到消息源，ApplicationContext将尝试查找包含同名bean的父类。如果有，它就使用该bean作为MessageSource。如果ApplicationContext不能为消息找到任何源，则实例化一个空的DelegatingMessageSource，以便能够接受对上面定义的方法的调用。

Spring提供了三种MessageSource实现，ResourceBundleMessageSource, ReloadableResourceBundleMessageSource和StaticMessageSource。它们都实现了HierarchicalMessageSource，以便进行嵌套消息传递。StaticMessageSource很少使用，但它提供了向源添加消息的编程方法。下面的例子显示了ResourceBundleMessageSource:

```xml
<beans>
    <bean id="messageSource"
            class="org.springframework.context.support.ResourceBundleMessageSource">
        <property name="basenames">
            <list>
                <value>format</value>
                <value>exceptions</value>
                <value>windows</value>
            </list>
        </property>
    </bean>
</beans>
```

该示例假设您在类路径中定义了三个资源包，分别是format、exceptions和windows。任何解析消息的请求都采用通过ResourceBundle对象解析消息的jdk标准方式进行处理。为了本例的目的，假设上述两个资源包文件的内容如下:

```properties
    # in format.properties
    message=Alligators rock!
```

```properties
    # in exceptions.properties
    argument.required=The {0} argument is required.
```

下一个示例显示运行MessageSource功能的程序。请记住，所有ApplicationContext实现也是MessageSource实现，因此可以转换为MessageSource接口。

```java
public static void main(String[] args) {
    MessageSource resources = new ClassPathXmlApplicationContext("beans.xml");
    String message = resources.getMessage("message", null, "Default", Locale.ENGLISH);
    System.out.println(message);
}
```

以上程序的输出结果如下所示:

```
Alligators rock!
```

总之，MessageSource定义在一个名为beans.xml的文件中，该文件存在于类路径的根目录中。messageSource bean定义通过其basenames属性引用许多资源包。在列表中传递给basenames属性的三个文件作为类路径根文件存在，分别称为format.properties, exceptions.properties, and windows.properties。

下一个示例显示传递给消息查找的参数。这些参数被转换为String对象并插入到查找消息中的占位符中。

```xml
<beans>

    <!-- this MessageSource is being used in a web application -->
    <bean id="messageSource" class="org.springframework.context.support.ResourceBundleMessageSource">
        <property name="basename" value="exceptions"/>
    </bean>

    <!-- lets inject the above MessageSource into this POJO -->
    <bean id="example" class="com.something.Example">
        <property name="messages" ref="messageSource"/>
    </bean>

</beans>
```

```java
public class Example {

    private MessageSource messages;

    public void setMessages(MessageSource messages) {
        this.messages = messages;
    }

    public void execute() {
        String message = this.messages.getMessage("argument.required",
            new Object [] {"userDao"}, "Required", Locale.ENGLISH);
        System.out.println(message);
    }
}
```

调用execute()方法的结果输出如下所示:

```
The userDao argument is required.
```

关于国际化(“i18n”)，Spring的各种MessageSource实现遵循与标准JDK ResourceBundle相同的地区解析和回退规则。简而言之，继续前面定义的示例messageSource，如果希望根据British (en-GB)语言环境解析消息，则需要分别创建名为en_GB.properties、exceptions_en_GB.properties, and windows_en_GB.properties。

通常，区域设置解析由应用程序的周围环境管理。在下面的例子中，解析(英国)消息的语言环境是手动指定的:

```properties
# in exceptions_en_GB.properties
argument.required=Ebagum lad, the ''{0}'' argument is required, I say, required.
```

```java
public static void main(final String[] args) {
    MessageSource resources = new ClassPathXmlApplicationContext("beans.xml");
    String message = resources.getMessage("argument.required",
        new Object [] {"userDao"}, "Required", Locale.UK);
    System.out.println(message);
}
```

运行上述程序的输出结果如下所示:

```
Ebagum lad, the 'userDao' argument is required, I say, required.
```

您还可以使用MessageSourceAware接口来获取对已定义的任何MessageSource的引用。在创建和配置bean时，在实现MessageSourceAware接口的ApplicationContext中定义的任何bean都将被注入应用程序上下文的MessageSource。

> 补充信息
>
> 因为Spring的MessageSource基于Java的ResourceBundle，所以它不会合并具有相同基名的包，而是只使用找到的第一个包。具有相同基名的后续消息包将被忽略。

> 补充信息
>
> 作为ResourceBundleMessageSource的替代方案，Spring提供了一个ReloadableResourceBundleMessageSource类。该变体支持相同的包文件格式，但比基于标准JDK的ResourceBundleMessageSource实现更灵活。特别是，它允许从任何Spring资源位置读取文件(不仅仅是从类路径)，并支持包属性文件的热重新加载(同时在两者之间高效地缓存它们)。详见ReloadableResourceBundleMessageSource javadoc。

### 1.15.2. 标准与自定义事件

ApplicationContext中的事件处理是通过ApplicationEvent类和ApplicationListener接口提供的。如果将实现ApplicationListener接口的bean部署到上下文中，那么每次ApplicationEvent发布到ApplicationContext时，都会通知该bean。本质上，这是标准的Observer设计模式。

> 说明
>
> 在Spring 4.2中，事件基础设施得到了显著的改进，并提供了基于注释的模型以及发布任意事件的能力(即不一定从ApplicationEvent扩展的对象)。当这样的对象被发布时，我们为您将其包装在一个事件中。

下表描述了Spring提供的标准事件:

| 事件                       | 说明                                                         |
| -------------------------- | ------------------------------------------------------------ |
| ContextRefreshedEvent      | 当ApplicationContext初始化或刷新时发布(例如，通过使用ConfigurableApplicationContext接口上的refresh()方法)。在这里，“初始化”意味着加载所有bean，检测和激活后处理器bean，预实例化单例，并准备使用ApplicationContext对象。只要上下文没有关闭，刷新就可以被触发多次，只要所选的ApplicationContext实际上支持这种“热”刷新。例如，XmlWebApplicationContext支持热刷新，但GenericApplicationContext不支持。 |
| ContextStartedEvent        | 在使用ConfigurableApplicationContext接口上的start()方法启动ApplicationContext时发布。在这里，“started”意味着所有生命周期bean都接收一个显式的开始信号。通常，此信号用于显式停止后重新启动bean，但它也可以用于启动未配置为自动启动的组件(例如，在初始化时尚未启动的组件)。 |
| ContextStoppedEvent        | 在使用ConfigurableApplicationContext接口上的stop()方法停止ApplicationContext时发布。在这里，“停止”意味着所有生命周期bean都接收到一个显式的停止信号。一个已停止的上下文可以通过start()调用重新启动。 |
| ContextClosedEvent         | 通过在ConfigurableApplicationContext接口上使用close()方法或通过JVM关闭钩子关闭ApplicationContext时发布。在这里，“关闭”意味着将销毁所有单例bean。一旦上下文被关闭，它就会到达生命的尽头，无法刷新或重新启动。 |
| RequestHandledEvent        | 一个特定于web的事件，告诉所有bean一个HTTP请求已经得到服务。此事件在请求完成后发布。此事件仅适用于使用Spring的DispatcherServlet的web应用程序。 |
| ServletRequestHandledEvent | RequestHandledEvent的子类，用于添加特定于servlet的上下文信息。 |

您还可以创建和发布自己的自定义事件。下面的例子展示了一个扩展Spring的ApplicationEvent基类的简单类:

```java
public class BlockedListEvent extends ApplicationEvent {

    private final String address;
    private final String content;

    public BlockedListEvent(Object source, String address, String content) {
        super(source);
        this.address = address;
        this.content = content;
    }

    // accessor and other methods...
}
```

要发布一个自定义的ApplicationEvent，调用ApplicationEventPublisher上的publishEvent()方法。通常，这是通过创建实现ApplicationEventPublisherAware的类并将其注册为Spring bean来实现的。下面的例子展示了这样一个类:

```java
public class EmailService implements ApplicationEventPublisherAware {

    private List<String> blockedList;
    private ApplicationEventPublisher publisher;

    public void setBlockedList(List<String> blockedList) {
        this.blockedList = blockedList;
    }

    public void setApplicationEventPublisher(ApplicationEventPublisher publisher) {
        this.publisher = publisher;
    }

    public void sendEmail(String address, String content) {
        if (blockedList.contains(address)) {
            publisher.publishEvent(new BlockedListEvent(this, address, content));
            return;
        }
        // send email...
    }
}
```

在配置时，Spring容器检测到EmailService实现了ApplicationEventPublisherAware，并自动调用setApplicationEventPublisher()。实际上，传入的参数是Spring容器本身。您正在通过应用程序上下文的ApplicationEventPublisher接口与它交互。

要接收自定义的ApplicationEvent，您可以创建实现ApplicationListener的类，并将其注册为Spring bean。下面的例子展示了这样一个类:

```java
public class BlockedListNotifier implements ApplicationListener<BlockedListEvent> {

    private String notificationAddress;

    public void setNotificationAddress(String notificationAddress) {
        this.notificationAddress = notificationAddress;
    }

    public void onApplicationEvent(BlockedListEvent event) {
        // notify appropriate parties via notificationAddress...
    }
}
```

注意，ApplicationListener使用自定义事件的类型进行了一般参数化(在前面的例子中为BlockedListEvent)。这意味着onApplicationEvent()方法可以保持类型安全，避免任何向下转换的需要。您可以根据需要注册任意数量的事件监听器，但请注意，默认情况下，事件监听器同步接收事件。这意味着publishEvent()方法会阻塞，直到所有侦听器都完成对事件的处理。这种同步和单线程方法的一个优点是，当侦听器接收到事件时，如果事务上下文可用，它将在发布器的事务上下文中操作。如果需要另一种策略来发布事件，请参阅javadoc Spring的ApplicationEventMulticaster接口和SimpleApplicationEventMulticaster实现中的配置选项。

下面的示例显示了用于注册和配置上述每个类的bean定义:

```xml
<bean id="emailService" class="example.EmailService">
    <property name="blockedList">
        <list>
            <value>known.spammer@example.org</value>
            <value>known.hacker@example.org</value>
            <value>john.doe@example.org</value>
        </list>
    </property>
</bean>

<bean id="blockedListNotifier" class="example.BlockedListNotifier">
    <property name="notificationAddress" value="blockedlist@example.org"/>
</bean>
```

把它们放在一起，当调用emailService bean的sendEmail()方法时，如果有任何应该阻止的电子邮件消息，则会发布一个BlockedListEvent类型的自定义事件。blockedListNotifier bean被注册为ApplicationListener并接收BlockedListEvent，此时它可以通知适当的方。

> 补充信息
>
> Spring的事件机制设计用于在相同应用程序上下文中的Spring bean之间进行简单通信。然而，对于更复杂的企业集成需求，单独维护的Spring integration项目为构建轻量级的、面向模式的、事件驱动的体系结构提供了完整的支持，这些体系结构构建在众所周知的Spring编程模型之上。

#### 基于注解的事件监听

您可以使用@EventListener注释在托管bean的任何方法上注册一个事件监听器。BlockedListNotifier可以重写如下:

```java
public class BlockedListNotifier {

    private String notificationAddress;

    public void setNotificationAddress(String notificationAddress) {
        this.notificationAddress = notificationAddress;
    }

    @EventListener
    public void processBlockedListEvent(BlockedListEvent event) {
        // notify appropriate parties via notificationAddress...
    }
}
```

方法签名再次声明它侦听的事件类型，但这一次使用了一个灵活的名称，并且没有实现特定的侦听器接口。只要实际事件类型在实现层次结构中解析泛型参数，就可以通过泛型缩小事件类型。

如果您的方法应该监听多个事件，或者如果您想在定义它时不使用任何参数，那么也可以在注释本身上指定事件类型。下面的例子展示了如何做到这一点:

```java
@EventListener({ContextStartedEvent.class, ContextRefreshedEvent.class})
public void handleContextStart() {
    // ...
}
```

还可以通过使用定义SpEL表达式的注释的条件属性来添加附加的运行时筛选，它应该匹配以实际调用特定事件的方法。

下面的例子展示了如何重写我们的通知器，使其仅在事件的内容属性等于my-event时才被调用:

```java
@EventListener(condition = "#blEvent.content == 'my-event'")
public void processBlockedListEvent(BlockedListEvent blEvent) {
    // notify appropriate parties via notificationAddress...
}
```

每个SpEL表达式针对一个专用上下文进行计算。下表列出了对上下文可用的项，以便您可以使用它们进行条件事件处理:

| 名称            | 位置               | 描述                                                         | 示例                                                         |
| --------------- | ------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Event           | root object        | 实际的ApplicationEvent。                                     | #root.event` or `event                                       |
| Arguments array | root object        | 用于调用方法的参数(作为对象数组)。                           | `#root.args` or `args`; `args[0]` to access the first argument, etc. |
| *Argument name* | evaluation context | 任何方法参数的名称。如果由于某种原因，名称不可用(例如，因为在编译的字节码中没有调试信息)，也可以使用#a<#arg>语法，其中<#arg>表示参数索引(从0开始)。 | `#blEvent` or `#a0` (you can also use `#p0` or `#p<#arg>` parameter notation as an alias) |

请注意 #root.event ，使您能够访问底层事件，即使您的方法签名实际上引用了已发布的任意对象。

如果你需要发布一个事件作为处理另一个事件的结果，你可以改变方法签名来返回应该发布的事件，如下例所示:

```java
@EventListener
public ListUpdateEvent handleBlockedListEvent(BlockedListEvent event) {
    // notify appropriate parties via notificationAddress and
    // then publish a ListUpdateEvent...
}
```

> 异步侦听器不支持此特性。

handleBlockedListEvent()方法为它处理的每个BlockedListEvent发布一个新的ListUpdateEvent。如果需要发布多个事件，则可以返回一个Collection或事件数组。

#### 异步监听

如果您想要一个特定的侦听器异步处理事件，您可以重用常规的@Async支持。下面的例子展示了如何做到这一点:

```java
@EventListener
@Async
public void processBlockedListEvent(BlockedListEvent event) {
    // BlockedListEvent is processed in a separate thread
}
```

使用异步事件时要注意以下限制:

- 如果异步事件监听器抛出异常，则不会将其传播给调用者。更多细节请参见AsyncUncaughtExceptionHandler。

- 异步事件监听器方法不能通过返回值来发布后续事件。如果您需要作为处理的结果发布另一个事件，请注入一个ApplicationEventPublisher来手动发布事件。

#### 订购监听

如果你需要一个监听器在另一个监听器之前被调用，你可以在方法声明中添加@Order注释，如下面的例子所示:

```java
@EventListener
@Order(42)
public void processBlockedListEvent(BlockedListEvent event) {
    // notify appropriate parties via notificationAddress...
}
```

#### 泛型事件

还可以使用泛型进一步定义事件的结构。考虑使用EntityCreatedEvent<t>，其中T是被创建的实际实体的类型。</t>例如，您可以创建以下侦听器定义来仅接收Person的EntityCreatedEvent:

```java
@EventListener
public void onPersonCreated(EntityCreatedEvent<Person> event) {
    // ...
}
```

由于类型擦除，只有当触发的事件解析事件监听器筛选的泛型参数(也就是说，类PersonCreatedEvent扩展了EntityCreatedEvent <Person>{…})时，这种方法才有效。

在某些情况下，如果所有事件都遵循相同的结构，这可能会变得非常繁琐(对于上例中的事件应该是这样)。在这种情况下，您可以实现ResolvableTypeProvider来指导框架超越运行时环境所提供的内容。下面的事件显示了如何做到这一点:

```java
public class EntityCreatedEvent<T> extends ApplicationEvent implements ResolvableTypeProvider {

    public EntityCreatedEvent(T entity) {
        super(entity);
    }

    @Override
    public ResolvableType getResolvableType() {
        return ResolvableType.forClassWithGenerics(getClass(), ResolvableType.forInstance(getSource()));
    }
}
```

> 说明
>
> 这不仅适用于ApplicationEvent，还适用于作为事件发送的任何任意对象。

### 1.15.3. 方便地访问低级资源

为了优化应用程序上下文的使用和理解，您应该熟悉Spring的Resource抽象，如参考资料中所述。

应用程序上下文是一个ResourceLoader，可用于加载Resource对象。Resource本质上是JDK java.net.URL类的一个功能更丰富的版本。事实上，Resource的实现在适当的地方包装了java.net.URL的实例。Resource可以以透明的方式从几乎任何位置获得低级资源，包括从类路径、文件系统位置、任何可以用标准URL描述的位置，以及其他一些变体。如果资源位置字符串是没有任何特殊前缀的简单路径，那么这些资源的来源是特定的，并且适合于实际的应用程序上下文类型。

您可以配置部署到应用程序上下文中的bean，以实现特殊的回调接口ResourceLoaderAware，该接口在初始化时自动回调，同时将应用程序上下文本身作为ResourceLoader传入。您还可以公开Resource类型的属性，以用于访问静态资源。它们像其他属性一样被注入其中。您可以将这些Resource属性指定为简单的String路径，并在部署bean时依靠从这些文本字符串到实际Resource对象的自动转换。

提供给ApplicationContext构造函数的位置路径或路径实际上是资源字符串，以简单的形式根据特定的上下文实现进行适当处理。例如，ClassPathXmlApplicationContext将一个简单的位置路径视为一个类路径位置。您还可以使用带有特殊前缀的位置路径(资源字符串)来强制从类路径或URL加载定义，而不管实际的上下文类型是什么。

### 1.15.4. 应用程序启动跟踪

ApplicationContext管理Spring应用程序的生命周期，并围绕组件提供丰富的编程模型。因此，复杂的应用程序可能具有同样复杂的组件图和启动阶段。

用特定的指标跟踪应用程序启动步骤可以帮助了解启动阶段的时间花费在哪里，但它也可以作为一种更好地理解整个上下文生命周期的方法。

AbstractApplicationContext(及其子类)使用ApplicationStartup进行检测，后者收集关于各种启动阶段的StartupStep数据:

- 应用程序上下文生命周期(基本包扫描、配置类管理)

- bean生命周期(实例化、智能初始化、后处理)

- 应用程序事件处理

下面是AnnotationConfigApplicationContext中的插装示例:

```java
// create a startup step and start recording
StartupStep scanPackages = this.getApplicationStartup().start("spring.context.base-packages.scan");
// add tagging information to the current step
scanPackages.tag("packages", () -> Arrays.toString(basePackages));
// perform the actual phase we're instrumenting
this.scanner.scan(basePackages);
// end the current step
scanPackages.end();
```

应用程序上下文已经用多个步骤进行了测试。一旦记录下来，这些启动步骤就可以用特定的工具收集、显示和分析。关于现有启动步骤的完整列表，您可以查看专用的附录部分。

默认的ApplicationStartup实现是一个无操作的变体，以实现最小的开销。这意味着在默认情况下，在应用程序启动期间不会收集任何指标。Spring框架附带了一个用于跟踪Java FlightRecorder启动步骤的实现:FlightRecorderApplicationStartup。要使用此变体，您必须在创建它后立即将其实例配置到ApplicationContext。

如果开发人员正在提供自己的AbstractApplicationContext子类，或者希望收集更精确的数据，他们也可以使用ApplicationStartup基础设施。

> 注意
>
> ApplicationStartup只能在应用程序启动期间用于核心容器;这绝不是Java剖析器或度量库(如Micrometer)的替代品。

要开始收集自定义的StartupStep，组件可以直接从应用程序上下文中获取ApplicationStartup实例，让它们的组件实现ApplicationStartupAware，或者在任何注入点上请求ApplicationStartup类型。

> 开发人员不应该使用“spring.*”。创建自定义启动步骤时的命名空间。这个名称空间是为Spring内部使用保留的，可能会发生变化。

### 1.15.5. 方便的Web应用程序的ApplicationContext实例化

您可以通过使用(例如，ContextLoader)声明式地创建ApplicationContext实例。当然，您也可以通过使用ApplicationContext实现之一以编程方式创建ApplicationContext实例。

你可以使用ContextLoaderListener来注册一个ApplicationContext，如下面的例子所示:

```xml
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>/WEB-INF/daoContext.xml /WEB-INF/applicationContext.xml</param-value>
</context-param>

<listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
```

监听器检查contextConfigLocation参数。如果该参数不存在，侦听器将使用/WEB-INF/applicationContext.xml作为默认值。当参数存在时，监听器使用预定义的分隔符(逗号、分号和空格)将String分隔开，并使用值作为搜索应用程序上下文的位置。也支持蚁式路径模式。例如/WEB-INF/*Context.xml(用于所有名称以Context.xml结尾且位于WEB-INF目录中的文件)和/WEB-INF/**/*Context.xml(用于WEB-INF任意子目录中的所有此类文件)。

### 1.15.6.将Spring ApplicationContext部署为Java EE RAR文件

可以将Spring ApplicationContext部署为RAR文件，将上下文及其所有所需的bean类和库jar封装在Java EE RAR部署单元中。这相当于引导能够访问Java EE服务器设施的独立ApplicationContext(仅托管在Java EE环境中)。RAR部署是部署无头WAR文件的一种更自然的替代方案——实际上，一个没有任何HTTP入口点的WAR文件仅用于在Java EE环境中引导Spring ApplicationContext。

RAR部署非常适合不需要HTTP入口点而只由消息端点和计划的作业组成的应用程序上下文。在这样的上下文中，bean可以使用JTA事务管理器、JNDI绑定的JDBC DataSource实例和JMS ConnectionFactory实例等应用程序服务器资源，还可以通过Spring的标准事务管理、JNDI和JMX支持工具向平台的JMX服务器注册。应用组件还可以通过Spring的TaskExecutor抽象与应用服务器的JCA WorkManager交互。

有关RAR部署中涉及的配置细节，请参阅SpringContextResourceAdapter类的javadoc。

将Spring ApplicationContext作为Java EE RAR文件进行简单部署:

1. 将所有应用程序类打包到一个RAR文件中(RAR文件是一个具有不同文件扩展名的标准JAR文件)。

2. 将所有必需的库jar添加到RAR归档文件的根中。

3. 添加一个META-INF/ra.xml部署描述符(如用于SpringContextResourceAdapter的javadoc所示)和相应的Spring XML bean定义文件(通常是META-INF/applicationContext.xml)。

4. 将生成的RAR文件放到应用程序服务器的部署目录中。

> 补充信息
>
> 这种RAR部署单元通常是自包含的。它们不向外界公开组件，甚至不向同一应用程序的其他模块公开组件。与基于rar的ApplicationContext的交互通常通过它与其他模块共享的JMS目的地发生。例如，基于rar的ApplicationContext还可以调度一些作业或对文件系统中的新文件(或类似的东西)作出反应。如果它需要允许来自外部的同步访问，它可以(例如)导出RMI端点，这可以被同一机器上的其他应用程序模块使用。

## 1.16.`BeanFactory` API

BeanFactory API为Spring的IoC功能提供了底层基础。它的特定契约主要用于与Spring的其他部分和相关的第三方框架集成，它的DefaultListableBeanFactory实现是高级GenericApplicationContext容器中的一个关键委托。

BeanFactory和相关接口(如BeanFactoryAware、InitializingBean、DisposableBean)是其他框架组件的重要集成点。由于不需要任何注释甚至反射，它们允许在容器及其组件之间进行非常高效的交互。应用程序级bean可以使用相同的回调接口，但通常更喜欢声明性依赖注入，或者通过注释，或者通过编程配置。

请注意，核心BeanFactory API级别及其DefaultListableBeanFactory实现不假设要使用的配置格式或任何组件注释。所有这些风格都是通过扩展(如XmlBeanDefinitionReader和AutowiredAnnotationBeanPostProcessor)来实现的，并将共享BeanDefinition对象作为核心元数据表示来操作。这是使Spring的容器如此灵活和可扩展的本质。

### 1.16.1.`BeanFactory` 还是 `ApplicationContext`?

本节解释BeanFactory和ApplicationContext容器级别之间的区别以及引导的含义。

您应该使用ApplicationContext，除非您有很好的理由不这样做，使用GenericApplicationContext及其子类AnnotationConfigApplicationContext作为自定义引导的通用实现。这些是Spring核心容器的主要入口点，用于所有常见目的:加载配置文件、触发类路径扫描、以编程方式注册bean定义和注释类，以及(从5.0开始)注册功能性bean定义。

因为ApplicationContext包含BeanFactory的所有功能，所以除了需要对bean处理进行完全控制的场景外，一般建议使用ApplicationContext而不是普通BeanFactory。在ApplicationContext(例如GenericApplicationContext实现)中，有几种类型的bean是通过约定(即通过bean名称或bean类型—特别是后处理器)检测的，而普通的DefaultListableBeanFactory则与任何特殊的bean无关。

对于许多扩展容器特性，例如注释处理和AOP代理，BeanPostProcessor扩展点是必不可少的。如果只使用普通的DefaultListableBeanFactory，则默认情况下不会检测和激活这种后处理器。这种情况可能令人困惑，因为您的bean配置实际上没有任何错误。相反，在这种情况下，需要通过额外的设置来完全引导容器。

下表列出了BeanFactory和ApplicationContext接口和实现提供的特性。

| 特性                                | BeanFactory | ApplicationContext |
| ----------------------------------- | ----------- | ------------------ |
| Bean实例化/布线                     | Yes         | Yes                |
| 集成的生命周期管理                  | No          | Yes                |
| 自动BeanPostProcessor登记           | No          | Yes                |
| 自动BeanFactoryPostProcessor登记    | No          | Yes                |
| 方便的MessageSource访问(用于国际化) | No          | Yes                |
| 内置的ApplicationEvent发布机制      | No          | Yes                |

要显式地用DefaultListableBeanFactory注册一个bean后处理器，您需要以编程方式调用addBeanPostProcessor，如下例所示:

```java
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
// populate the factory with bean definitions

// now register any needed BeanPostProcessor instances
factory.addBeanPostProcessor(new AutowiredAnnotationBeanPostProcessor());
factory.addBeanPostProcessor(new MyBeanPostProcessor());

// now start using the factory
```

要将BeanFactoryPostProcessor应用到普通的DefaultListableBeanFactory，您需要调用它的postProcessBeanFactory方法，如下面的示例所示:

```java
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
reader.loadBeanDefinitions(new FileSystemResource("beans.xml"));

// bring in some property values from a Properties file
PropertySourcesPlaceholderConfigurer cfg = new PropertySourcesPlaceholderConfigurer();
cfg.setLocation(new FileSystemResource("jdbc.properties"));

// now actually do the replacement
cfg.postProcessBeanFactory(factory);
```

在这两种情况下，显式注册步骤都是不方便的，这就是为什么在spring支持的应用程序中，各种ApplicationContext变量优于普通的DefaultListableBeanFactory，特别是在典型企业设置中依赖BeanFactoryPostProcessor和BeanPostProcessor实例来实现扩展容器功能时。

> 补充信息
>
> AnnotationConfigApplicationContext注册了所有公共注释后处理器，并可能通过配置注释引入额外的处理器，例如@EnableTransactionManagement。在Spring基于注释的配置模型的抽象级别，bean后处理器的概念仅仅是内部容器的细节。





































































































































































































































































