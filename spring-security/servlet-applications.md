# 1.Servlet应用

Spring Security通过使用标准的Servlet过滤器与Servlet容器集成。这意味着它可以与在Servlet容器中运行的任何应用程序一起工作。
更具体地说，您不需要在基于Servlet的应用程序中的Spring来利用Spring Security。

## 1.1.整体架构

本节讨论了Spring Security在基于Servlet的应用程序中的高级体系结构。我们在参考的身份验证、授权和防范漏洞部分中建立在这一高级理解的基础上。

### 1.1.1. 过滤器回顾

Spring Security的Servlet支持是基于Servlet过滤器，因此首先了解筛选器的作用是很有帮助的。下图显示了单个HTTP请求的处理程序的典型分层。

![filterchain](filterchain.png)

客户端向应用程序发送请求，容器创建一个FilterChain，其中包含过滤器和Servlet，这些过滤器和Servlet应根据请求URI的路径处理HttpServletRequest。在Spring MVC应用程序中，Servlet是Dispatcher Servlet的实例。最多一个Servlet可以处理单个HttpServletRequest和HttpServletResponse。但是，可以使用多个过滤器来执行以下操作：

- 防止调用下游过滤器或Servlet。在这种情况下，过滤器通常会写入HttpServletResponse。
- 修改下游过滤器和Servlet使用的HttpServletRequest或HttpServletResponse。

Filter的功能来自传递给它的FilterChain。

```java
public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) {
	// do something before the rest of the application
    chain.doFilter(request, response); // invoke the rest of the application
    // do something after the rest of the application
}
```

因为过滤器只影响下游过滤器和Servlet，所以每个过滤器的调用顺序是非常重要的。

### 1.1.2. DelegatingFilterProxy

Spring提供了一个名为DelegatingFilterProxy的Filter实现，它允许Servlet容器的生命周期和Spring的ApplicationContext之间的桥接。Servlet容器允许使用它自己的标准注册过滤器，但是它不知道Spring定义的bean。DelegatingFilterProxy可以通过标准的Servlet容器机制注册，但是将所有工作委派给实现Filter的Spring Bean。

下面是一幅如何将DelegatingFilterProxy适配到Filters和FilterChain中的图片。

![delegatingfilterproxy](delegatingfilterproxy.png)

DelegatingFilterProxy从ApplicationContext查找Bean Filter0，然后调用Bean Filter0。下面是DelegatingFilterProxy的伪代码。

```java
public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) {
	// Lazily get Filter that was registered as a Spring Bean
	// For the example in DelegatingFilterProxy 
delegate
 is an instance of Bean Filter0
	Filter delegate = getFilterBean(someBeanName);
	// delegate work to the Spring Bean
	delegate.doFilter(request, response);
}
```

DelegatingFilterProxy的另一个好处是它允许延迟查找Filter bean实例。这很重要，因为容器需要在启动容器之前注册Filter实例。然而，Spring通常使用ContextLoaderListener来加载Spring bean，这在需要注册Filter实例之后才会完成。

### 1.1.3.FilterChainProxy

Spring Security的Servlet支持包含在FilterChainProxy中。FilterChainProxy是Spring Security提供的一个特殊的过滤器，它允许通过SecurityFilterChain向多个Filter实例进行委托。由于FilterChainProxy是一个Bean，它通常被包装在DelegatingFilterProxy中。

![filterchainproxy](filterchainproxy.png)

### 1.1.4.SecurityFilterChain

FilterChainProxy使用SecurityFilterChain来确定应该为这个请求调用哪个Spring安全过滤器。

![securityfilterchain](securityfilterchain.png)

SecurityFilterChain中的安全过滤器通常是bean，但它们是用FilterChainProxy注册的，而不是DelegatingFilterProxy。FilterChainProxy为直接注册Servlet容器或DelegatingFilterProxy提供了许多优点。首先，它为Spring Security的所有Servlet支持提供了一个起点。因此，如果您试图排除Spring Security的Servlet支持的故障，在FilterChainProxy中添加一个调试点是一个很好的开始。

其次，由于FilterChainProxy是Spring Security使用的核心，它可以执行一些不被视为可选的任务。例如，它清除SecurityContext以避免内存泄漏。它还应用Spring Security的HttpFirewall来保护应用程序免受某些类型的攻击。

此外，它在确定何时调用SecurityFilterChain方面提供了更大的灵活性。在Servlet容器中，仅根据URL调用过滤器。然而，FilterChainProxy可以通过利用RequestMatcher接口基于HttpServletRequest中的任何内容来确定调用。

事实上，FilterChainProxy可以用来确定应该使用哪个SecurityFilterChain。这允许为应用程序的不同部分提供完全独立的配置。

![multi-securityfilterchain](multi-securityfilterchain.png)

在多个SecurityFilterChain图中，FilterChainProxy决定应该使用哪个SecurityFilterChain。只调用第一个匹配的SecurityFilterChain。如果一个/api/messages/的URL被请求，它将首先匹配SecurityFilterChain0的模式/api/\*\*，因此只有SecurityFilterChain0将被调用，即使它也匹配SecurityFilterChainn。如果一个/messages/的URL被请求，它将不匹配SecurityFilterChain0的模式/api/\*\*，因此FilterChainProxy将继续尝试每个SecurityFilterChain。假设没有其他，将调用与SecurityFilterChainn匹配的SecurityFilterChain实例。

注意，SecurityFilterChain0只配置了三个安全过滤器实例。然而，SecurityFilterChainn配置了四个安全过滤器。需要注意的是，每个SecurityFilterChain可以是唯一的，并且可以单独配置。事实上，如果应用程序希望Spring security忽略某些请求，那么SecurityFilterChain可能没有安全过滤器。

### 1.1.5.Security过滤器

安全过滤器通过SecurityFilterChain API被插入到FilterChainProxy中。过滤器的顺序很重要。通常不需要知道Spring Security的过滤器的顺序。然而，有时了解顺序是有益的

以下是Spring Security Filter的清单:

- [`ForceEagerSessionCreationFilter`](https://docs.spring.io/spring-security/reference/servlet/authentication/session-management.html#session-mgmt-force-session-creation)
- ChannelProcessingFilter
- WebAsyncManagerIntegrationFilter
- SecurityContextPersistenceFilter
- HeaderWriterFilter
- CorsFilter
- CsrfFilter
- LogoutFilter
- OAuth2AuthorizationRequestRedirectFilter
- Saml2WebSsoAuthenticationRequestFilter
- X509AuthenticationFilter
- AbstractPreAuthenticatedProcessingFilter
- CasAuthenticationFilter
- OAuth2LoginAuthenticationFilter
- Saml2WebSsoAuthenticationFilter
- [`UsernamePasswordAuthenticationFilter`](https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/form.html#servlet-authentication-usernamepasswordauthenticationfilter)
- OpenIDAuthenticationFilter
- DefaultLoginPageGeneratingFilter
- DefaultLogoutPageGeneratingFilter
- ConcurrentSessionFilter
- [`DigestAuthenticationFilter`](https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/digest.html#servlet-authentication-digest)
- BearerTokenAuthenticationFilter
- [`BasicAuthenticationFilter`](https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/basic.html#servlet-authentication-basic)
- RequestCacheAwareFilter
- SecurityContextHolderAwareRequestFilter
- JaasApiIntegrationFilter
- RememberMeAuthenticationFilter
- AnonymousAuthenticationFilter
- OAuth2AuthorizationCodeGrantFilter
- SessionManagementFilter
- [`ExceptionTranslationFilter`](https://docs.spring.io/spring-security/reference/servlet/architecture.html#servlet-exceptiontranslationfilter)
- [`FilterSecurityInterceptor`](https://docs.spring.io/spring-security/reference/servlet/authorization/authorize-requests.html#servlet-authorization-filtersecurityinterceptor)
- SwitchUserFilter

### 1.1.6.Security 的异常处理

ExceptionTranslationFilter允许将AccessDeniedException和AuthenticationException转换为HTTP响应。

ExceptionTranslationFilter被插入到FilterChainProxy中，作为安全过滤器之一。

![exceptiontranslationfilter](exceptiontranslationfilter.png)

- ①首先，ExceptionTranslationFilter调用FilterChain。doFilter(request, response)调用应用程序的剩余部分。
- ②如果用户没有经过身份验证或它是一个AuthenticationException，则启动身份验证。
  - SecurityContextHolder 被清除
  - HttpServletRequest保存在RequestCache中。当用户成功进行身份验证时，使用RequestCache重放原始请求。
  - AuthenticationEntryPoint用于从客户端请求凭据。例如，它可能重定向到登录页面或发送WWW-Authenticate报头。
- ③否则，如果是AccessDeniedException，则为AccessDenied。调用AccessDeniedHandler来处理拒绝访问。

> 扩展信息
>
> 如果应用程序不抛出AccessDeniedException或AuthenticationException，则ExceptionTranslationFilter不做任何事情。

ExceptionTranslationFilter的伪代码看起来像这样:

```java
try {
	filterChain.doFilter(request, response); ①
} catch (AccessDeniedException | AuthenticationException ex) {
	if (!authenticated || ex instanceof AuthenticationException) {
		startAuthentication(); ②
	} else {
		accessDenied(); ③
	}
}
```

①您可以回忆一下《过滤器回顾》中调用FilterChain的内容。doFilter(请求、响应)相当于调用应用程序的剩余部分。这意味着如果应用程序的另一部分(例如FilterSecurityInterceptor或方法安全性)抛出AuthenticationException或AccessDeniedException，它将在这里被捕获和处理。

②如果用户没有经过身份验证或它是一个AuthenticationException，则启动身份验证。

③否则,拒绝访问

## 1.2. 身份验证（Authentication）

Spring Security为身份验证提供了全面的支持。我们首先讨论整体的Servlet身份验证体系结构。如您所料，本节对体系结构的描述更加抽象，而没有详细讨论它如何应用于具体流。

如果您愿意，您可以参考身份验证机制，了解用户进行身份验证的具体方法。这些部分着重于您可能希望进行身份验证的特定方式，并指向体系结构部分，以描述特定流是如何工作的。

