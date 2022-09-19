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

> Spring Boot遵循不同的初始化顺序。Spring Boot没有与Servlet容器的生命周期挂钩，而是使用Spring配置来引导自身和嵌入的Servlet容器。筛选器和Servlet声明在Spring配置中检测到，并注册到Servlet容器中。要了解更多细节，请参阅Spring Boot文档。