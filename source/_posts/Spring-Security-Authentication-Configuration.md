---
title: Spring Security（二） 认证配置类
date: 2020-06-26 12:08:25
tags:
    - Spring
    - Java
    - Spring Security
categories: Spring Security
---

&emsp;&emsp;继上一篇 [Spring Security（一） 体系结构之认证](Spring-Security-Architecture-Authentication.html) 后，这里研究一下Spring Security一个重要配置--认证配置。`AuthenticationConfiguration`是Spring提供的认证配置的配置类，该配置类的主要作用是用于生成一个全局的`AuthenticationManagerBuilder`注入到Spring容器中。使用者注入该配置类然后获取AuthenticationManagerBuilder再生成`AuthenticationManager`。

<!-- more -->

&emsp;&emsp;AuthenticationConfiguration中最核心的方法就是`getAuthenticationManager()`，其他的方法都是为这个方法服务的，该方法主要目的就是生成一个全局的认证管理器，但在讲解这个方法之前，需要先介绍一下在这个方法中用到的几个类。

# AuthenticationManagerBuilder

&emsp;&emsp;**这个类采取的是建造者模式，用于创建一个认证管理器(AuthenticationManager)，能够轻松的建造出一个基于内存的认证，LDAP认证或者是基于jdbc的认证，添加`UserDetailsService`以及添加`AuthenticationProvider`** 。

```java
@Bean
public AuthenticationManagerBuilder authenticationManagerBuilder(
                ObjectPostProcessor<Object> objectPostProcessor, ApplicationContext context) {
    LazyPasswordEncoder defaultPasswordEncoder = new LazyPasswordEncoder(context);
    AuthenticationEventPublisher authenticationEventPublisher = getBeanOrNull(context, AuthenticationEventPublisher.class);

    DefaultPasswordEncoderAuthenticationManagerBuilder result = new DefaultPasswordEncoderAuthenticationManagerBuilder(objectPostProcessor, defaultPasswordEncoder);
    if (authenticationEventPublisher != null) {
        result.authenticationEventPublisher(authenticationEventPublisher);
    }
    return result;
}
```

&emsp;&emsp;以上代码片段是截取自`AuthenticationConfiguration`类。`ObjectPostProcessor<Object> objectPostProcessor`是后置处理器，他的一个重要实现是`AutowireBeanFactoryObjectPostProcessor`，作用是生成Spring bean。`ApplicationContext context`是应用上下文。

&emsp;&emsp;这里初始化了一个解码器，解码器的类型是该配置类中的内部类`LazyPasswordEncoder`,这个内部类采用了委托模式，它从Spring 上下文中寻找解码器，如果找到了就委托给这个解码器，如果没找到就调用`PasswordEncoderFactories.createDelegatingPasswordEncoder();`生成解码器。而这个方法生成的是`DelegatingPasswordEncoder`的实例，该类的设计采用的是也是委托模式，他实际委派的对象是聚合了多个不同的解码器。

&emsp;&emsp;接下来从上下文中获取认证事件发布器，如果找到了就将该发布器的引用赋值给稍后生成的`AuthenticationManagerBuilder`。

&emsp;&emsp;注入到Spring上下文中的AuthenticationManagerBuilder的实例类型是`DefaultPasswordEncoderAuthenticationManagerBuilder`，这是认证配置类中的一个内部类，其内部方法实现都是调用父类的方法的，只不过实现类添加了一个解码器的属性，通过构造器将传入一个解码器，在进行配置的时候将解码器传递给对应的认证配置对象。

&emsp;&emsp;需要注意的是，使用AuthenticationManagerBuilder.build()构造出来的AuthenticationManager是`ProviderManager`的实例。看一下它的继承关系。

![继承关系](/Spring-Security-Authentication-Configuration/AuthenticationManagerBuilder-extends.png)

&emsp;&emsp;通过继承关系可以看出，AuthenticationManagerBuilder和它的父类AbstractConfiguredecurityBuilder都没有实现build方法，所以调用的是AbstractSecurityBuilder方法。

```java
public final O build() throws Exception {
    if (this.building.compareAndSet(false, true)) {
        this.object = doBuild();
        return this.object;
    }
    throw new AlreadyBuiltException("This object has already been built");
}
```

&emsp;&emsp;调用了`doBuild()`方法并返回其结果，AuthenticationManagerBuilder没有实现此方法，而父类AbstractConfiguredecurityBuilder实现了。

````java
protected final O doBuild() throws Exception {
    synchronized (configurers) {
        buildState = BuildState.INITIALIZING;
        
        beforeInit();
        init();
        
        buildState = BuildState.CONFIGURING;
        
        beforeConfigure();
        configure();
        
        buildState = BuildState.BUILDING;
        
        O result = performBuild();
        
        buildState = BuildState.BUILT;
        
        return result;
    }
}
````

&emsp;&emsp;可以看到，最终返回的是`performBuild()`方法的返回值，而AuthenticationManagerBuilder实现了此方法。

```java
protected ProviderManager performBuild() throws Exception {
    if (!isConfigured()) {
        logger.debug("No authenticationProviders and no parentAuthenticationManager defined. Returning null.");
        return null;
    }
    ProviderManager providerManager = new ProviderManager(authenticationProviders, parentAuthenticationManager);
    if (eraseCredentials != null) {
        providerManager.setEraseCredentialsAfterAuthentication(eraseCredentials);
    }
    if (eventPublisher != null) {
        providerManager.setAuthenticationEventPublisher(eventPublisher);
    }
    providerManager = postProcess(providerManager);
    return providerManager;
}
```

&emsp;&emsp;显然，使用AuthenticationManagerBuilder的build方法构造AuthenticationManager其实例的类型就是ProviderManager。

# AuthenticationManagerDelegator

&emsp;&emsp;这个类是配置类中的一个内部类，他的作用是在获取认证管理器的时候防止出现无限递归（具体在什么样的情况下会出现无限递归我还没搞明白，源码注释是这么说的），它是采取了委托模式，将实际的认证任务委托给了前面我们初始化出来的AuthenticationManagerBuilder生成的认证管理器。

```java
static final class AuthenticationManagerDelegator implements AuthenticationManager {
    private AuthenticationManagerBuilder delegateBuilder;
    private AuthenticationManager delegate;
    private final Object delegateMonitor = new Object();
    AuthenticationManagerDelegator(AuthenticationManagerBuilder delegateBuilder) {
        Assert.notNull(delegateBuilder, "delegateBuilder cannot be null");
        this.delegateBuilder = delegateBuilder;
    }
    @Override
    public Authentication authenticate(Authentication authentication)
                throws AuthenticationException {
        if (this.delegate != null) { 
            return this.delegate.authenticate(authentication);
        }
        synchronized (this.delegateMonitor) {
            if (this.delegate == null) {
                this.delegate = this.delegateBuilder.getObject();
                this.delegateBuilder = null;
            }
        }
        return this.delegate.authenticate(authentication);
    }
    @Override
    public String toString() {
        return "AuthenticationManagerDelegator [delegate=" + this.delegate + "]";
    }
}
```

&emsp;&emsp;通过源码可以看到，在调用认证方法时，判断委托对象delegate是否为空，如果为空那么就构造一个认证管理器赋值给委托对象并且使用委托对象完成认证。

# GlobalAuthenticationConfigurerAdapter

&emsp;&emsp;这个一个安全配置抽象类，里面对实现的接口`SecurityConfigurer`中的方法进行了空实现。在安全配置类AuthenticationConfiguration中，有多个它的子类被注入到Spring上下文中，随后被应用在了上下文中的AuthenticationManagerBuilder中。接下来我们抽取AuthenticationConfiguration中的部分代码展示。

```java
private List<GlobalAuthenticationConfigurerAdapter> globalAuthConfigurers = Collections.emptyList();

@Bean
public static GlobalAuthenticationConfigurerAdapter enableGlobalAuthenticationAutowiredConfigurer(
        ApplicationContext context) {
    return new EnableGlobalAuthenticationAutowiredConfigurer(context);
}

@Bean
public static InitializeUserDetailsBeanManagerConfigurer initializeUserDetailsBeanManagerConfigurer(ApplicationContext context) {
    return new InitializeUserDetailsBeanManagerConfigurer(context);
}

@Bean
public static InitializeAuthenticationProviderBeanManagerConfigurer initializeAuthenticationProviderBeanManagerConfigurer(ApplicationContext context) {
    return new InitializeAuthenticationProviderBeanManagerConfigurer(context);
}

@Autowired(required = false)
public void setGlobalAuthenticationConfigurers(List<GlobalAuthenticationConfigurerAdapter> configurers) throws Exception {
    Collections.sort(configurers, AnnotationAwareOrderComparator.INSTANCE);
    this.globalAuthConfigurers = configurers;
}
```

&emsp;&emsp;这里面涉及三个子类：

> `EnableGlobalAuthenticationAutowiredConfigurer`：看源码就是在init方法中获取使用了`EnableGlobalAuthentication`注解的bean，然后通过debug日志打印，再就没有其他操作了。
> `InitializeUserDetailsBeanManagerConfigurer`：初始化一个认证数据服务配置对象应用到全局的 AuthenticationManagerBuilder 中，其主要作用就是初始化一个认证实现添加到Spring上下文的 AuthenticationManagerBuilder bean中，其类型为`DaoAuthenticationProvider`，获取Spring上下文中的`UserDetailsService`，`PasswordEncoder`和`UserDetailsPasswordService`，将其放入认证对象中。默认的表单登陆认证就是在这里配置的。这里加载的`UserDetailsService`bean是在`UserDetailsServiceAutoConfiguration`配置类中初始化的，初始化了一个基于内存的数据服务对象，Spring Security的默认的UUID密码就是在这里设置进去的，是从`SecurityProperties.User`类中获取。
> `InitializeAuthenticationProviderBeanManagerConfigurer`：初始化一个认证管理配置对象，从Spring上下文中获取 AuthenticationProvider 并加载到 AuthenticationManagerBuilder 中。

&emsp;&emsp;说到这里，认证的全局配置基本说完了，这里总结一下，在`AuthenticationConfiguration`中，初始化了三个认证配置对象加载到Spring上下文中并且注入到自身对象中，其次初始化了`AuthenticationManagerBuilder`并且将之前的三个配置类应用到此对象中，然后将对象加载到Spring上下文中，这是一个全局的构造器，用来生成一个全局的认证配置器。此外，该类还提供了一个获取全局认证器的方法。
