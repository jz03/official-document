## 1.1. DispatcherServlet

与许多其他web框架一样，Spring MVC是围绕前端控制器模式设计的，其中一个中心Servlet DispatcherServlet为请求处理提供了共享算法，而实际工作是由可配置的委托组件执行的。该模型非常灵活，支持多种工作流程。

DispatcherServlet和任何Servlet一样，需要使用Java配置或web.xml根据Servlet规范声明和映射。反过来，DispatcherServlet使用Spring配置来发现请求映射、视图解析、异常处理等所需的委托组件。

下面的Java配置示例注册并初始化DispatcherServlet，它由Servlet容器自动检测(参见Servlet配置):

```java
public class MyWebApplicationInitializer implements WebApplicationInitializer {

    @Override
    public void onStartup(ServletContext servletContext) {

        // Load Spring web application configuration
        AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();
        context.register(AppConfig.class);

        // Create and register the DispatcherServlet
        DispatcherServlet servlet = new DispatcherServlet(context);
        ServletRegistration.Dynamic registration = servletContext.addServlet("app", servlet);
        registration.setLoadOnStartup(1);
        registration.addMapping("/app/*");
    }
}
```

> 除了直接使用ServletContext API，您还可以扩展AbstractAnnotationConfigDispatcherServletInitializer并覆盖特定的方法(请参阅context层次结构下的示例)。

> 对于编程用例，可以使用GenericWebApplicationContext替代AnnotationConfigWebApplicationContext。详细信息请参见GenericWebApplicationContext javadoc。

下面的web.xml配置示例注册并初始化DispatcherServlet:

```xml
<web-app>

    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/app-context.xml</param-value>
    </context-param>

    <servlet>
        <servlet-name>app</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value></param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>

    <servlet-mapping>
        <servlet-name>app</servlet-name>
        <url-pattern>/app/*</url-pattern>
    </servlet-mapping>

</web-app>
```

> Spring Boot遵循不同的初始化顺序。Spring Boot没有与Servlet容器的生命周期挂钩，而是使用Spring配置来引导自身和嵌入的Servlet容器。Filter和Servlet声明在Spring配置中检测到，并注册到Servlet容器中。要了解更多细节，请参阅Spring Boot文档。

### 1.1.1. context层次结构

 DispatcherServlet需要一个WebApplicationContext(普通ApplicationContext的扩展)来进行它自己的配置。WebApplicationContext有一个到ServletContext和与之相关联的Servlet的链接。它还被绑定到ServletContext，以便应用程序在需要访问WebApplicationContext时可以使用requestcontexttutils上的静态方法来查找它。

对于许多应用程序来说，拥有一个单独的WebApplicationContext就足够了。也可以有一个context层次结构，其中一个根WebApplicationContext跨多个DispatcherServlet(或其他Servlet)实例共享，每个实例都有自己的子WebApplicationContext配置。有关context层次结构特性的更多信息，请参阅ApplicationContext的附加功能。

根WebApplicationContext通常包含基础设施bean，例如需要跨多个Servlet实例共享的数据存储库和业务服务。这些bean被有效地继承，并且可以在特定于Servlet的子WebApplicationContext中被重写(即重新声明)，该子WebApplicationContext通常包含给定Servlet的本地bean。下图显示了这种关系:

![mvc-context-hierarchy](mvc-context-hierarchy.png)

下面的例子配置了一个WebApplicationContext层次结构:

```java
public class MyWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    @Override
    protected Class<?>[] getRootConfigClasses() {
        return new Class<?>[] { RootConfig.class };
    }

    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class<?>[] { App1Config.class };
    }

    @Override
    protected String[] getServletMappings() {
        return new String[] { "/app1/*" };
    }
}
```

> 说明
>
> 如果不需要应用程序上下文层次结构，应用程序可以通过getRootConfigClasses()返回所有配置，并从getServletConfigClasses()返回null。

下面的例子展示了web.xml的等效内容:

```xml
<web-app>

    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/root-context.xml</param-value>
    </context-param>

    <servlet>
        <servlet-name>app1</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>/WEB-INF/app1-context.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>

    <servlet-mapping>
        <servlet-name>app1</servlet-name>
        <url-pattern>/app1/*</url-pattern>
    </servlet-mapping>

</web-app>
```

> 说明
>
> 如果不需要应用程序上下文层次结构，应用程序可以只配置“根”上下文，并将contextConfigLocation Servlet参数保留为空。

### 1.1.2. 特殊的Bean类型

DispatcherServlet委托给特殊的bean来处理请求并呈现适当的响应。所谓“特殊bean”，我们指的是实现框架契约的spring管理的Object实例。它们通常带有内置契约，但您可以自定义它们的属性并扩展或替换它们。

下表列出了DispatcherServlet检测到的特殊bean:

 

| Bean类型                                                     | 说明                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| HandlerMapping                                               | 将一个请求映射到一个处理程序，以及用于预处理和后处理的拦截器列表。映射基于一些标准，具体细节因HandlerMapping实现而异。<br/><br/>两个主要的HandlerMapping实现是RequestMappingHandlerMapping(它支持@RequestMapping注解方法)和SimpleUrlHandlerMapping(它维护URI路径模式到处理程序的显式注册)。 |
| HandlerAdapter                                               | 帮助DispatcherServlet调用映射到请求的处理程序，而不管处理程序实际是如何调用的。例如，调用带注解的控制器需要解析注解。HandlerAdapter的主要目的是保护DispatcherServlet不受这些细节的影响。 |
| [`HandlerExceptionResolver`](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-exceptionhandlers) | 策略来解决异常，可能将它们映射到处理程序、HTML错误视图或其他目标。 |
| [`ViewResolver`](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-viewresolver) | 将从处理程序返回的基于逻辑字符串的视图名称解析为要呈现给响应的实际视图。参见视图分辨率和视图技术。 |
| [`LocaleResolver`](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-localeresolver), [LocaleContextResolver](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-timezone) | 解析客户端正在使用的Locale，可能还有他们的时区，以便能够提供国际化的视图。看地区。 |
| [`ThemeResolver`](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-themeresolver) | 解析您的web应用程序可以使用的主题——例如，提供个性化布局。看到主题。 |
| [`MultipartResolver`](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-multipart) | 在一些多部分解析库的帮助下解析多部分请求(例如，浏览器表单文件上传)的抽象。 |
| [`FlashMapManager`](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-flash-attributes) | 存储和检索“输入”和“输出”FlashMap，可用于将属性从一个请求传递到另一个请求，通常是通过重定向。 |

### 1.1.3. web mvc 配置

应用程序可以声明特殊Bean类型中列出的处理请求所需的基础设施Bean。DispatcherServlet为每个特殊的bean检查WebApplicationContext。如果没有匹配的bean类型，它将退回到DispatcherServlet.properties中列出的默认类型。

在大多数情况下，MVC配置是最好的起点。它用Java或XML声明所需的bean，并提供高级配置回调API来自定义它。

> Spring Boot依赖于MVC Java配置来配置Spring MVC，并提供了许多额外的方便选项。

### 1.1.4. Servlet 配置

在Servlet 3.0+环境中，您可以选择以编程方式配置Servlet容器作为备选方案，或者与web.xml文件结合使用。下面的例子注册了一个DispatcherServlet:

```java
import org.springframework.web.WebApplicationInitializer;

public class MyWebApplicationInitializer implements WebApplicationInitializer {

    @Override
    public void onStartup(ServletContext container) {
        XmlWebApplicationContext appContext = new XmlWebApplicationContext();
        appContext.setConfigLocation("/WEB-INF/spring/dispatcher-config.xml");

        ServletRegistration.Dynamic registration = container.addServlet("dispatcher", new DispatcherServlet(appContext));
        registration.setLoadOnStartup(1);
        registration.addMapping("/");
    }
}
```

WebApplicationInitializer是Spring MVC提供的一个接口，它确保检测到您的实现并自动用于初始化任何Servlet 3容器。WebApplicationInitializer的抽象基类实现名为AbstractDispatcherServletInitializer，通过覆盖指定servlet映射和DispatcherServlet配置位置的方法，使注册DispatcherServlet更加容易。

这对于使用基于java的Spring配置的应用程序是推荐的，如下面的示例所示:

```java
public class MyWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    @Override
    protected Class<?>[] getRootConfigClasses() {
        return null;
    }

    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class<?>[] { MyWebConfig.class };
    }

    @Override
    protected String[] getServletMappings() {
        return new String[] { "/" };
    }
}
```

如果您使用基于xml的Spring配置，您应该直接从AbstractDispatcherServletInitializer扩展，如下面的示例所示:

```java
public class MyWebAppInitializer extends AbstractDispatcherServletInitializer {

    @Override
    protected WebApplicationContext createRootApplicationContext() {
        return null;
    }

    @Override
    protected WebApplicationContext createServletApplicationContext() {
        XmlWebApplicationContext cxt = new XmlWebApplicationContext();
        cxt.setConfigLocation("/WEB-INF/spring/dispatcher-config.xml");
        return cxt;
    }

    @Override
    protected String[] getServletMappings() {
        return new String[] { "/" };
    }
}
```

AbstractDispatcherServletInitializer还提供了一种方便的方法来添加Filter实例，并让它们自动映射到DispatcherServlet，如下面的示例所示:

```java
public class MyWebAppInitializer extends AbstractDispatcherServletInitializer {

    // ...

    @Override
    protected Filter[] getServletFilters() {
        return new Filter[] {
            new HiddenHttpMethodFilter(), new CharacterEncodingFilter() };
    }
}
```

每个筛选器都会根据其具体类型添加一个默认名称，并自动映射到DispatcherServlet。

AbstractDispatcherServletInitializer的isAsyncSupported受保护方法提供了一个单独的位置，用于在DispatcherServlet和映射到它的所有过滤器上启用异步支持。默认情况下，该标志被设置为true。

最后，如果您需要进一步定制DispatcherServlet本身，您可以重写createDispatcherServlet方法。

### 1.1.5 处理

DispatcherServlet处理请求如下:

- 在请求中搜索WebApplicationContext并将其绑定为控制器和流程中的其他元素可以使用的属性。它默认绑定在DispatcherServlet.WEB_APPLICATION_CONTEXT_ATTRIBUTE属性中。
- 区域设置解析器被绑定到请求，以让流程中的元素解析处理请求时使用的区域设置(呈现视图、准备数据等)。如果不需要区域设置解析，则不需要区域设置解析器。
- 主题解析器被绑定到请求，以让视图等元素决定使用哪个主题。如果不使用主题，可以忽略它。
- 如果指定了多部分文件解析器，则将对请求进行多部分检查。如果发现了多个部分，则将请求包装在MultipartHttpServletRequest中，以供流程中的其他元素进一步处理。有关多部分处理的更多信息，请参见多部分解析器。
- 搜索合适的处理程序。如果找到一个处理程序，则运行与该处理程序(预处理器、后处理器和控制器)相关联的执行链，以准备一个用于呈现的模型。另外，对于带注解的控制器，可以呈现响应(在HandlerAdapter中)，而不是返回视图。
- 如果返回模型，则呈现视图。如果没有返回模型(可能是由于预处理器或后处理器拦截了请求，也可能是出于安全原因)，则不会呈现视图，因为请求可能已经被满足了。

在WebApplicationContext中声明的HandlerExceptionResolver bean用于解决请求处理期间抛出的异常。这些异常解析器允许自定义处理异常的逻辑。有关更多详细信息，请参见异常。

对于HTTP缓存支持，处理程序可以使用WebRequest的checkNotModified方法，以及在控制器的HTTP缓存中描述的注解控制器的进一步选项。

通过在web.xml文件中的Servlet声明中添加Servlet初始化参数(init-param元素)，您可以自定义各个DispatcherServlet实例。支持的参数如下表所示:

| 参数                           | 说明                                                         |
| ------------------------------ | ------------------------------------------------------------ |
| contextClass                   | 实现ConfigurableWebApplicationContext的类，由这个Servlet实例化并在本地配置。默认情况下，使用XmlWebApplicationContext。 |
| contextConfigLocation          | 传递给上下文实例(由contextClass指定)的字符串，以指示在何处可以找到上下文。字符串可能由多个字符串(使用逗号作为分隔符)组成，以支持多个上下文。在定义两次bean的多个上下文位置的情况下，最新的位置优先。 |
| namespace                      | WebApplicationContext的命名空间。默认为servlet-name servlet。 |
| throwExceptionIfNoHandlerFound | 当没有为请求找到处理程序时，是否抛出NoHandlerFoundException。然后，可以使用HandlerExceptionResolver捕获异常(例如，通过使用@ExceptionHandler控制器方法)并像处理其他异常一样处理异常。<br/><br/>默认情况下，这被设置为false，在这种情况下，DispatcherServlet将响应状态设置为404 (NOT_FOUND)，而不会引发异常。<br/><br/>注意，如果还配置了默认的servlet处理，未解析的请求总是被转发到默认的servlet，而不会引发404。 |

### 1.1.6. 路径匹配

Servlet API将完整的请求路径公开为requestURI，并进一步将其细分为contextPath、servletPath和pathInfo，它们的值根据Servlet的映射方式而变化。从这些输入中，Spring MVC需要确定用于处理程序映射的查找路径，这是DispatcherServlet本身映射中的路径，不包括contextPath和任何servletMapping前缀(如果存在的话)。

servletPath和pathInfo是经过解码的，这使得它们不可能直接与完整的requestURI进行比较，以派生lookupath，这使得有必要解码requestURI。然而，这也带来了自己的问题，因为路径可能包含编码的保留字符，如"/"或";"，这些字符在被解码后又会改变路径的结构，这也会导致安全问题。此外，Servlet容器可能会对servletPath进行不同程度的规范化，这使得它进一步不可能对requestURI执行startsWith比较。

这就是为什么最好避免依赖于基于前缀的servletPath映射类型附带的servletPath。如果DispatcherServlet被映射为默认的Servlet，带有“/”或者不带“/*”前缀，并且Servlet容器是4.0+，那么Spring MVC能够检测Servlet映射类型，并避免使用servletPath和pathInfo。在3.1 Servlet容器中，假设相同的Servlet映射类型，可以通过在MVC配置中通过路径匹配提供一个带有alwaysUseFullPath=true的UrlPathHelper来实现等价的功能。

幸运的是，默认的Servlet映射“/”是一个很好的选择。然而，仍然存在一个问题，即需要对requestURI进行解码，以便能够与控制器映射进行比较。这也是不可取的，因为有可能解码改变路径结构的保留字符。如果不需要这样的字符，那么您可以拒绝它们(像Spring Security HTTP防火墙)，或者您可以用urlDecode=false配置UrlPathHelper，但控制器映射将需要与已编码的路径匹配，这可能并不总是工作良好。此外，有时DispatcherServlet需要与另一个Servlet共享URL空间，可能需要通过前缀进行映射。

通过从PathMatcher切换到5.3或更高版本中可用的经过解析的PathPattern，可以更全面地解决上述问题，请参阅模式比较。与需要解码查找路径或编码控制器映射的AntPathMatcher不同，解析后的PathPattern匹配名为RequestPath的路径解析表示，每次匹配一个路径段。这允许单独解码和清除路径段值，而不会有改变路径结构的风险。Parsed PathPattern还支持使用servletPath前缀映射，只要前缀保持简单，并且不包含任何需要编码的字符。

### 1.1.7. 拦截

所有HandlerMapping实现都支持处理程序拦截器，当您希望将特定功能应用于某些请求时(例如，检查主体)，这些拦截器非常有用。拦截器必须从org.springframework.web.servlet包中实现HandlerInterceptor，它有三个方法，应该提供足够的灵活性来进行各种预处理和后处理:

- `preHandle(..)`:在实际处理程序运行之前
- `postHandle(..)`: 在运行处理程序之后
- `fterCompletion(..)`:在完成完整的请求之后

preHandle(..)方法返回一个布尔值。您可以使用此方法中断或继续执行链的处理。当此方法返回true时，处理程序执行链将继续。当它返回false时，DispatcherServlet假定拦截器本身已经处理了请求(并且，例如，呈现了一个适当的视图)，不再继续执行执行链中的其他拦截器和实际的处理程序。

有关如何配置拦截器的示例，请参见MVC配置部分中的拦截器。还可以通过在单独的HandlerMapping实现上使用setter直接注册它们。

对于@ResponseBody和ResponseEntity方法，postHandle方法就不太有用了，因为它们的响应是在HandlerAdapter内部和postHandle之前编写和提交的。这意味着要对响应进行任何更改(例如添加额外的标头)都太晚了。对于这样的场景，您可以实现ResponseBodyAdvice并将其声明为Controller Advice bean，或者直接在RequestMappingHandlerAdapter上配置它。

### 1.1.8. 异常

如果在请求映射期间发生异常或从请求处理程序(例如@Controller)抛出异常，DispatcherServlet将委托给HandlerExceptionResolver bean链来解决异常并提供替代处理，这通常是一个错误响应。

下表列出了可用的HandlerExceptionResolver实现:

| HandlerExceptionResolver实现类                               | 说明                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| SimpleMappingExceptionResolver                               | 异常类名和错误视图名之间的映射。用于在浏览器应用程序中呈现错误页面。 |
| [`DefaultHandlerExceptionResolver`](https://docs.spring.io/spring-framework/docs/5.3.23/javadoc-api/org/springframework/web/servlet/mvc/support/DefaultHandlerExceptionResolver.html) | 解决Spring MVC引发的异常，并将它们映射到HTTP状态码。参见可选的ResponseEntityExceptionHandler和REST API异常。 |
| ResponseStatusExceptionResolver                              | 使用@ResponseStatus注解解决异常，并基于注解中的值将其映射到HTTP状态码。 |
| ExceptionHandlerExceptionResolver                            | 通过调用@Controller或@ControllerAdvice类中的@ExceptionHandler方法来解决异常。看到@ExceptionHandler方法。 |

#### 解析器链

您可以通过在Spring配置中声明多个HandlerExceptionResolver bean并根据需要设置它们的顺序属性来形成异常解析器链。顺序属性越高，异常解析器的位置越晚。

HandlerExceptionResolver的契约指定它可以返回:

- 指向错误视图的ModelAndView。
- 如果异常是在解析器中处理的，则为空ModelAndView。
- 如果异常仍然未解决，则为null，以便后续的解析器尝试;如果异常在结束时仍然存在，则允许它向上冒泡到Servlet容器。

MVC Config会自动为默认的Spring MVC异常、@ResponseStatus注解异常以及对@ExceptionHandler方法的支持声明内置解析器。您可以自定义该列表或替换它。

#### 容器错误页面

如果任何HandlerExceptionResolver仍然无法解析异常，因此让其传播，或者将响应状态设置为错误状态(即4xx, 5xx)， Servlet容器可以用HTML呈现默认的错误页面。要自定义容器的默认错误页面，可以在web.xml中声明一个错误页面映射。下面的例子展示了如何做到这一点:

```xml
<error-page>
    <location>/error</location>
</error-page>
```

对于前面的示例，当出现异常或响应具有错误状态时，Servlet容器在容器内向配置的URL(例如/error)发出error分派。然后DispatcherServlet对其进行处理，可能将其映射到@Controller，它可以被实现为返回带有模型的错误视图名或呈现JSON响应，如下面的示例所示:

```java
@RestController
public class ErrorController {

    @RequestMapping(path = "/error")
    public Map<String, Object> handle(HttpServletRequest request) {
        Map<String, Object> map = new HashMap<String, Object>();
        map.put("status", request.getAttribute("javax.servlet.error.status_code"));
        map.put("reason", request.getAttribute("javax.servlet.error.message"));
        return map;
    }
}
```

> 说明
>
> Servlet API不提供在Java中创建错误页面映射的方法。但是，您可以同时使用WebApplicationInitializer和最小的web.xml。

### 1.1.9. 视图解析

Spring MVC定义了ViewResolver和View接口，使您可以在浏览器中呈现模型，而无需将您绑定到特定的视图技术。ViewResolver提供了视图名称和实际视图之间的映射。视图处理在移交给特定视图技术之前的数据准备。

下表提供了ViewResolver层次结构的更多细节:

| 视图解析器                     | 描述                                                         |
| ------------------------------ | ------------------------------------------------------------ |
| AbstractCachingViewResolver    | AbstractCachingViewResolver的子类缓存它们解析的实例。缓存可以提高某些视图技术的性能。您可以通过将缓存属性设置为false来关闭缓存。此外，如果你必须在运行时刷新某个视图(例如，当FreeMarker模板被修改时)，你可以使用removeFromCache(String viewName, Locale loc)方法。 |
| UrlBasedViewResolver           | ViewResolver接口的简单实现，该接口直接将逻辑视图名解析为url，无需显式映射定义。如果您的逻辑名以直接的方式匹配视图资源的名称，而不需要任意的映射，那么这是合适的。 |
| InternalResourceViewResolver   | UrlBasedViewResolver的方便子类，它支持InternalResourceView(实际上，servlet和jsp)和JstlView和TilesView等子类。您可以使用setViewClass(..)为这个解析器生成的所有视图指定视图类。详见UrlBasedViewResolver javadoc。 |
| FreeMarkerViewResolver         | UrlBasedViewResolver的方便子类，支持FreeMarkerView和它们的自定义子类。 |
| ContentNegotiatingViewResolver | ViewResolver接口的实现，该接口基于请求文件名或Accept头解析视图。看到内容协商。 |
| BeanNameViewResolver           | ViewResolver接口的实现，该接口将视图名称解释为当前应用程序上下文中的bean名称。这是一个非常灵活的变体，它允许根据不同的视图名称混合和匹配不同的视图类型。每个这样的视图都可以定义为一个bean，例如在XML或配置类中。 |

#### 处理

您可以通过声明多个解析器bean来链接视图解析器，如果有必要，还可以通过设置order属性来指定顺序。记住，顺序属性越高，视图解析器在链中的位置越晚。

ViewResolver的契约指定它可以返回null来表示无法找到视图。然而，在JSP和InternalResourceViewResolver的情况下，确定JSP是否存在的唯一方法是通过RequestDispatcher执行分派。因此，您必须始终将InternalResourceViewResolver配置为在视图解析器的总体顺序中位于最后。

配置视图解析就像向Spring配置中添加ViewResolver bean一样简单。MVC配置为视图解析器和添加无逻辑视图控制器提供了专用的配置API，这对于没有控制器逻辑的HTML模板渲染非常有用。

#### 重定向（Redirecting）

视图名中的特殊前缀允许执行重定向。UrlBasedViewResolver(及其子类)将此识别为需要重定向的指令。视图名的其余部分是重定向URL。

最终效果与控制器返回一个RedirectView是一样的，但是现在控制器本身可以根据逻辑视图名进行操作。逻辑视图名(例如redirect:/myapp/some/resource)相对于当前Servlet上下文重定向，而像redirect:https://myhost.com/some/arbitrary/path这样的名称则重定向到一个绝对URL。

注意，如果控制器方法使用@ResponseStatus注解，那么注解值优先于RedirectView设置的响应状态。

#### 转发（Forwarding）

您还可以使用一个特殊的转发:前缀来表示最终由UrlBasedViewResolver及其子类解析的视图名称。这将创建一个InternalResourceView，它执行RequestDispatcher.forward()。因此，这个前缀对于InternalResourceViewResolver和InternalResourceView(对于JSP)没有用处，但是如果您使用另一种视图技术，但仍然希望强制转发由Servlet/JSP引擎处理的资源，那么它会很有帮助。注意，您也可以连接多个视图解析器。

#### 内容协商

ContentNegotiatingViewResolver不解析视图本身，而是委托给其他视图解析器，并选择与客户端请求的表示相似的视图。可以从Accept报头或查询参数(例如，"/path?format=pdf")确定表示形式。

ContentNegotiatingViewResolver通过比较请求媒体类型和与每个viewresolver相关联的视图支持的媒体类型(也称为Content-Type)来选择一个适当的视图来处理请求。列表中第一个具有兼容Content-Type的视图将表示形式返回给客户端。如果ViewResolver链不能提供兼容的视图，则会参考通过DefaultViews属性指定的视图列表。后一种选项适用于单例视图，它可以呈现当前资源的适当表示，而不管逻辑视图名是什么。Accept报头可以包含通配符(例如text/*)，在这种情况下，内容类型为text/xml的视图是兼容的匹配。

### 1.1.10. 本地化

### 1.1.11.主题

您可以应用Spring Web MVC框架主题来设置应用程序的整体外观，从而增强用户体验。主题是静态资源的集合，通常是样式表和图像，它们会影响应用程序的视觉样式。

### 1.1.12. Multipart 解析器

MultipartResolver来自于org.springframework.web.multipart包，它是一种解析包括文件上传在内的multipart 请求的策略。一种是基于Commons FileUpload的实现，另一种是基于Servlet 3.0的multipart 请求解析。

要启用多部分处理，您需要在DispatcherServlet Spring配置中声明MultipartResolver bean，名称为MultipartResolver。DispatcherServlet检测它并将其应用于传入的请求。当接收到内容类型为multipart/form-data的POST时，解析器解析内容，将当前HttpServletRequest包装为MultipartHttpServletRequest，以提供对解析文件的访问，并将部分作为请求参数公开。

#### Apache Commons `FileUpload`

要使用Apache Commons FileUpload，您可以配置一个名为multipartResolver的CommonsMultipartResolver类型的bean。您还需要将commons-fileupload jar作为类路径的依赖项。

这个解析器变体委托给应用程序中的本地库，提供了最大的Servlet容器间可移植性。作为一种替代方案，可以考虑通过容器自己的解析器进行标准Servlet多部分解析，如下所述。

> 扩展信息
>
> Commons FileUpload传统上只应用于POST请求，但接受任何多部分/内容类型。有关详细信息和配置选项，请参阅CommonsMultipartResolver javadoc。

#### Servlet 3.0

Servlet 3.0多部分解析需要通过Servlet容器配置来启用。这样做:

- 在Java中，在Servlet注册中设置MultipartConfigElement。
- 在web.xml配置文件中，在servlet中声明<multipart-config>

下面的例子展示了如何在Servlet注册中设置MultipartConfigElement:

```java
public class AppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    // ...

    @Override
    protected void customizeRegistration(ServletRegistration.Dynamic registration) {

        // Optionally also set maxFileSize, maxRequestSize, fileSizeThreshold
        registration.setMultipartConfig(new MultipartConfigElement("/tmp"));
    }

}
```

一旦Servlet 3.0配置就绪，您就可以添加一个名为multipartResolver的StandardServletMultipartResolver类型的bean。

> 此解析器变体原样使用Servlet容器的多部分解析器，潜在地将应用程序暴露于容器实现差异。默认情况下，它将尝试用任何HTTP方法解析任何多部分/内容类型，但这在所有Servlet容器中可能不受支持。有关详细信息和配置选项，请参阅StandardServletMultipartResolver javadoc。

### 1.1.13.日志

Spring MVC中的调试级日志被设计成紧凑、最小和人性化的。它关注的是那些反复使用的高价值信息位，而不是那些只在调试特定问题时有用的信息位。

跟踪级日志通常遵循与DEBUG相同的原则(例如，也不应该是消防水管)，但可以用于调试任何问题。此外，在TRACE和DEBUG中，一些日志消息可能显示不同级别的详细信息。

好的日志记录来自于使用日志的经验。如果您发现任何不符合规定的目标，请告诉我们。

#### 敏感数据

DEBUG和TRACE日志可能记录敏感信息。这就是为什么默认情况下请求参数和报头是被屏蔽的，并且必须通过DispatcherServlet上的enableLoggingRequestDetails属性显式启用它们的完整登录。

下面的例子展示了如何使用Java配置来实现这一点:

```java
public class MyInitializer
        extends AbstractAnnotationConfigDispatcherServletInitializer {

    @Override
    protected Class<?>[] getRootConfigClasses() {
        return ... ;
    }

    @Override
    protected Class<?>[] getServletConfigClasses() {
        return ... ;
    }

    @Override
    protected String[] getServletMappings() {
        return ... ;
    }

    @Override
    protected void customizeRegistration(ServletRegistration.Dynamic registration) {
        registration.setInitParameter("enableLoggingRequestDetails", "true");
    }

}
```

## 1.2.过滤器

spring-web模块提供了一些有用的过滤器:

### 1.2.1.表单数据

浏览器只能通过HTTP GET或HTTP POST提交表单数据，但非浏览器客户端也可以使用HTTP PUT、PATCH和DELETE。Servlet API需要ServletRequest.getParameter*()方法来支持仅针对HTTP POST的表单字段访问。

spring-web模块提供FormContentFilter来拦截内容类型为application/x-www-form-urlencoded的HTTP PUT、PATCH和DELETE请求，从请求体中读取表单数据，并包装ServletRequest以使表单数据通过ServletRequest. getparameter *()方法系列可用。

### 1.2.2. 转发头

当一个请求经过代理(例如负载平衡器)时，主机、端口和方案可能会发生变化，这使得从客户端角度创建指向正确的主机、端口和方案的链接成为一项挑战。



RFC 7239定义了转发的HTTP头，代理可以使用它来提供关于原始请求的信息。还有其他非标准头文件，包括X-Forwarded-Host, X-Forwarded-Port, X-Forwarded-Proto, X-Forwarded-Ssl, and X-Forwarded-Prefix。

ForwardedHeaderFilter是一个Servlet过滤器，它修改请求的目的是:a)根据转发头更改主机、端口和方案;b)删除这些头以消除进一步的影响。过滤器依赖于包装请求，因此排序上，必须在其他过滤器(如RequestContextFilter)之前，这些过滤器应该处理修改后的请求，而不是原始的请求。

对于转发的标头有一些安全考虑，因为应用程序无法知道标头是由代理添加的(如预期的那样)还是由恶意客户端添加的。这就是为什么在信任边界上的代理应该配置为删除来自外部的不受信任的转发标头。您还可以使用removeOnly=true配置ForwardedHeaderFilter，在这种情况下，它删除不使用头文件。

为了支持异步请求和错误分派，这个过滤器应该映射到DispatcherType.ASYNC和DispatcherType.ERROR。如果使用Spring框架的AbstractAnnotationConfigDispatcherServletInitializer(参见Servlet配置)，所有的过滤器都会自动为所有分派类型注册。然而，如果通过web.xml或通过FilterRegistrationBean在Spring Boot中注册过滤器，请确保包含DispatcherType.ASYNC和DispatcherType.ERROR。除了DispatcherType.REQUEST之外。

### 1.2.3.Shallow ETag

> ETag是URL的tag，用来标示URL对象是否改变。这样可以应用于客户端的缓存：服务器产生ETag，并在HTTP响应头中将其传送到客户端，服务器用它来判断页面是否被修改过，如果未修改返回304，无需传输整个对象。

ShallowEtagHeaderFilter过滤器通过缓存写入响应的内容并从中计算MD5哈希值来创建一个“shallow”ETag。下次客户端发送时，它也执行同样的操作，但它也将计算值与If-None-Match请求头进行比较，如果两者相等，则返回304 (NOT_MODIFIED)。

此策略节省了网络带宽，但不能节省CPU，因为必须为每个请求计算完整响应。前面描述的控制器级别的其他策略可以避免计算。参考HTTP缓存。

这个过滤器有一个writeWeakETag参数，它将过滤器配置为写类似于以下的writeWeakETag : W/"02a2d595e6ed9a0b24f027f2b63b134d6"(在RFC 7232章节2.3中定义)。

为了支持异步请求，该过滤器必须与DispatcherType.ASYNC映射，以便过滤器能够延迟并成功生成到最后一个异步调度结束的ETag。如果使用Spring框架的AbstractAnnotationConfigDispatcherServletInitializer(参见Servlet配置)，所有的过滤器都会自动为所有分派类型注册。然而，如果通过web.xml或通过FilterRegistrationBean在Spring Boot中注册过滤器，请确保包含DispatcherType.ASYNC。

### 1.2.4.cors

Spring MVC通过控制器上的注解为CORS配置提供了细粒度的支持。然而，当与Spring Security一起使用时，我们建议依赖内置的CorsFilter，它必须在Spring Security的过滤器链之前订购。

有关更多详细信息，请参阅CORS和CORS过滤器部分。

## 1.3.注解控制器

Spring MVC提供了一个基于注解的编程模型，其中@Controller和@RestController组件使用注解来表示请求映射、请求输入、异常处理等。带注解的控制器具有灵活的方法签名，不必扩展基类或实现特定的接口。下面的例子显示了一个由注解定义的控制器:

```java
@Controller
public class HelloController {

    @GetMapping("/hello")
    public String handle(Model model) {
        model.addAttribute("message", "Hello World!");
        return "index";
    }
}
```

在前面的例子中，该方法接受Model并返回一个视图名作为String，但是还有许多其他选项，将在本章后面解释。

> 说明
>
> 关于spring.io的指南和教程使用本节中描述的基于注释的编程模型。

### 1.3.1. 声明

您可以在Servlet的WebApplicationContext中使用标准Spring bean定义来定义控制器bean。@Controller构造型允许自动检测，与Spring对类路径中检测@Component类和为它们自动注册bean定义的一般保持一致。它还充当注释类的原型，表明它作为web组件的角色。

要启用这种@Controller bean的自动检测，您可以在Java配置中添加组件扫描，如下面的示例所示:

```java
@Configuration
@ComponentScan("org.example.web")
public class WebConfig {

    // ...
}
```

下面的示例显示了与上一个示例相同的XML配置:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:p="http://www.springframework.org/schema/p"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        https://www.springframework.org/schema/context/spring-context.xsd">

    <context:component-scan base-package="org.example.web"/>

    <!-- ... -->

</beans>
```

@RestController是一个组合注释，它本身用@Controller和@ResponseBody进行元注释，以指示一个控制器，它的每个方法都继承类型级别的@ResponseBody注释，因此直接写入响应体，而不是使用HTML模板进行视图解析和呈现。

#### AOP代理

在某些情况下，您可能需要在运行时用AOP代理装饰控制器。一个例子是，如果您选择在控制器上直接使用@Transactional注释。在这种情况下，特别是对于控制器，我们建议使用基于类的代理。这通常是控制器的默认选择。然而，如果控制器必须实现一个不是Spring Context回调的接口(例如InitializingBean、*Aware和其他)，则可能需要显式配置基于类的代理。例如，使用<tx:annotation-driven>，您可以更改为<tx:annotation-driven proxy-target-class="true">，使用@EnableTransactionManagement，您可以更改为@EnableTransactionManagement(proxyTargetClass = true)。

### 1.3.2. 请求映射

可以使用@RequestMapping注释将请求映射到控制器方法。它具有各种属性，可以根据URL、HTTP方法、请求参数、报头和媒体类型进行匹配。您可以在类级别使用它来表示共享映射，也可以在方法级别使用它来缩小到特定端点映射。

还有一些特定于HTTP方法的快捷方式变体@RequestMapping:

- `@GetMapping`
- `@PostMapping`
- `@PutMapping`
- `@DeleteMapping`
- `@PatchMapping`

提供快捷方式是Custom Annotations，因为大多数控制器方法应该映射到特定的HTTP方法，而不是使用@RequestMapping，默认情况下，它匹配所有HTTP方法。在类级别仍然需要@RequestMapping来表示共享映射。

下面的例子有类型和方法级别的映射:

```java
@RestController
@RequestMapping("/persons")
class PersonController {

    @GetMapping("/{id}")
    public Person getPerson(@PathVariable Long id) {
        // ...
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public void add(@RequestBody Person person) {
        // ...
    }
}
```

#### URI模式

@RequestMapping方法可以使用URL模式进行映射。有两种选择:

- PathPattern—一个与URL路径匹配的预解析模式，也被预解析为PathContainer。该解决方案专为web使用，有效地处理编码和路径参数，并有效地匹配。
- AntPathMatcher —根据字符串路径匹配字符串模式。这是Spring配置中用于选择类路径、文件系统和其他位置上的资源的原始解决方案。它的效率较低，字符串路径输入对于有效处理url的编码和其他问题是一个挑战。

PathPattern是web应用程序的推荐解决方案，也是Spring WebFlux的唯一选择。在5.3版本之前，AntPathMatcher是Spring MVC中唯一的选择，并且一直是默认的。然而，PathPattern可以在MVC配置中启用。

PathPattern支持与AntPathMatcher相同的模式语法。此外，它还支持捕获模式，例如{*spring}，用于在路径末尾匹配0个或多个路径段。PathPattern还限制使用**来匹配多个路径段，这样只允许在模式的末尾使用。这在为给定请求选择最佳匹配模式时消除了许多不明确的情况。完整的模式语法请参考PathPattern和AntPathMatcher。

一些示例模式:

- `"/resources/ima?e.png"` - 匹配路径段中的一个字符
- `"/resources/*.png"` - 在路径段中匹配零个或多个字符
- `"/resources/**"` - 匹配多个路径段
- `"/projects/{project}/versions"` - 匹配一个路径段并将其捕获为一个变量
- `"/projects/{project:[a-z]+}/versions"` - 用正则表达式匹配和捕获变量

可以使用@PathVariable访问捕获的URI变量。例如:

```java
@GetMapping("/owners/{ownerId}/pets/{petId}")
public Pet findPet(@PathVariable Long ownerId, @PathVariable Long petId) {
    // ...
}
```

你可以在类和方法级别声明URI变量，如下例所示:

```java
@Controller
@RequestMapping("/owners/{ownerId}")
public class OwnerController {

    @GetMapping("/pets/{petId}")
    public Pet findPet(@PathVariable Long ownerId, @PathVariable Long petId) {
        // ...
    }
}
```

URI变量自动转换为适当的类型，或者引发TypeMismatchException。默认情况下支持简单类型(int、long、Date等)，您可以注册对任何其他数据类型的支持。参见类型转换和数据绑定器。

您可以显式地命名URI变量(例如，@PathVariable("customId"))，但如果名称相同，并且您的代码是用调试信息或Java 8上的-parameters编译器标志编译的，则可以省略该细节。

语法{varName:regex}用正则表达式声明一个URI变量，其语法为{varName:regex}。例如，给定URL "/spring-web-3.0.5.jar"，下面的方法提取名称、版本和文件扩展名:

```java
@GetMapping("/{name:[a-z-]+}-{version:\\d\\.\\d\\.\\d}{ext:\\.[a-z]+}")
public void handle(@PathVariable String name, @PathVariable String version, @PathVariable String ext) {
    // ...
}
```

URI路径模式也可以有嵌入的${…}占位符，在启动时通过针对本地、系统、环境和其他属性源使用PropertySourcesPlaceholderConfigurer来解析这些占位符。例如，您可以使用它来根据一些外部配置参数化基本URL。

#### 模式对比

当多个模式匹配一个URL时，必须选择最佳匹配。这取决于是否启用parsed PathPattern的使用，可以通过以下方式之一完成:

- [`PathPattern.SPECIFICITY_COMPARATOR`](https://docs.spring.io/spring-framework/docs/5.3.23/javadoc-api/org/springframework/web/util/pattern/PathPattern.html#SPECIFICITY_COMPARATOR)
- [`AntPathMatcher.getPatternComparator(String path)`](https://docs.spring.io/spring-framework/docs/5.3.23/javadoc-api/org/springframework/util/AntPathMatcher.html#getPatternComparator-java.lang.String-)

两者都有助于将更具体的模式排在最上面。如果一个模式具有较低的URI变量计数(计算为1)、单个通配符计数(计算为1)和双通配符计数(计算为2)，那么该模式就不那么特定。如果得分相等，则选择较长的模式。给定相同的分数和长度，将选择URI变量多于通配符的模式。

默认映射模式(/\*\*)不计入评分，总是最后排序。此外，前缀模式(例如/public/\*\*)被认为不如没有双通配符的其他模式具体。

有关详细信息，请参见上面的模式比较器链接。

#### 后缀匹配

从5.3开始，默认情况下Spring MVC不再执行.\*后缀模式匹配，其中映射到/person的控制器也隐式映射到/person.*。因此，路径扩展不再用于解释响应所请求的内容类型—例如/person.pdf、/person.xml等等。

当浏览器用来发送难以一致解释的Accept头文件时，以这种方式使用文件扩展名是必要的。目前，这不再是必要的，使用Accept标头应该是首选。

随着时间的推移，使用文件扩展名在许多方面都被证明存在问题。当与URI变量、路径参数和URI编码的使用重叠时，可能会导致歧义。推理基于url的授权和安全性(更多细节见下一节)也变得更加困难。

要完全禁用5.3之前版本中路径扩展的使用，请设置以下内容:

- `useSuffixPatternMatching(false)`, 参见 [PathMatchConfigurer](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-config-path-matching)
- `favorPathExtension(false)`, 参见 [ContentNegotiationConfigurer](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-config-content-negotiation)

有一种方法来请求内容类型，而不是通过“Accept”头仍然是有用的，例如在浏览器中输入URL时。替代路径扩展的一种安全方法是使用查询参数策略。如果必须使用文件扩展名，可以考虑通过ContentNegotiationConfigurer的mediaTypes属性将它们限制为显式注册的扩展名列表。

#### 后缀匹配与RFD

反射文件下载(RFD)攻击类似于XSS，因为它依赖于响应中反映的请求输入(例如，查询参数和URI变量)。但是，RFD攻击不是将JavaScript插入到HTML中，而是依靠浏览器切换来执行下载，并在稍后双击时将响应作为可执行脚本处理。

在Spring MVC中，@ResponseBody和ResponseEntity方法是有风险的，因为它们可以呈现不同的内容类型，客户机可以通过URL路径扩展请求这些内容类型。禁用后缀模式匹配和使用路径扩展进行内容协商可以降低风险，但不足以防止RFD攻击。

为了防止RFD攻击，在呈现响应体之前，Spring MVC添加了Content-Disposition:inline;filename=f.txt头来建议固定和安全的下载文件。只有当URL路径包含一个既不安全也不显式注册用于内容协商的文件扩展名时，才会这样做。但是，当直接在浏览器中输入url时，可能会产生潜在的副作用。

默认情况下，许多公共路径扩展是安全的。具有自定义HttpMessageConverter实现的应用程序可以显式地为内容协商注册文件扩展名，以避免为那些扩展添加content - disposition头。看到内容类型。

参见CVE-2015-5211了解与RFD相关的其他建议。

#### 消费者媒体类型

您可以根据请求的Content-Type缩小请求映射，如下面的示例所示:

```java
@PostMapping(path = "/pets", consumes = "application/json") ①
public void addPet(@RequestBody Pet pet) {
    // ...
}
```

①使用consumes属性按内容类型缩小映射。

consumes属性还支持否定表达式——例如，!text/plain表示除了text/plain以外的任何内容类型。

您可以在类级别声明共享consumes属性。然而，与大多数其他请求映射属性不同的是，当在类级别使用时，方法级别使用属性覆盖而不是扩展类级别声明。

> 说明
>
> MediaType为常用的媒体类型提供常量，例如APPLICATION_JSON_VALUE和APPLICATION_XML_VALUE。

#### 生产者媒体类型

你可以根据Accept请求头和控制器方法生成的内容类型列表缩小请求映射，如下例所示:

```java
@GetMapping(path = "/pets/{petId}", produces = "application/json") ①
@ResponseBody
public Pet getPet(@PathVariable String petId) {
    // ...
}
```

①使用produces属性按内容类型缩小映射范围。

produces属性还支持否定表达式——例如，!text/plain表示除了text/plain以外的任何内容类型。

您可以在类级别声明共享produces属性。然而，与大多数其他请求映射属性不同的是，当在类级别使用时，方法级别使用属性覆盖而不是扩展类级别声明。

#### 变量与头部

可以根据请求参数条件缩小请求映射。您可以测试是否存在请求参数(myParam)，是否缺少请求参数(!myParam)，或者是否有特定的值(myParam=myValue)。下面的例子展示了如何测试一个特定的值:

```java
@GetMapping(path = "/pets/{petId}", params = "myParam=myValue") ①
public void findPet(@PathVariable String petId) {
    // ...
}
```

①测试myParam是否等于myValue。

你也可以使用同样的请求头条件，如下例所示:

```java
@GetMapping(path = "/pets", headers = "myHeader=myValue") ①
public void findPet(@PathVariable String petId) {
    // ...
}
```

①测试myParam是否等于myValue。

> 说明
>
> 您可以用header条件匹配Content-Type和Accept，但最好使用consumes和produces来代替。

#### HTTP HEAD, OPTIONS

@GetMapping(和@RequestMapping(method=HttpMethod.GET))对请求映射透明地支持HTTP HEAD。控制器方法不需要更改。一个响应包装器，应用于javax.servlet.http.HttpServlet，确保Content-Length头被设置为写入的字节数(不实际写入响应)。

@GetMapping(和@RequestMapping(method=HttpMethod.GET))隐式映射到并支持HTTP HEAD。HTTP HEAD请求被当作HTTP GET处理，不同的是，不是写入主体，而是计算字节数并设置Content-Length头。

默认情况下，HTTP OPTIONS是通过将允许响应头设置为所有具有匹配URL模式的@RequestMapping方法中列出的HTTP方法列表来处理的。

对于没有HTTP方法声明的@RequestMapping, Allow头被设置为GET,HEAD,POST,PUT,PATCH,DELETE,OPTIONS。控制器方法应该始终声明支持的HTTP方法(例如，通过使用HTTP方法特定的变体:@GetMapping、@PostMapping和其他)。

您可以显式地将@RequestMapping方法映射到HTTP HEAD和HTTP OPTIONS，但在一般情况下这是不必要的。

#### 自定义注解

Spring MVC支持使用组合注释进行请求映射。这些注释本身是用@RequestMapping进行元注释的，组合起来是为了重新声明@RequestMapping属性的一个子集(或全部)，具有更窄、更具体的用途。

@GetMapping、@PostMapping、@PutMapping、@DeleteMapping和@PatchMapping是组合注释的例子。提供它们是因为大多数控制器方法应该映射到特定的HTTP方法，而不是使用@RequestMapping，默认情况下，它匹配所有HTTP方法。如果需要组合注释的示例，请查看这些注释是如何声明的。

Spring MVC还支持使用自定义请求匹配逻辑的自定义请求映射属性。这是一个更高级的选项，它需要继承RequestMappingHandlerMapping并重写getCustomMethodCondition方法，在这个方法中您可以检查自定义属性并返回您自己的RequestCondition。

#### 显式注册

您可以以编程方式注册处理程序方法，您可以将其用于动态注册或高级情况，例如在不同的url下使用相同处理程序的不同实例。下面的例子注册了一个处理程序方法:

```java
@Configuration
public class MyConfig {

    @Autowired
    public void setHandlerMapping(RequestMappingHandlerMapping mapping, UserHandler handler) ①
            throws NoSuchMethodException {

        RequestMappingInfo info = RequestMappingInfo
                .paths("/user/{id}").methods(RequestMethod.GET).build(); ②

        Method method = UserHandler.class.getMethod("getUser", Long.class); ③

        mapping.registerMapping(info, handler, method); ④
    }
}
```

①为控制器注入目标处理程序和处理程序映射。

②准备请求映射元数据。

③获取handler方法。

④添加注册。

### 1.3.3.方法处理

@RequestMapping方法处理具有灵活的签名，可以从支持的控制器方法参数和返回值中选择。

#### 方法参数

下表描述了支持的控制器方法参数。任何参数都不支持响应式类型。

JDK 8的java.util.Optional被支持作为方法参数，与带有required属性的注释(例如，@RequestParam， @RequestHeader和其他)结合使用，相当于required=false。

| 控制器方法参数                                               | 说明                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| WebRequest`, `NativeWebRequest                               | 对请求参数、请求和会话属性的通用访问，而不直接使用Servlet API。 |
| javax.servlet.ServletRequest，javax.servlet.ServletResponse  | 选择任何特定的请求或响应类型——例如，ServletRequest, HttpServletRequest，或Spring的MultipartRequest, MultipartHttpServletRequest。 |
| javax.servlet.http.HttpSession                               | 强制会话的存在。因此，这样的参数永远不会为空。注意，会话访问不是线程安全的。如果允许多个请求并发访问一个会话，可以考虑将RequestMappingHandlerAdapter实例的synchronizeOnSession标志设置为true。 |
| javax.servlet.http.PushBuilder                               | 用于编程HTTP/2资源推送的Servlet 4.0推送构建器API。注意，根据Servlet规范，如果客户端不支持HTTP/2特性，则注入的PushBuilder实例可以为空。 |
| java.security.Principal                                      | 当前经过身份验证的用户——如果知道，可能是特定的Principal实现类。<br/><br/>注意，如果这个参数被注释，以便自定义解析器在通过HttpServletRequest#getUserPrincipal退回到默认解析之前解析它，那么这个参数不会被急切地解析。例如，Spring安全认证实现了Principal，并将通过HttpServletRequest#getUserPrincipal进行注入，除非它还注释了@AuthenticationPrincipal，在这种情况下，它将由自定义的Spring安全解析器通过Authentication#getPrincipal进行解析。 |
| HttpMethod                                                   | 请求的HTTP方法。                                             |
| java.util.Locale                                             | 当前请求区域设置，由可用的最具体的LocaleResolver确定(实际上，配置的LocaleResolver或LocaleContextResolver)。 |
| java.util.TimeZone` + `java.time.ZoneId                      | 与当前请求关联的时区，由LocaleContextResolver确定。          |
| java.io.InputStream`, `java.io.Reader                        | 用于访问由Servlet API公开的原始请求体。                      |
| java.io.OutputStream`, `java.io.Writer                       | 用于访问由Servlet API公开的原始请求体。                      |
| @PathVariable                                                | 用于访问URI模板变量。参见URI模式。                           |
| @MatrixVariable                                              | 用于访问URI路径段中的名称-值对。参见矩阵变量。               |
| @RequestParam                                                | 用于访问Servlet请求参数，包括多部分文件。参数值被转换为声明的方法参数类型。参见@RequestParam和Multipart。<br/>注意，对于简单的参数值，使用@RequestParam是可选的。参见表格末尾的“任何其他参数”。 |
| @RequestHeader                                               | 用于访问请求标头。头值被转换为声明的方法参数类型。参见@RequestHeader。 |
| @CookieValue                                                 | 访问cookie。Cookies值被转换为声明的方法参数类型。参见@CookieValue。 |
| @RequestBody                                                 | 用于访问HTTP请求体。主体内容通过使用HttpMessageConverter实现转换为声明的方法参数类型。参见@RequestBody。 |
| HttpEntity<B>                                                | 用于访问请求头和请求体。主体使用HttpMessageConverter进行转换。参见HttpEntity。 |
| @RequestPart                                                 | 为了访问多部分/表单数据请求中的部分，使用HttpMessageConverter转换部分的主体。参见Multipart。 |
| java.util.Map，org.springframework.ui.Model，org.springframework.ui.ModelMap | 用于访问HTML控制器中使用的模型，并作为视图呈现的一部分公开给模板。 |
| RedirectAttributes                                           | 指定在重定向的情况下使用的属性(即被附加到查询字符串中)，以及临时存储的flash属性，直到重定向后的请求。参见重定向属性和Flash属性。 |
| @ModelAttribute                                              | 用于访问模型中已存在的属性(如果不存在则实例化)，并应用数据绑定和验证。参见@ModelAttribute以及Model和DataBinder。<br/>注意，使用@ModelAttribute是可选的(例如，设置其属性)。参见本表末尾的“任何其他参数”。 |
| Errors`, `BindingResult                                      | 用于访问来自命令对象(即@ModelAttribute参数)的验证和数据绑定的错误，或来自@RequestBody或@RequestPart参数验证的错误。必须在经过验证的方法参数之后立即声明Errors或BindingResult参数。 |
| SessionStatus` + class-level `@SessionAttributes             | 用于标记表单处理完成，这将触发清除通过类级别@SessionAttributes注释声明的会话属性。更多细节请参见@SessionAttributes。 |
| UriComponentsBuilder                                         | 用于准备相对于当前请求的主机、端口、方案、上下文路径和servlet映射的文字部分的URL。更多细节请参见URI链接。 |
| @SessionAttribute                                            | 对于任何会话属性的访问，与作为类级别@SessionAttributes声明的结果存储在会话中的模型属性相反。更多细节请参见@SessionAttribute。 |
| @RequestAttribute                                            | 用于访问请求属性。详见@RequestAttribute。                    |
| Any other argument                                           | 如果一个方法参数与该表中前面的任何值都不匹配，并且它是一个简单类型(由BeanUtils#isSimpleProperty确定)，则它将被解析为@RequestParam。否则，它将被解析为@ModelAttribute。 |

#### 返回值

下一个表描述了支持的控制器方法返回值。所有返回值都支持响应式类型。

| 控制器方法返回值                                             | 描述                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| @ResponseBody                                                | 返回值通过HttpMessageConverter实现转换并写入响应。参见@ResponseBody。 |
| HttpEntity<B>，ResponseEntity<B>                             | 指定完整响应(包括HTTP报头和正文)的返回值将通过HttpMessageConverter实现进行转换，并写入响应。参见ResponseEntity。 |
| HttpHeaders                                                  | 用于返回带有头部而没有正文的响应。                           |
| String                                                       | 用ViewResolver实现解析并与隐式模型一起使用的视图名称——通过命令对象和@ModelAttribute方法确定。处理程序方法还可以通过声明model参数(参见显式注册)以编程方式丰富模型。 |
| View                                                         | 一个视图实例，用来与隐式模型一起渲染——通过命令对象和@ModelAttribute方法确定。处理程序方法还可以通过声明model参数(参见显式注册)以编程方式丰富模型。 |
| java.util.Map，org.springframework.ui.Model                  | 属性要添加到隐式模型中，视图名称通过RequestToViewNameTranslator隐式确定。 |
| @ModelAttribute                                              | 添加到模型中的属性，通过RequestToViewNameTranslator隐式确定视图名称。<br/>注意，@ModelAttribute是可选的。参见该表末尾的“任何其他返回值”。 |
| `ModelAndView` object                                        | 要使用的视图和模型属性，以及可选的响应状态。                 |
| void                                                         | 具有void返回类型(或空返回值)的方法如果还具有ServletResponse、OutputStream参数或@ResponseStatus注释，则被认为已经完全处理了响应。如果控制器已经做了一个正的ETag或lastModified时间戳检查，也会发生同样的情况(详见控制器)。<br/>如果以上都不为真，void返回类型也可以为REST控制器指示“无响应体”，或为HTML控制器指示默认视图名称选择。 |
| DeferredResult<V>                                            | 从任何线程异步地产生上述任何返回值—例如，作为某个事件或回调的结果。参见异步请求和DeferredResult。 |
| Callable<V>                                                  | 在Spring mvc管理的线程中异步生成上述任何返回值。请参见异步请求和可调用。 |
| ListenableFuture<V>，java.util.concurrent.CompletionStage<V>，java.util.concurrent.CompletableFuture<V> | 作为一种方便(例如，当底层服务返回其中之一时)，可以替代DeferredResult。 |
| ResponseBodyEmitter，SseEmitter                              | 发出异步的对象流，用HttpMessageConverter实现将其写入响应。也支持作为ResponseEntity的主体。请参见异步请求和HTTP流。 |
| StreamingResponseBody                                        | 异步写入响应OutputStream。也支持作为ResponseEntity的主体。请参见异步请求和HTTP流。 |
| Reactive types — Reactor，RxJava，<br> or others through `ReactiveAdapterRegistry` | 可替代DeferredResult与多值流(例如，Flux, Observable)收集到一个列表。<br/>对于流场景(例如，文本/事件流，应用程序/json+流)，使用SseEmitter和ResponseBodyEmitter来代替，其中ServletOutputStream阻塞I/O在Spring mvc管理的线程上执行，并对每次写操作的完成施加反压力。请参见异步请求和响应式类型。 |
| Any other return value                                       | 任何不匹配该表中任何前面值的返回值，即String或void将被视为视图名(通过RequestToViewNameTranslator使用默认的视图名选择)，前提是它不是简单类型，由BeanUtils#isSimpleProperty确定。简单类型的值仍然无法解析。 |

#### 类型转换

一些表示基于字符串的请求输入的带注释的控制器方法参数(如@RequestParam、@RequestHeader、@PathVariable、@MatrixVariable和@CookieValue)如果参数声明为String以外的内容，则可能需要类型转换。

对于这种情况，将根据配置的转换器自动应用类型转换。默认情况下，支持简单类型(int、long、Date和其他类型)。您可以通过WebDataBinder(参见DataBinder)或通过向FormattingConversionService注册格式化器来定制类型转换。参见Spring字段格式化。

类型转换中的一个实际问题是空String源值的处理。如果此类值由于类型转换而变为空，则被视为缺失。Long、UUID和其他目标类型可以是这种情况。如果希望允许注入null，可以在参数注释上使用所需的标志，或者将参数声明为@Nullable。

> 扩展信息
>
> 从5.3开始，即使在类型转换之后，也会强制使用非空参数。如果你的处理方法也打算接受空值，要么声明你的参数为@Nullable，要么在相应的@RequestParam等注释中标记为required=false。这是一个最佳实践，也是针对5.3升级中遇到的问题的推荐解决方案。
>
> 或者，你可以专门处理例如，在需要@PathVariable的情况下产生的MissingPathVariableException。转换后的空值将被视为空的原始值，因此相应的Missing…Exception变量将被抛出。

#### 矩阵变量

RFC 3986讨论了路径段中的名称-值对。在Spring MVC中，根据Tim Berners-Lee的一篇“老文章”，我们将这些称为“矩阵变量”，但它们也可以被称为URI路径参数。

矩阵变量可以出现在任何路径段中，每个变量用分号分隔，多个值用逗号分隔(例如，/cars;color=red,green;year=2012)。还可以通过重复的变量名指定多个值(例如，color=red;color=green;color=blue)。

如果希望URL包含矩阵变量，则控制器方法的请求映射必须使用URI变量来屏蔽变量内容，并确保请求能够成功匹配，而不受矩阵变量顺序和存在性的影响。下面的例子使用了一个矩阵变量:

```java
// GET /pets/42;q=11;r=22

@GetMapping("/pets/{petId}")
public void findPet(@PathVariable String petId, @MatrixVariable int q) {

    // petId == 42
    // q == 11
}
```

考虑到所有路径段都可能包含矩阵变量，您有时可能需要澄清矩阵变量应该在哪个路径变量中。下面的例子展示了如何做到这一点:

```java
// GET /owners/42;q=11/pets/21;q=22

@GetMapping("/owners/{ownerId}/pets/{petId}")
public void findPet(
        @MatrixVariable(name="q", pathVar="ownerId") int q1,
        @MatrixVariable(name="q", pathVar="petId") int q2) {

    // q1 == 11
    // q2 == 22
}
```

矩阵变量可以定义为可选的，并指定默认值，如下例所示:

```java
// GET /pets/42

@GetMapping("/pets/{petId}")
public void findPet(@MatrixVariable(required=false, defaultValue="1") int q) {

    // q == 1
}
```

要获取所有矩阵变量，你可以使用MultiValueMap，如下例所示:

```java
// GET /owners/42;q=11;r=12/pets/21;q=22;s=23

@GetMapping("/owners/{ownerId}/pets/{petId}")
public void findPet(
        @MatrixVariable MultiValueMap<String, String> matrixVars,
        @MatrixVariable(pathVar="petId") MultiValueMap<String, String> petMatrixVars) {

    // matrixVars: ["q" : [11,22], "r" : 12, "s" : 23]
    // petMatrixVars: ["q" : 22, "s" : 23]
}
```

注意，您需要启用矩阵变量的使用。在MVC Java配置中，你需要通过路径匹配设置一个带有removeSemicolonContent=false的UrlPathHelper。在MVC XML命名空间中，你可以设置< MVC:annotation-driven enable-matrix-variables="true"/>。

#### `@RequestParam`

您可以使用@RequestParam注释将Servlet请求参数(即查询参数或表单数据)绑定到控制器中的方法参数。

下面的例子展示了如何做到这一点:

```java
@Controller
@RequestMapping("/pets")
public class EditPetForm {

    // ...

    @GetMapping
    public String setupForm(@RequestParam("petId") int petId, Model model) { ①
        Pet pet = this.clinic.loadPet(petId);
        model.addAttribute("pet", pet);
        return "petForm";
    }

    // ...

}
```

①使用@RequestParam绑定petId。

默认情况下，使用该注释的方法参数是必需的，但是您可以通过将@RequestParam注释的required标志设置为false或使用java.util.Optional包装器声明实参来指定方法参数是可选的。

如果目标方法参数类型不是String，则自动应用类型转换。参见类型转换。

将实参类型声明为数组或列表允许为相同的形参名解析多个形参值。

当@RequestParam注释声明为Map<string, string="">或MultiValueMap<string, string="">，且没有在注释中指定参数名时，则用每个给定参数名的请求参数值填充映射。</string,></string,>

注意，使用@RequestParam是可选的(例如，设置其属性)。默认情况下，任何简单值类型的参数(由BeanUtils#isSimpleProperty决定)并且没有被任何其他参数解析器解析的参数，都被视为使用@RequestParam注释的。

#### `@RequestHeader`

可以使用@RequestHeader注释将请求头绑定到控制器中的方法参数。

考虑以下带头的请求:

```
Host                    localhost:8080
Accept                  text/html,application/xhtml+xml,application/xml;q=0.9
Accept-Language         fr,en-gb;q=0.7,en;q=0.3
Accept-Encoding         gzip,deflate
Accept-Charset          ISO-8859-1,utf-8;q=0.7,*;q=0.7
Keep-Alive              300
```

下面的示例获取Accept-Encoding和Keep-Alive报头的值:

```java
@GetMapping("/demo")
public void handle(
        @RequestHeader("Accept-Encoding") String encoding, ①
        @RequestHeader("Keep-Alive") long keepAlive) { ②
    //...
}
```

①获取Accept-Encoding头的值。

②获取Keep-Alive头的值。

如果目标方法参数类型不是String，则自动应用类型转换。参见类型转换。

当在Map<String, String>`, `MultiValueMap<String, String>`或者 `HttpHeaders参数上使用@RequestHeader注释时，映射将用所有的头值填充。

> 说明
>
> 内置支持将逗号分隔的字符串转换为数组、字符串集合或类型转换系统已知的其他类型。例如，一个带有@RequestHeader("Accept")注释的方法参数可以是String类型，也可以是String[]或List<string>。

#### `@CookieValue`

可以使用@CookieValue注释将HTTP cookie的值绑定到控制器中的方法参数。

考虑一个带有以下cookie的请求:

```
JSESSIONID=415A4AC178C59DACE0B2C9CA727CDD84
```

下面的例子展示了如何获取cookie值:

```java
@GetMapping("/demo")
public void handle(@CookieValue("JSESSIONID") String cookie) { ①
    //...
}
```

①获取JSESSIONID cookie的值。

如果参数不是string类型，将会自动进行类型转换。参见类型转换。

#### `@ModelAttribute`

您可以在方法参数上使用@ModelAttribute注释来从模型中访问属性，或者在属性不存在时实例化它。模型属性还覆盖了来自HTTP Servlet请求参数的值，这些参数的名称与字段名匹配。这被称为数据绑定，它使您不必处理解析和转换单个查询参数和表单字段。下面的例子展示了如何做到这一点:

```java
@PostMapping("/owners/{ownerId}/pets/{petId}/edit")
public String processSubmit(@ModelAttribute Pet pet) {
    // method logic...
}
```

Pet实例是以下几种方式之一实现的：

- 从可能通过@ModelAttribute方法添加它的模型中检索。
- 如果模型属性列在类级别的@SessionAttributes注释中，则从HTTP会话检索。
- 通过Converter获得，其中模型属性名称与请求值的名称相匹配，例如路径变量或请求参数(参见下一个示例)。
- 使用其默认构造函数实例化。
- 通过带有与Servlet请求参数匹配的参数的“主构造函数”实例化。参数名是通过JavaBeans @ConstructorProperties或字节码中运行时保留的参数名确定的。

除了使用@ModelAttribute方法来提供它或依赖框架来创建模型属性之外，另一种选择是使用Converter<string, T>来提供实例。当模型属性名称与请求值(如路径变量或请求参数)的名称相匹配，并且存在从String到模型属性类型的Converter时，就会应用此方法。在下面的例子中，模型属性名称是account，它匹配URI路径变量account，并且有一个注册的Converter<string, Account>，它可以从数据存储加载account:

```java
@PutMapping("/accounts/{account}")
public String save(@ModelAttribute("account") Account account) {
    // ...
}
```

获得模型属性实例后，应用数据绑定。WebDataBinder类将Servlet请求参数名(查询参数和表单字段)与目标对象上的字段名匹配。在需要时，在应用类型转换后填充匹配字段。有关数据绑定(和验证)的更多信息，请参见验证。有关自定义数据绑定的更多信息，请参见DataBinder。

数据绑定可能会导致错误。默认情况下，会引发BindException异常。然而，要在控制器方法中检查此类错误，您可以在@ModelAttribute旁边立即添加一个BindingResult参数，如下例所示:

```java
@PostMapping("/owners/{ownerId}/pets/{petId}/edit")
public String processSubmit(@ModelAttribute("pet") Pet pet, BindingResult result) { ①
    if (result.hasErrors()) {
        return "petForm";
    }
    // ...
}
```

①@ModelAttribute注解后边添加了一个BindingResult

在某些情况下，您可能希望在没有数据绑定的情况下访问模型属性。对于这种情况，您可以将Model注入到控制器中并直接访问它，或者，设置@ModelAttribute(binding=false)，如下例所示:

```java
@ModelAttribute
public AccountForm setUpForm() {
    return new AccountForm();
}

@ModelAttribute
public Account findAccount(@PathVariable String accountId) {
    return accountRepository.findOne(accountId);
}

@PostMapping("update")
public String update(@Valid AccountForm form, BindingResult result,
        @ModelAttribute(binding=false) Account account) { ①
    // ...
}
```

①设置@ModelAttribute(binding=false)

通过添加javax.validation.Valid注释或Spring的@Validated注释(Bean验证和Spring验证)，您可以在数据绑定之后自动应用验证。下面的例子展示了如何做到这一点:

```java
@PostMapping("/owners/{ownerId}/pets/{petId}/edit")
public String processSubmit(@Valid @ModelAttribute("pet") Pet pet, BindingResult result) { ①
    if (result.hasErrors()) {
        return "petForm";
    }
    // ...
}
```

①检验pet实例

注意，使用@ModelAttribute是可选的(例如，设置其属性)。默认情况下，任何不是简单值类型的参数(由BeanUtils#isSimpleProperty确定)并且没有被任何其他参数解析器解析的参数都被视为使用@ModelAttribute注释的。

#### `@SessionAttributes`

#### `@SessionAttribute`

#### `@RequestAttribute`

####  重定向属性

#### Flash属性

#### Multipart

#### `@RequestBody`

您可以使用@RequestBody注释通过HttpMessageConverter读取请求主体并将其反序列化为对象。下面的例子使用了@RequestBody参数:

```java
@PostMapping("/accounts")
public void handle(@RequestBody Account account) {
    // ...
}
```

您可以使用MVC配置的消息转换器选项来配置或自定义消息转换。

您可以将@RequestBody与javax.validation.Valid或Spring的@Validated注释结合使用，这两者都会导致应用标准Bean验证。默认情况下，验证错误会导致methodargumentnotvalideexception，该异常被转换为400 (BAD_REQUEST)响应。或者，你可以通过errors或BindingResult参数在控制器内部本地处理验证错误，如下例所示:

```java
@PostMapping("/accounts")
public void handle(@Valid @RequestBody Account account, BindingResult result) {
    // ...
}
```

#### HttpEntity

HttpEntity或多或少与使用@RequestBody相同，但它基于一个公开请求头和请求体的容器对象。下面的清单显示了一个示例:

```java
@PostMapping("/accounts")
public void handle(HttpEntity<Account> entity) {
    // ...
}
```

#### `@ResponseBody`

您可以在方法上使用@ResponseBody注释，通过HttpMessageConverter将返回序列化到响应体。下面的清单显示了一个示例:

```java
@GetMapping("/accounts/{id}")
@ResponseBody
public Account handle() {
    // ...
}
```

在类级别上也支持@ResponseBody，在这种情况下，它被所有控制器方法继承。这就是@RestController的效果，它不过是一个用@Controller和@ResponseBody标记的元注释。

您可以将@ResponseBody与响应类型一起使用。有关详细信息，请参阅异步请求和响应式类型。

您可以使用MVC配置的消息转换器选项来配置或自定义消息转换。

可以将@ResponseBody方法与JSON序列化视图组合在一起。详细信息请参见Jackson JSON。

#### ResponseEntity

ResponseEntity类似于@ResponseBody，但带有状态和头文件。例如:

```java
@GetMapping("/something")
public ResponseEntity<String> handle() {
    String body = ... ;
    String etag = ... ;
    return ResponseEntity.ok().eTag(etag).body(body);
}
```

Spring MVC支持使用单值响应类型异步生成ResponseEntity，以及/或主体的单值和多值响应类型。这允许以下类型的异步响应:

- ResponseEntity<Mono<T>> or ResponseEntity<Flux<T>> 使响应状态和标头立即可知，而主体稍后将异步提供。如果主体包含0..1个值或Flux(如果它可以产生多个值)。
- Mono<ResponseEntity<T>> 提供所有这三种功能——响应状态、报头和正文，稍后将异步提供。这允许响应状态和报头根据异步请求处理的结果而变化。

#### Jackson JSON

Spring提供对Jackson JSON库的支持。

**JSON Views**

Spring MVC为Jackson的序列化视图提供了内置支持，它只允许呈现对象中所有字段的一个子集。要将它与@ResponseBody或ResponseEntity控制器方法一起使用，您可以使用Jackson的@JsonView注释来激活序列化视图类，如下面的示例所示:

```java
@RestController
public class UserController {

    @GetMapping("/user")
    @JsonView(User.WithoutPasswordView.class)
    public User getUser() {
        return new User("eric", "7!jd#h23");
    }
}

public class User {

    public interface WithoutPasswordView {};
    public interface WithPasswordView extends WithoutPasswordView {};

    private String username;
    private String password;

    public User() {
    }

    public User(String username, String password) {
        this.username = username;
        this.password = password;
    }

    @JsonView(WithoutPasswordView.class)
    public String getUsername() {
        return this.username;
    }

    @JsonView(WithPasswordView.class)
    public String getPassword() {
        return this.password;
    }
}
```

> @JsonView允许一个视图类数组，但是每个控制器方法只能指定一个。如果需要激活多个视图，可以使用复合接口。

如果你想以编程的方式完成上述操作，而不是声明一个@JsonView注释，用MappingJacksonValue包装返回值并使用它来提供序列化视图:

```java
@RestController
public class UserController {

    @GetMapping("/user")
    public MappingJacksonValue getUser() {
        User user = new User("eric", "7!jd#h23");
        MappingJacksonValue value = new MappingJacksonValue(user);
        value.setSerializationView(User.WithoutPasswordView.class);
        return value;
    }
}
```

对于依赖于视图解析的控制器，可以将序列化视图类添加到模型中，如下例所示:

```java
@Controller
public class UserController extends AbstractController {

    @GetMapping("/user")
    public String getUser(Model model) {
        model.addAttribute("user", new User("eric", "7!jd#h23"));
        model.addAttribute(JsonView.class.getName(), User.WithoutPasswordView.class);
        return "userView";
    }
}
```



