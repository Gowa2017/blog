---
title: 关于SprintMVC配置文件的加载及初始化
categories:
  - Java
date: 2019-06-15 07:52:51
updated: 2019-06-15 07:52:51
tags: 
  - Java
  - Spring
  - SpringMVC
---
我们知道，SpringMVC 是以 Servlet 的形式由 Tomcat 中的容器根据 web app 应用的 web.xml(部署描述符) 配置来启动的。但是，我们写在 xml 文件的配置，是什么时候，用什么样的方式加载和初始化的呢？

<!--more-->

# Tomcat 容器模型

要先了解更多，首先得了解一下 Tomcat 的容器模型是什么样的。

![](https://www.ibm.com/developerworks/cn/java/j-lo-servlet/image002.jpg)

可以和我们在 config/server.xml 中的配置对应起来。

对于我们的 web 应用来说。每个 web 应用对应一个 Context 对象。

然后 Tomcat 会将 web.xml 解析到一个 WebXml 对象， Context 会持有这个对象。同时还会创建 *web.xml* 内定义的 filter, listener, servlet。

然后就会初始化我们的 Servlet 了。


# web.xml 中的配置

```xml
	<context-param>
		<param-name>contextConfigLocation</param-name>
		<param-value>
			classpath:spring/ApplicationContext-main.xml,
			classpath:spring/ApplicationContext-dataSource.xml,
			classpath:spring/ApplicationContext-shiro.xml
		</param-value>
	</context-param>
	<listener>
		<listener-class>org.springframework.web.util.Log4jConfigListener</listener-class>
	</listener>
	<servlet>
		<servlet-name>springMvc</servlet-name>
		<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
		<init-param>
			<param-name>contextConfigLocation</param-name>
			<param-value>classpath:spring/ApplicationContext-mvc.xml</param-value>
		</init-param>
		<load-on-startup>1</load-on-startup>
	</servlet>
	<servlet-mapping>
		<servlet-name>springMvc</servlet-name>
		<url-pattern>/</url-pattern>
	</servlet-mapping>
```

其中 ：

- init-param 全局变量
- listener 这里 ContextLoaderListener 实现了 ServletContextListener，主要是用来监听 Servlet 的初始化和销毁事件。
- servlet 要启动的 Servlet

# Servlet 的实例化

我们知道， Servlet 是由 Tomcat 来进行实例化的。对于  ServletContextListener 来说，当我们初始化和销毁 Servlet 的时候就会收到通知，被调用。

其中事件 ServletContextListener 的 ServletContext 也是由 Tomcat 来负责的。

对于我们定义的 ContextLoaderListener：

```java
	public void contextInitialized(ServletContextEvent event) {
		initWebApplicationContext(event.getServletContext());
	}
```

```java
	public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
		if (servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null) {
			throw new IllegalStateException(
					"Cannot initialize context because there is already a root application context present - " +
					"check whether you have multiple ContextLoader* definitions in your web.xml!");
		}

		Log logger = LogFactory.getLog(ContextLoader.class);
		servletContext.log("Initializing Spring root WebApplicationContext");
		if (logger.isInfoEnabled()) {
			logger.info("Root WebApplicationContext: initialization started");
		}
		long startTime = System.currentTimeMillis();

		try {
			// Store context in local instance variable, to guarantee that
			// it is available on ServletContext shutdown.
			if (this.context == null) {
				this.context = createWebApplicationContext(servletContext);
			}
			if (this.context instanceof ConfigurableWebApplicationContext) {
				ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
				if (!cwac.isActive()) {
					// The context has not yet been refreshed -> provide services such as
					// setting the parent context, setting the application context id, etc
					if (cwac.getParent() == null) {
						// The context instance was injected without an explicit parent ->
						// determine parent for root web application context, if any.
						ApplicationContext parent = loadParentContext(servletContext);
						cwac.setParent(parent);
					}
					configureAndRefreshWebApplicationContext(cwac, servletContext);
				}
			}
			servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);

			ClassLoader ccl = Thread.currentThread().getContextClassLoader();
			if (ccl == ContextLoader.class.getClassLoader()) {
				currentContext = this.context;
			}
			else if (ccl != null) {
				currentContextPerThread.put(ccl, this.context);
			}

			if (logger.isDebugEnabled()) {
				logger.debug("Published root WebApplicationContext as ServletContext attribute with name [" +
						WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE + "]");
			}
			if (logger.isInfoEnabled()) {
				long elapsedTime = System.currentTimeMillis() - startTime;
				logger.info("Root WebApplicationContext: initialization completed in " + elapsedTime + " ms");
			}

			return this.context;
		}
		catch (RuntimeException ex) {
			logger.error("Context initialization failed", ex);
			servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, ex);
			throw ex;
		}
		catch (Error err) {
			logger.error("Context initialization failed", err);
			servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, err);
			throw err;
		}
	}
```

在 Servlet 初始化的时候，就会建立一个 Spring 的 应用上下文。同时将其作为 ServletContext 的一个属性挂起来：

```java			servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);
```

## configureAndRefreshWebApplicationContext()

上下建立后，就会解析配置文件，也即是我们定义的 *contextConfigLocation*。

```java
	protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc) {
		if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
			// The application context id is still set to its original default value
			// -> assign a more useful id based on available information
			String idParam = sc.getInitParameter(CONTEXT_ID_PARAM);
			if (idParam != null) {
				wac.setId(idParam);
			}
			else {
				// Generate default id...
				wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
						ObjectUtils.getDisplayString(sc.getContextPath()));
			}
		}

		wac.setServletContext(sc);
		String configLocationParam = sc.getInitParameter(CONFIG_LOCATION_PARAM);
		if (configLocationParam != null) {
			wac.setConfigLocation(configLocationParam);
		}

		// The wac environment's #initPropertySources will be called in any case when the context
		// is refreshed; do it eagerly here to ensure servlet property sources are in place for
		// use in any post-processing or initialization that occurs below prior to #refresh
		ConfigurableEnvironment env = wac.getEnvironment();
		if (env instanceof ConfigurableWebEnvironment) {
			((ConfigurableWebEnvironment) env).initPropertySources(sc, null);
		}

		customizeContext(sc, wac);
		wac.refresh();
	}
```

默认情况下，我们的 WebApplicationContext 的实现是：XmlWebApplicationContext。会使用我们定义的 contextConfigLocation 参数，然后更新 WebApplicationContext。

## AbstractApplicationContext.refresh()

这个类继承得比较远，最终调用的是 AbstractApplicationContext.refresh。

```java
    public void refresh() throws BeansException, IllegalStateException {
        synchronized(this.startupShutdownMonitor) {
            this.prepareRefresh();
            ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();
            this.prepareBeanFactory(beanFactory);

            try {
                this.postProcessBeanFactory(beanFactory);
                this.invokeBeanFactoryPostProcessors(beanFactory);
                this.registerBeanPostProcessors(beanFactory);
                this.initMessageSource();
                this.initApplicationEventMulticaster();
                this.onRefresh();
                this.registerListeners();
                this.finishBeanFactoryInitialization(beanFactory);
                this.finishRefresh();
            } catch (BeansException var9) {
                if (this.logger.isWarnEnabled()) {
                    this.logger.warn("Exception encountered during context initialization - cancelling refresh attempt: " + var9);
                }

                this.destroyBeans();
                this.cancelRefresh(var9);
                throw var9;
            } finally {
                this.resetCommonCaches();
            }

        }
    }
```

这个方法中干的活是比较多的了。重要的是在 obtainFreshBeanFactory 内开始进行解析 bean。

## AbstractApplicationContext

```java
	protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
		refreshBeanFactory();
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		if (logger.isDebugEnabled()) {
			logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
		}
		return beanFactory;
	}

```

## AbstractRefreshableApplicationContext

```java

	@Override
	protected final void refreshBeanFactory() throws BeansException {
		if (hasBeanFactory()) {
			destroyBeans();
			closeBeanFactory();
		}
		try {
			DefaultListableBeanFactory beanFactory = createBeanFactory();
			beanFactory.setSerializationId(getId());
			customizeBeanFactory(beanFactory);
			loadBeanDefinitions(beanFactory);
			synchronized (this.beanFactoryMonitor) {
				this.beanFactory = beanFactory;
			}
		}
		catch (IOException ex) {
			throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
		}
	}
	
	@Override
	public final ConfigurableListableBeanFactory getBeanFactory() {
		synchronized (this.beanFactoryMonitor) {
			if (this.beanFactory == null) {
				throw new IllegalStateException("BeanFactory not initialized or already closed - " +
						"call 'refresh' before accessing beans via the ApplicationContext");
			}
			return this.beanFactory;
		}
	}
```

## XmlWebApplicationContext

```java
	@Override
	protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
		// Create a new XmlBeanDefinitionReader for the given BeanFactory.
		XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

		// Configure the bean definition reader with this context's
		// resource loading environment.
		beanDefinitionReader.setEnvironment(getEnvironment());
		beanDefinitionReader.setResourceLoader(this);
		beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

		// Allow a subclass to provide custom initialization of the reader,
		// then proceed with actually loading the bean definitions.
		initBeanDefinitionReader(beanDefinitionReader);
		loadBeanDefinitions(beanDefinitionReader);
	}
	
	
	protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws IOException {
		String[] configLocations = getConfigLocations();
		if (configLocations != null) {
			for (String configLocation : configLocations) {
				reader.loadBeanDefinitions(configLocation);
			}
		}
	}
```

loadBeanDefinitions() 就会根据我们定义的路径去寻找 bean 的 xml 文件，进行解析了。

## AbstractApplicationContextfinishBeanFactoryInitialization(beanFactory);

初始化 bean。

```java
    public void preInstantiateSingletons() throws BeansException {
        if (this.logger.isDebugEnabled()) {
            this.logger.debug("Pre-instantiating singletons in " + this);
        }

        List<String> beanNames = new ArrayList(this.beanDefinitionNames);
        Iterator var2 = beanNames.iterator();

        while(true) {
            while(true) {
                String beanName;
                RootBeanDefinition bd;
                do {
                    do {
                        do {
                            if (!var2.hasNext()) {
                                var2 = beanNames.iterator();

                                while(var2.hasNext()) {
                                    beanName = (String)var2.next();
                                    Object singletonInstance = this.getSingleton(beanName);
                                    if (singletonInstance instanceof SmartInitializingSingleton) {
                                        final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton)singletonInstance;
                                        if (System.getSecurityManager() != null) {
                                            AccessController.doPrivileged(new PrivilegedAction<Object>() {
                                                public Object run() {
                                                    smartSingleton.afterSingletonsInstantiated();
                                                    return null;
                                                }
                                            }, this.getAccessControlContext());
                                        } else {
                                            smartSingleton.afterSingletonsInstantiated();
                                        }
                                    }
                                }

                                return;
                            }

                            beanName = (String)var2.next();
                            bd = this.getMergedLocalBeanDefinition(beanName);
                        } while(bd.isAbstract());
                    } while(!bd.isSingleton());
                } while(bd.isLazyInit());

                if (this.isFactoryBean(beanName)) {
                    final FactoryBean<?> factory = (FactoryBean)this.getBean("&" + beanName);
                    boolean isEagerInit;
                    if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
                        isEagerInit = (Boolean)AccessController.doPrivileged(new PrivilegedAction<Boolean>() {
                            public Boolean run() {
                                return ((SmartFactoryBean)factory).isEagerInit();
                            }
                        }, this.getAccessControlContext());
                    } else {
                        isEagerInit = factory instanceof SmartFactoryBean && ((SmartFactoryBean)factory).isEagerInit();
                    }

                    if (isEagerInit) {
                        this.getBean(beanName);
                    }
                } else {
                    this.getBean(beanName);
                }
            }
        }
    }
```
# DispatchServlet 继承关系

```java
DispatcherServlet <-
    FrameworkServlet <-
        HttpServletBean <-
            HttpServlet <-
                GenericServlet
```

我们知道 HttpServlet 也是规范的一部分，所以说，我们要关注的内容是从 HttpServletBean 开始的。

## HttpServletBean.init()

这个类是 HttpServlet 的直接继承，其主要的作用是：

将 *web.xml* 内 *servlet* 标签当作 bean properties 对待。

对于一个 Servlet 来说，当其实例化后，容器（Tomcat）就会立刻调用其 init(ServletConfig) 方法。

对于一个 Servlet 来说，其  ServletConfig 即是 *web.xml* 内的配置。

```java
    // GenericServlet
    public void init(ServletConfig config) throws ServletException {
	this.config = config;
	this.init();
    }
    
    // HttpServletBean
	@Override
	public final void init() throws ServletException {
		if (logger.isDebugEnabled()) {
			logger.debug("Initializing servlet '" + getServletName() + "'");
		}

		// Set bean properties from init parameters.
		// 这段代码，即是将 web.xml 内配置的所有 context-param 进行处理
		try {
			PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
			BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
			ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
			bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
			initBeanWrapper(bw);
			bw.setPropertyValues(pvs, true);
		}
		catch (BeansException ex) {
			logger.error("Failed to set bean properties on servlet '" + getServletName() + "'", ex);
			throw ex;
		}

		// Let subclasses do whatever initialization they like.
		initServletBean();

		if (logger.isDebugEnabled()) {
			logger.debug("Servlet '" + getServletName() + "' configured successfully");
		}
	}
```

## FrameworkServlet.initServletBean()

这就是 Spring 的 web 框架的 Servlet。其以一个基于 JavaBean 的形式与一个 Spring 应用上下文相集成。我们之前初始化的 wac 在此处被使用。

```java
	@Override
	protected final void initServletBean() throws ServletException {
		getServletContext().log("Initializing Spring FrameworkServlet '" + getServletName() + "'");
		if (this.logger.isInfoEnabled()) {
			this.logger.info("FrameworkServlet '" + getServletName() + "': initialization started");
		}
		long startTime = System.currentTimeMillis();

		try {
			this.webApplicationContext = initWebApplicationContext();
			initFrameworkServlet();
		}
		catch (ServletException ex) {
			this.logger.error("Context initialization failed", ex);
			throw ex;
		}
		catch (RuntimeException ex) {
			this.logger.error("Context initialization failed", ex);
			throw ex;
		}

		if (this.logger.isInfoEnabled()) {
			long elapsedTime = System.currentTimeMillis() - startTime;
			this.logger.info("FrameworkServlet '" + getServletName() + "': initialization completed in " +
					elapsedTime + " ms");
		}
	}
```

# Servlet 接口和请求响应对象

对于 web 应用来说，通过继承 *javax.servlet.http.HttpServlet* 来实现 Servlet。

- init(...): Initialize the servlet.
- destroy(...): Terminate the servlet.
- doGet(...): Execute an HTTP GET request.
- doPost(...): Execute an HTTP POST request.
- doPut(...): Execute an HTTP PUT request.
- doDelete(...): Execute an HTTP DELETE request.
- service(...): Receive HTTP requests and, by default, dispatch them to the appropriate doXXX() methods.
- getServletInfo(...): Retrieve information about the servlet.


## Servlet Context 基础

一个 Servlet Context(上下文) 是一个实现了 *javax.servlet.ServletContext* 接口的实例。

一个 Servlet 上下文对象提供了 Servlet 的运行环境信息（如服务器名称），允许单一 JVM (支持多 JVM 的容器需要实现资源复用变量) 同一组内的 Servlets 间进行资源共享。

一个 Servlet 上下文提供了应用运行实例的范围信息。通过这个算法，每个 web 应用都用特定的 classloader 来加载同时其运行时对象和其他 web 应用都是不同的。

单一 Host 内可拥有多个 Servlet 上下文对象。

### 获取一个上下文

可以使用一个**上下文配置对象**的 `getServletContext()` 方法来获取一个 Servlet 上下文。


## Servlet 配置对象

一个 **servlet configuration objec** 包含了一个 Servlet 初始化和启动参数，其是 *javax.servlet.ServletConfig* 的一个实例。

可以通过 Servlet 的 `getServletConfig()` 来获取其配置对象。此方法在 *javax.servlet.Servlet* 定义，默认实现在 *javax.servlet.http.HttpServlet* 中。

- `ServletContext getServletContext()` 获取 Servlet 的上下文
- `String getServletName()` 获取 Servlet 名称
- `Enumeration getInitParameterNames()` 获取 Servlet 的初始化参数。
- `String getInitParameter(String name)` 获取某一初始化参数的值。
