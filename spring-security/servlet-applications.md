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

# Servlet Authentication Architecture

本讨论扩展了 Servlet Security: The Big Picture 以描述 Spring Security 在 Servlet 身份验证中使用的主要架构组件。 如果您需要解释这些部分如何组合在一起的具体流程，请查看身份验证机制特定部分。

- SecurityContextHolder :SecurityContextHolder 是 Spring Security 存储身份验证者详细信息的地方。
- SecurityContext ：从 SecurityContextHolder 中获取，包含当前经过身份验证的用户的 Authentication。
- Authentication ：可以是 AuthenticationManager 的输入，以提供用户提供的凭据以进行身份验证，也可以是 SecurityContext 中的当前用户。
- GrantedAuthority ：在身份验证上授予主体的权限（即角色、范围等）
- AuthenticationManager ：定义 Spring Security Filters 如何执行身份验证的 API。
- ProviderManager：AuthenticationManager 最常见的实现。
- AuthenticationProvider ：ProviderManager 使用它来执行特定类型的身份验证。
- 使用 AuthenticationEntryPoint 请求凭据 ：用于从客户端请求凭据（即重定向到登录页面、发送 WWW-Authenticate 响应等）
- AbstractAuthenticationProcessingFilter ：用于身份验证的基本过滤器。 这也很好地了解了身份验证的高级流程以及各个部分如何协同工作。

## SecurityContextHolder
Spring Security 认证模型的核心是 SecurityContextHolder。 它包含 SecurityContext。
SecurityContextHolder 是 Spring Security 存储身份验证者详细信息的地方。 Spring Security 不关心 SecurityContextHolder 是如何填充的。 如果它包含一个值，则将其用作当前经过身份验证的用户。

指示用户已通过身份验证的最简单方法是直接设置 SecurityContextHolder。

默认情况下，SecurityContextHolder 使用 ThreadLocal 来存储这些详细信息，这意味着 SecurityContext 始终可用于同一线程中的方法，即使 SecurityContext 没有明确地作为参数传递给这些方法。 如果在处理当前主体的请求后注意清除线程，那么以这种方式使用 ThreadLocal 是非常安全的。 Spring Security 的 FilterChainProxy 确保 SecurityContext 总是被清除。

有些应用程序并不完全适合使用 ThreadLocal，因为它们使用线程的特定方式。 例如，Swing 客户端可能希望 Java 虚拟机中的所有线程使用相同的安全上下文。 SecurityContextHolder 可以在启动时配置一个策略，以指定您希望如何存储上下文。 对于独立应用程序，您将使用 SecurityContextHolder.MODE_GLOBAL 策略。 其他应用程序可能希望安全线程产生的线程也采用相同的安全身份。 这是通过使用 SecurityContextHolder.MODE_INHERITABLETHREADLOCAL 来实现的。 您可以通过两种方式从默认的 SecurityContextHolder.MODE_THREADLOCAL 更改模式。 第一个是设置系统属性，第二个是调用SecurityContextHolder的静态方法。 大多数应用程序不需要更改默认设置，但如果您这样做，请查看 Javadoc for SecurityContextHolder 以了解更多信息。

## SecurityContext

SecurityContext 是从 SecurityContextHolder 获得的。 SecurityContext 包含一个 Authentication 对象。

## Authentication

Authentication在 Spring Security 中有两个主要目的：
- AuthenticationManager 的输入，用于提供用户提供的身份验证凭据。 在这种情况下使用时，isAuthenticated() 返回 false。
- 表示当前经过身份验证的用户。 当前的Authentication可以从SecurityContext中获取。

Authentication包含：
- principal ：用户标识。 当使用用户名/密码进行身份验证时，这通常是 UserDetails 的一个实例。
- credentials ：通常是密码。 在许多情况下，这将在用户通过身份验证后被清除，以确保它不被泄露。
- authorities ：GrantedAuthoritys 是授予用户的高级权限。 一些示例是角色或范围。

## GrantedAuthority

GrantedAuthoritys 是授予用户的高级权限。一些示例是角色或范围。

GrantedAuthoritys 可以从 Authentication.getAuthorities() 方法中获得。此方法提供 GrantedAuthority 对象的 Collection。毫不奇怪，GrantedAuthority 是授予委托人的权限。此类权限通常是“角色”，例如 ROLE_ADMINISTRATOR 或 ROLE_HR_SUPERVISOR。这些角色稍后会针对 Web 授权、方法授权和域对象授权进行配置。 Spring Security 的其他部分能够解释这些权限，并期望它们存在。当使用基于用户名/密码的身份验证时，GrantedAuthoritys 通常由 UserDetailsS​​ervice 加载。

通常，GrantedAuthority 对象是应用程序范围的权限。它们并不特定于给定的域对象。因此，您不可能有一个 GrantedAuthority 来表示对 Employee 对象编号 54 的权限，因为如果有数千个这样的权限，您将很快耗尽内存（或者，至少会导致应用程序花费很长时间验证用户的时间）。当然，Spring Security 是专门为处理这种常见需求而设计的，但您应该为此目的使用项目的域对象安全功能。

## AuthenticationManager

AuthenticationManager 是定义 Spring Security 的过滤器如何执行身份验证的 API。 返回的 Authentication 然后由调用 AuthenticationManager 的控制器（即 Spring Security 的 Filterss）在 SecurityContextHolder 上设置。 如果你没有与 Spring Security 的 Filterss 集成，你可以直接设置 SecurityContextHolder 并且不需要使用 AuthenticationManager。

虽然 AuthenticationManager 的实现可以是任何东西，但最常见的实现是 ProviderManager。

## ProviderManager

ProviderManager 是 AuthenticationManager 最常用的实现。 ProviderManager 委托给一个 AuthenticationProviders 列表。 每个 AuthenticationProvider 都有机会指示身份验证应该成功、失败，或指示它不能做出决定并允许下游 AuthenticationProvider 做出决定。 如果配置的 AuthenticationProviders 都不能进行身份验证，则身份验证将失败并出现 ProviderNotFoundException，这是一个特殊的 AuthenticationException，表明 ProviderManager 未配置为支持传递给它的身份验证类型。

实际上，每个 AuthenticationProvider 都知道如何执行特定类型的身份验证。 例如，一个 AuthenticationProvider 可能能够验证用户名/密码，而另一个可能能够验证 SAML 断言。 这允许每个 AuthenticationProvider 执行非常特定类型的身份验证，同时支持多种类型的身份验证并且只公开单个 AuthenticationManager bean。

ProviderManager 还允许配置一个可选的父 AuthenticationManager，在没有 AuthenticationProvider 可以执行身份验证的情况下进行咨询。 父级可以是任何类型的 AuthenticationManager，但它通常是 ProviderManager 的一个实例。

事实上，多个 ProviderManager 实例可能共享同一个父 AuthenticationManager。 这在有多个 SecurityFilterChain 实例具有一些共同的身份验证（共享父 AuthenticationManager）但也有不同的身份验证机制（不同的 ProviderManager 实例）的场景中有些常见。

默认情况下，ProviderManager 将尝试从成功的身份验证请求返回的 Authentication 对象中清除任何敏感的凭据信息。 这可以防止诸如密码之类的信息在 HttpSession 中保留的时间超过必要的时间。

当您使用用户对象的缓存时，这可能会导致问题，例如，为了提高无状态应用程序的性能。 如果 Authentication 包含对缓存中的对象（例如 UserDetails 实例）的引用，并且已删除其凭据，则将不再可能针对缓存的值进行身份验证。 如果您使用缓存，则需要考虑到这一点。 一个明显的解决方案是首先在缓存实现中或在创建返回的 Authentication 对象的 AuthenticationProvider 中制作对象的副本。 或者，您可以禁用 ProviderManager 上的 eraseCredentialsAfterAuthentication 属性。 有关更多信息，请参阅 Javadoc。

## AuthenticationProvider

可以将多个 AuthenticationProviders 注入 ProviderManager。 每个 AuthenticationProvider 执行特定类型的身份验证。 例如，DaoAuthenticationProvider 支持基于用户名/密码的身份验证，而 JwtAuthenticationProvider 支持对 JWT 令牌进行身份验证。

## 使用 AuthenticationEntryPoint 请求凭据

AuthenticationEntryPoint 用于发送从客户端请求凭据的 HTTP 响应。

有时，客户端会主动包含凭据（例如用户名/密码）来请求资源。 在这些情况下，Spring Security 不需要提供从客户端请求凭据的 HTTP 响应，因为它们已经包含在内。

在其他情况下，客户端将对他们无权访问的资源发出未经身份验证的请求。 在这种情况下，AuthenticationEntryPoint 的实现用于从客户端请求凭据。 AuthenticationEntryPoint 实现可能会重定向到登录页面，使用 WWW-Authenticate 标头等进行响应。

## AbstractAuthenticationProcessingFilter

AbstractAuthenticationProcessingFilter 用作验证用户凭据的基本过滤器。 在可以对凭据进行身份验证之前，Spring Security 通常使用 AuthenticationEntryPoint 请求凭据。

接下来，AbstractAuthenticationProcessingFilter 可以对提交给它的任何身份验证请求进行身份验证。






















