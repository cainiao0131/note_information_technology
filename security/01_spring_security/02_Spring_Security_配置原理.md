# Spring Security 配置原理

## 总览

### 配置原理

配置 Spring Security 就是构建 FilterChainProxy 对象，由 WebSecurity（SecurityBuilder<Filter&gt;）构建  
WebSecurity 持有一个 configurer 列表  
对应字段：LinkedHashMap<Class<? extends SecurityConfigurer<O, B>>, List<SecurityConfigurer<O, B>>> configurers  
由这些 configurer 来配置 WebSecurity  
WebSecurityConfigurerAdapter 就是一个 configurer（SecurityConfigurer<Filter, WebSecurity>）  

> 注意：WebSecurityConfigurerAdapter 必须是 Spring 容器中的 Bean，否则不会加入到 configurers 中

FilterChainProxy 会把任务委派给 List<SecurityFilterChain&gt; filterChains  
所以构建 FilterChainProxy 时，需要构建这个 filterChains  
SecurityFilterChain 由 WebSecurityConfigurerAdapter 中的 HttpSecurity http 构建  
HttpSecurity 是一个构建者：SecurityBuilder<DefaultSecurityFilterChain&gt;

- WebSecurity（SecurityBuilder<Filter&gt;）  
  持有：configurers   
  对应多个：WebSecurityConfigurerAdapter（SecurityConfigurer<Filter, WebSecurity>）  
  对应多个：HttpSecurity  
- HttpSecurity（SecurityBuilder<DefaultSecurityFilterChain&gt;）  
  持有：configurers  
  对应多个：SecurityConfigurer<DefaultSecurityFilterChain, HttpSecurity>>（容器中会提供一个默认的配置器列表）

WebSecurityConfigurerAdapter 中重写的方法：  
- configure(HttpSecurity http)  
  HttpSecurity 即上边提到的用于构建 SecurityFilterChain 的构建者：SecurityBuilder<DefaultSecurityFilterChain&gt;  
  用来构建 FilterChainProxy 中的其中一个 SecurityFilterChain
- configure(WebSecurity web)  
  WebSecurity 即上边提到的用于构建 FilterChainProxy 的构建者：SecurityBuilder<Filter&gt;  

客户端程序员，每多定义一个 WebSecurityConfigurerAdapter（SecurityConfigurer<Filter, WebSecurity>） 的 Spring Bean  
WebSecurity（SecurityBuilder<Filter&gt;）的 configurers 就多一个 SecurityConfigurer<Filter, WebSecurity>  
也就多了一个 SecurityConfigurer.http（HttpSecurity（SecurityBuilder<DefaultSecurityFilterChain&gt;））  
WebSecurity.securityFilterChainBuilders 中就多一个 HttpSecurity（SecurityBuilder<DefaultSecurityFilterChain&gt;）

最终，webSecurity.build() 构建 FilterChainProxy 时  
FilterChainProxy 中的 securityFilterChains 就多一个 SecurityFilterChain

一个请求，只会被 FilterChainProxy 中的一个 SecurityFilterChain 匹配到  
SecurityFilterChain 对应着一套安全策略  
因此，只有在系统中需要配置多套安全策略时，才需要配置多个自定义的 WebSecurityConfigurerAdapter

### 配置过程

WebSecurityConfiguration 类
1. setFilterChainProxySecurityConfigurer()
   1. new 了一个 WebSecurity（SecurityBuilder&lt;Filter&gt;） 对象，赋值给 this.webSecurity  
   1. 将注入的 List<SecurityConfigurer<Filter, WebSecurity>> webSecurityConfigurers 加入到 webSecurity 的  
   LinkedHashMap<Class<? extends SecurityConfigurer<Filter, WebSecurity>>, List<SecurityConfigurer<Filter, WebSecurity>>>
   configurers  
      其中包括自定义的 WebSecurityConfigurerAdapter
1. springSecurityFilterChain()  
   调用 this.webSecurity.build() 构建 FilterChainProxy 对象  
   1. 实际调用的是 AbstractSecurityBuilder<Filter&gt; 的 build() 方法，防止重复构建  
   1. 然后调用了 AbstractConfiguredSecurityBuilder<Filter, WebSecurity> 的 doBuild() 构建 FilterChainProxy 对象  
      AbstractConfiguredSecurityBuilder<Filter, WebSecurity>.doBuild()：
      1. init()（AbstractConfiguredSecurityBuilder 私有）  
         遍历调用 configurers 的 init() 方法  
         包括：WebSecurityConfigurerAdapter（SecurityConfigurer<Filter, WebSecurity>）  
         WebSecurityConfigurerAdapter.init()：  
            1. 调用 getHttp() 获得一个 HttpSecurity（SecurityBuilder&lt;DefaultSecurityFilterChain&gt;） 对象  
            getHttp() 方法：  
               1. this.http 不为 null，则返回 this.http，否则：  
               1. new 一个 HttpSecurity（SecurityBuilder&lt;DefaultSecurityFilterChain&gt;）赋值给 this.http  
               1. 将多个 SecurityConfigurer<DefaultSecurityFilterChain, HttpSecurity>> 加入 http 持有的 configurers  
               1. 调用 configure(this.http)（客户端程序员可重写）  
            1. 将 http 加入 WebSecurity 的 List<SecurityBuilder<? extends SecurityFilterChain>> securityFilterChainBuilders
      1. configure()（AbstractConfiguredSecurityBuilder 私有）  
         遍历调用 configurers 的 configure(WebSecurity web) 方法  
         包括 WebSecurityConfigurerAdapter（SecurityConfigurer<Filter, WebSecurity>）  
         即：WebSecurityConfigurerAdapter.configure(WebSecurity web)（客户端程序员可重写）
      1. performBuild()（抽象方法，实际由 WebSecurity 的 performBuild() 方法实现）  
         new 了一个 FilterChainProxy(securityFilterChains)  
         注入了一个 List&lt;SecurityFilterChain&gt; securityFilterChains 列表  
         securityFilterChains 中的 SecurityFilterChain 由 securityFilterChainBuilders 中的  
         SecurityBuilder<? extends SecurityFilterChain> 构建  
         即：HttpSecurity（SecurityBuilder<DefaultSecurityFilterChain&gt;）

### 设计模式

1. 模板方法  
   AbstractSecurityBuilder 的 build() 防止重复构建  
   实际调用的是 AbstractConfiguredSecurityBuilder 的 doBuild()  
   而 doBuild() 有多个固定逻辑：init()、configure()、performBuild()  
   其中 performBuild() 为抽象方法，由 WebSecurity 重写
1. 构建者  
   WebSecurity（SecurityBuilder<Filter&gt;）  
   HttpSecurity（SecurityBuilder<DefaultSecurityFilterChain&gt;）
1. 状态机  
   AbstractConfiguredSecurityBuilder.doBuild() 中有多个状态的设置操作  
   private BuildState buildState = BuildState.UNBUILT;

## 源码

**(1) 接入点**

只要引入了 Spring Security 包  
类路径下就会有 WebSecurityConfiguration

~~~ java
package org.springframework.security.config.annotation.web.configuration;

@Configuration(proxyBeanMethods = false)
public class WebSecurityConfiguration
    implements ImportAware, BeanClassLoaderAware {
    
    private WebSecurity webSecurity;
    ...
    // 构建 FilterChainProxy
    @Bean(name = AbstractSecurityWebApplicationInitializer.DEFAULT_FILTER_NAME)
	public Filter springSecurityFilterChain() throws Exception {
	    ...
	    return this.webSecurity.build();
	}
	...
	
	// 设置 webSecurity
	@Autowired(required = false)
	public void setFilterChainProxySecurityConfigurer(ObjectPostProcessor<Object> objectPostProcessor,
		@Value("#{@autowiredWebSecurityConfigurersIgnoreParents.getWebSecurityConfigurers()}") List<SecurityConfigurer<Filter, WebSecurity>> webSecurityConfigurers)
		throws Exception {
		
		this.webSecurity = objectPostProcessor.postProcess(new WebSecurity(objectPostProcessor));
		...
		for (SecurityConfigurer<Filter, WebSecurity> webSecurityConfigurer : webSecurityConfigurers) {
			this.webSecurity.apply(webSecurityConfigurer);
		}
		this.webSecurityConfigurers = webSecurityConfigurers;
	}
}
~~~

WebSecurityConfiguration 加了 @Configuration 注解  
Spring Boot 启动时会根据其中加了 @Bean 注解的方法创建对象（前提是开启了注解扫描）  
因此会根据 springSecurityFilterChain() 方法创建 FilterChainProxy 对象  
这里就是通过 this.webSecurity.build() 构建的

因为 @Configuration 自己也是一个 @Component，即，一个 Spring 容器中的 Bean  
因此会由 Spring 容器进行装配（[依赖注入](../../spring/01_Spring_IoC_与_DI.md)）  
因此，被 @Autowired 标记的方法 setFilterChainProxySecurityConfigurer() 会被执行  
setFilterChainProxySecurityConfigurer() 的方法参数由依赖注入设置  
可见，这里就是 new 了一个 WebSecurity 赋值给 this.webSecurity  
并且为 webSecurity 设置了 webSecurityConfigurers（客户端程序员定义的 WebSecurityConfigurerAdapter 包括在其中）

**(2) WebSecurity 构建 FilterChainProxy**

this.webSecurity.build() 构建 Filter 对象（FilterChainProxy 对象）  
即，WebSecurity 的 build() 方法

~~~ java
package org.springframework.security.config.annotation.web.builders;

public final class WebSecurity extends AbstractConfiguredSecurityBuilder<Filter, WebSecurity>
	implements SecurityBuilder<Filter>, ApplicationContextAware {
    
    ...
}

/*
WebSecurity 继承了 AbstractConfiguredSecurityBuilder<Filter, WebSecurity>
AbstractConfiguredSecurityBuilder<Filter, WebSecurity> 继承了 AbstractSecurityBuilder<Filter>
*/

package org.springframework.security.config.annotation;

public abstract class AbstractSecurityBuilder<O> implements SecurityBuilder<O> {

	private AtomicBoolean building = new AtomicBoolean();

	private O object;

   @Override
   public final O build() throws Exception {
      // 防止重复构建
      if (this.building.compareAndSet(false, true)) {
         // AbstractConfiguredSecurityBuilder.doBuild()
         this.object = doBuild();
         return this.object;
      }
      throw new AlreadyBuiltException("This object has already been built");
   }
   ...
   protected abstract O doBuild() throws Exception;
}
~~~

WebSecurity 的 build()，实际调用的是 AbstractSecurityBuilder 的 build() 方法  
这个方法的作用，只是为了保证对象只被构建一次（否则抛异常）  
实际的构建逻辑，是 AbstractConfiguredSecurityBuilder 重写的 doBuild() 方法

~~~ java
package org.springframework.security.config.annotation;

public abstract class AbstractConfiguredSecurityBuilder<O, B extends SecurityBuilder<O>>
	extends AbstractSecurityBuilder<O> {

    ...
    private final LinkedHashMap<Class<? extends SecurityConfigurer<O, B>>, List<SecurityConfigurer<O, B>>>
        configurers = new LinkedHashMap<>();
    ...
    
    @Override
	protected final O doBuild() throws Exception {
		synchronized (this.configurers) {
			this.buildState = BuildState.INITIALIZING;
			beforeInit();
			// 循环调用 SecurityConfigurer<Filter, WebSecurity> 的 init() 方法
			init();
			this.buildState = BuildState.CONFIGURING;
			beforeConfigure();
			// 循环调用 SecurityConfigurer<Filter, WebSecurity> 的 configure() 方法
			configure();
			this.buildState = BuildState.BUILDING;
			// 构建 FilterChainProxy，WebSecurity.performBuild()
			O result = performBuild();
			this.buildState = BuildState.BUILT;
			return result;
		}
	}
	...
	
	private void init() throws Exception {
		Collection<SecurityConfigurer<O, B>> configurers = getConfigurers();
		for (SecurityConfigurer<O, B> configurer : configurers) {
		    // 客户端程序员定义的 WebSecurityConfigurerAdapter.init(WebSecurity web) 被调用
			configurer.init((B) this);
		}
		for (SecurityConfigurer<O, B> configurer : this.configurersAddedInInitializing) {
			configurer.init((B) this);
		}
	}
	
	private void configure() throws Exception {
		Collection<SecurityConfigurer<O, B>> configurers = getConfigurers();
		for (SecurityConfigurer<O, B> configurer : configurers) {
		    // 客户端程序员定义的 WebSecurityConfigurerAdapter.configure(WebSecurity web) 被调用
			configurer.configure((B) this);
		}
	}
	
	...
}
~~~

~~~ java
package org.springframework.security.config.annotation.web.configuration;

@Order(100)
public abstract class WebSecurityConfigurerAdapter implements WebSecurityConfigurer<WebSecurity> {
    ...
    
    protected final HttpSecurity getHttp() throws Exception {
		if (this.http != null) {
			return this.http;
		}
		AuthenticationEventPublisher eventPublisher = getAuthenticationEventPublisher();
		this.localConfigureAuthenticationBldr.authenticationEventPublisher(eventPublisher);
		AuthenticationManager authenticationManager = authenticationManager();
		this.authenticationBuilder.parentAuthenticationManager(authenticationManager);
		Map<Class<?>, Object> sharedObjects = createSharedObjects();
		this.http = new HttpSecurity(this.objectPostProcessor, this.authenticationBuilder, sharedObjects);
		if (!this.disableDefaults) {
			applyDefaultConfiguration(this.http);
			ClassLoader classLoader = this.context.getClassLoader();
			List<AbstractHttpConfigurer> defaultHttpConfigurers = SpringFactoriesLoader
					.loadFactories(AbstractHttpConfigurer.class, classLoader);
			for (AbstractHttpConfigurer configurer : defaultHttpConfigurers) {
			   // 将一个 configurer 列表加入到 http 的 configurers 中
			   this.http.apply(configurer);
			}
		}
		// 客户端程序员定义的 WebSecurityConfigurerAdapter.configure(HttpSecurity http) 被调用
		configure(this.http);
		return this.http;
	}
	...
	
	@Override
	public void init(WebSecurity web) throws Exception {
	    // 创建 HttpSecurity（SecurityBuilder<DefaultSecurityFilterChain>）
		HttpSecurity http = getHttp();
		// 把 HttpSecurity 加入 WebSecurity（SecurityBuilder<Filter>）的 securityFilterChainBuilders
		web.addSecurityFilterChainBuilder(http).postBuildAction(() -> {
			FilterSecurityInterceptor securityInterceptor = http.getSharedObject(FilterSecurityInterceptor.class);
			web.securityInterceptor(securityInterceptor);
		});
	}
	...
}
~~~

~~~ java
package org.springframework.security.config.annotation.web.builders;

public final class WebSecurity extends AbstractConfiguredSecurityBuilder<Filter, WebSecurity>
	implements SecurityBuilder<Filter>, ApplicationContextAware {
   ...
   @Override
	protected Filter performBuild() throws Exception {
		Assert.state(!this.securityFilterChainBuilders.isEmpty(),
				() -> "At least one SecurityBuilder<? extends SecurityFilterChain> needs to be specified. "
						+ "Typically this is done by exposing a SecurityFilterChain bean "
						+ "or by adding a @Configuration that extends WebSecurityConfigurerAdapter. "
						+ "More advanced users can invoke " + WebSecurity.class.getSimpleName()
						+ ".addSecurityFilterChainBuilder directly");
		int chainSize = this.ignoredRequests.size() + this.securityFilterChainBuilders.size();
		List<SecurityFilterChain> securityFilterChains = new ArrayList<>(chainSize);
		for (RequestMatcher ignoredRequest : this.ignoredRequests) {
			securityFilterChains.add(new DefaultSecurityFilterChain(ignoredRequest));
		}
		for (SecurityBuilder<? extends SecurityFilterChain> securityFilterChainBuilder : this.securityFilterChainBuilders) {
		    // 构建 SecurityFilterChain，HttpSecurity.build()
			securityFilterChains.add(securityFilterChainBuilder.build());
		}
		// new 了一个 FilterChainProxy 对象
		FilterChainProxy filterChainProxy = new FilterChainProxy(securityFilterChains);
		if (this.httpFirewall != null) {
			filterChainProxy.setFirewall(this.httpFirewall);
		}
		if (this.requestRejectedHandler != null) {
			filterChainProxy.setRequestRejectedHandler(this.requestRejectedHandler);
		}
		filterChainProxy.afterPropertiesSet();

		Filter result = filterChainProxy;
		if (this.debugEnabled) {
			this.logger.warn("\n\n" + "********************************************************************\n"
					+ "**********        Security debugging is enabled.       *************\n"
					+ "**********    This may include sensitive information.  *************\n"
					+ "**********      Do not use in a production system!     *************\n"
					+ "********************************************************************\n\n");
			result = new DebugFilter(filterChainProxy);
		}
		this.postBuildAction.run();
		return result;
	}
   ...
}
~~~

**(3) HttpSecurity 构建 SecurityFilterChain**

HttpSecurity.build()  
和 WebSecurity.build() 一样，HttpSecurity 也是一个 SecurityBuilder.build()  
先是 AbstractSecurityBuilder.build() 保证不重复构建  
然后是 AbstractConfiguredSecurityBuilder.doBuild() 模板方法  
1. 遍历 AbstractConfiguredSecurityBuilder.configurers.init(HttpSecurity)
1. 遍历 AbstractConfiguredSecurityBuilder.configurers.configure(HttpSecurity)
1. HttpSecurity.performBuild() 返回 DefaultSecurityFilterChain 对象

~~~ java
package org.springframework.security.config.annotation.web.builders;

public final class HttpSecurity extends AbstractConfiguredSecurityBuilder<DefaultSecurityFilterChain, HttpSecurity>
	implements SecurityBuilder<DefaultSecurityFilterChain>, HttpSecurityBuilder<HttpSecurity> {
	...
	private List<OrderedFilter> filters = new ArrayList<>();
	
	private RequestMatcher requestMatcher = AnyRequestMatcher.INSTANCE;
	
	private FilterOrderRegistration filterOrders = new FilterOrderRegistration();
	...
	@Override
	protected DefaultSecurityFilterChain performBuild() { // 2618
		this.filters.sort(OrderComparator.INSTANCE);
		List<Filter> sortedFilters = new ArrayList<>(this.filters.size());
		for (Filter filter : this.filters) {
			sortedFilters.add(((OrderedFilter) filter).filter);
		}
		return new DefaultSecurityFilterChain(this.requestMatcher, sortedFilters);
	}
	...
	@Override
	public HttpSecurity addFilter(Filter filter) { // 2660
		Integer order = this.filterOrders.getOrder(filter.getClass());
		if (order == null) {
			throw new IllegalArgumentException("The Filter class " + filter.getClass().getName()
               + " does not have a registered order and cannot be added without a specified order. Consider using addFilterBefore or addFilterAfter instead.");
		}
		this.filters.add(new OrderedFilter(filter, order));
		return this;
	}
	...
~~~

下边以 OAuth2ClientConfigurer 为例，熟悉下配置的一般过程

~~~ java
package org.springframework.security.config.annotation.web.configurers.oauth2.client;

public final class OAuth2ClientConfigurer<B extends HttpSecurityBuilder<B>>
   extends AbstractHttpConfigurer<OAuth2ClientConfigurer<B>, B> {
   
   private AuthorizationCodeGrantConfigurer authorizationCodeGrantConfigurer = new AuthorizationCodeGrantConfigurer();
   ...
   @Override
   public void init(B builder) {
     this.authorizationCodeGrantConfigurer.init(builder);
   }
   
   @Override
   public void configure(B builder) {
     this.authorizationCodeGrantConfigurer.configure(builder);
   }
   
   public final class AuthorizationCodeGrantConfigurer {
      ...
      private void init(B builder) {
         OAuth2AuthorizationCodeAuthenticationProvider
            authorizationCodeAuthenticationProvider = new OAuth2AuthorizationCodeAuthenticationProvider(
               getAccessTokenResponseClient());
         builder.authenticationProvider(postProcess(authorizationCodeAuthenticationProvider));
      }
      
      private void configure(B builder) {
         OAuth2AuthorizationRequestRedirectFilter
            authorizationRequestRedirectFilter = createAuthorizationRequestRedirectFilter(builder);
         // 在 HttpSecurity.filters 中增加一个 OAuth2AuthorizationRequestRedirectFilter
         builder.addFilter(postProcess(authorizationRequestRedirectFilter));
         OAuth2AuthorizationCodeGrantFilter authorizationCodeGrantFilter = createAuthorizationCodeGrantFilter(builder);
         builder.addFilter(postProcess(authorizationCodeGrantFilter));
      }
      ...
   }
   
}
~~~

**(4) 客户端程序员配置**

前文提到 HttpSecurity（SecurityBuilder<DefaultSecurityFilterChain&gt;）持有：configurers  
对应多个：SecurityConfigurer<DefaultSecurityFilterChain, HttpSecurity>>  
其中一个 configurer 就是：OAuth2ClientConfigurer<HttpSecurity>

~~~ java
@Component
public class MySecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.oauth2Client(oAuth2ClientConfigurer -> {});
    }

}
~~~

HttpSecurity.oauth2Client(oAuth2ClientConfigurer -> {})
1. 如果 HttpSecurity 的 configurers 中还没有 OAuth2ClientConfigurer<HttpSecurity> 类的对象  
   则 new 一个并将其加入 http 的 configurers，如果有则取出这个对象
1. 执行 oauth2ClientCustomizer.customize(OAuth2ClientConfigurer<HttpSecurity>)  
   Customizer<OAuth2ClientConfigurer<HttpSecurity>> 就是客户自定义的匿名内部类对象（上边的拉姆达表达式）

~~~ java
package org.springframework.security.config.annotation.web.builders;

public final class HttpSecurity extends AbstractConfiguredSecurityBuilder<DefaultSecurityFilterChain, HttpSecurity>
	implements SecurityBuilder<DefaultSecurityFilterChain>, HttpSecurityBuilder<HttpSecurity> {
    
    ...
	public HttpSecurity oauth2Client(Customizer<OAuth2ClientConfigurer<HttpSecurity>> oauth2ClientCustomizer)
			throws Exception {
		oauth2ClientCustomizer.customize(getOrApply(new OAuth2ClientConfigurer<>()));
		return HttpSecurity.this;
	}
	...
~~~