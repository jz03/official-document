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

## 1.10. classpath扫描和组件管理

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

### 1.10.4. 使用过滤器自定义扫描

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



































































































































































































































































































































































