---
title: Spring源码分析 - PostProcessor们
categories:
  - Spring
tags:
  - Spring
date: 2021-11-21 09:42:23
---

`BeanPostProcessor`是Spring中参与Bean生命周期定制非常重要的一个手段，上文中分析过，其执行有两个时机

- 一前：Bean自动注入之后，自定义初始化方法调用前
- 一后：自定义方法调用之后

Spring中很多重要的特性利用了`BeanPostProcessor`达成，毕竟，算来算去，Spring中整个Bean的生命周期已经足够复杂了，如果每加一个功能就要在生命周期上做文章，只会增加复杂度，而`BeanPostProcessor`则是Spring提供的一种扩展方式。与其相对的，一般用户用的可能较少的`BeanFactoryPostProcessor`是针对整个容器初始化完成后提供定制化功能的扩展，我们也要观察一下。观察的主要内容是主要实现类及其作用。

## BeanFactoryPostProcessor

它的调用，在`org.springframework.context.support.AbstractApplicationContext#refresh.564行`。

其接口及其简单：一个简单的函数式接口。

```java
@FunctionalInterface
public interface BeanFactoryPostProcessor {
	void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
}
```

这意味着，在容器创建后，我们能够向其中设置任何内容，也可以利用容器刚刚创建这个时机，来做一些时机上再容器全局的一些事情。

![image-20211122183440854](https://gdz.oss-cn-shenzhen.aliyuncs.com/local/image-20211122183440854.png)

具体来说，有这么几类

### 修改现有容器配置

- `AbstractDependsOnBeanFactoryPostProcessor`、`CacheManagerEntityManagerFactoryDependsOnPostProcessor`。强制为某些Bean显式设置依赖关系，使得不满足依赖时容器无法启动。这在自动配时会有用
- `CustomScopeConfigurer`，添加自定义scope，这在`WebSocketMessageBrokerConfigurationSupport`有使用

- `CustomEditorConfigurer`，注册一些自定义的`PropertyEditor`
- `LazyInitializationBeanFactoryPostProcessor`，它将容器中没有指定延迟加载属性的bean定义，除以下条件的bean设置为延迟加载
  - `SmartInitializingSingleton`类型
  - `AopInfrastructureBean`类型
  - `TaskScheduler`类型
  - `ScheduledExecutorService`类型
  - 被`@Scheduled`或`Schedules`注解的类

### 向容器中添加Bean

- `ServletComponentRegisteringPostProcessor`，它创建了一个Servlet环境，注册了必要的Bean
- `EventListenerMethodProcessor`，它配合`SmartInitializingSingleton`，实现了`@EventListener`注解
  - 在容器初始化完成后，获取并持有了容器的`EventListenerFactory`、容器本身
  - 在单例Bean初始化完成后，检测带有`@Component`的Bean内部是否有`@EventListener`注解的方法，如果有，则使用上一步持有的`EventListenerFactory`将该方法创建为一个`ApplicationListener`实例，然后注入容器

### 一些全局开关

- `AspectJWeavingEnabler`，开启AspectJ。

## BeanPostProcessor

它的执行时机，有两个：Bean创建后，实例化前；Bean实例化后。

```java
public interface BeanPostProcessor {

  default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
    return bean;
  }

  default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
    return bean;
  }

}
```

### AOP相关

- `AbstractAdvisingBeanPostProcessor`
- `AdvisorAdapterRegistrationManager`

### xxxAware的实现

- `ApplicationContextAwareProcessor`，在Bean初始化之前，调用了如下接口的setxxx方法
  - `EnvironmentAware`
  - `EmbeddedValueResolverAware`
  - `ResourceLoaderAware`
  - `ApplicationEventPublisherAware`
  - `MessageSourceAware`
  - `ApplicationContextAware`
  - `ApplicationStartupAware`
- `BootstrapContextAwareProcessor`，在Bean初始化之前，调用了如下接口的setxxx方法
  - `BootstrapContextAware`
- `WebApplicationContextServletContextAwareProcessor`，在Bean初始化之前，调用了如下接口的setxxx方法
  - `ServletContextAware`
  - `ServletConfigAware`

### 属性配置类

`ConfigurationPropertiesBindingPostProcessor`，它将环境中的属性绑定到`@ConfigurationProperties`注解到的类上。比如

```kotlin
@ConfigurationProperties(prefix = "aliyun")
class AliyunConfig {
  lateinit var accessKey: String
  lateinit var secretKey: String
}
```

它能够将配置中的如下属性注入对象

```properties
aliyun.access-key=xxxx
aliyun.secret-key=xxx
```

来看看该类的源码

```java
public class ConfigurationPropertiesBindingPostProcessor implements BeanPostProcessor, PriorityOrdered, ApplicationContextAware, InitializingBean {

  private ApplicationContext applicationContext;

  private BeanDefinitionRegistry registry;

  private ConfigurationPropertiesBinder binder;

  @Override
  public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
    this.applicationContext = applicationContext;
  }

  @Override
  public void afterPropertiesSet() throws Exception {
    this.registry = (BeanDefinitionRegistry) this.applicationContext.getAutowireCapableBeanFactory();
    // 创建ConfigurationPropertiesBinder，这是关键点1
    this.binder = ConfigurationPropertiesBinder.get(this.applicationContext);
  }

  @Override
  public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
    // 关键点2：构建ConfigurationPropertiesBean
    bind(ConfigurationPropertiesBean.get(this.applicationContext, bean, beanName));
    return bean;
  }

  private void bind(ConfigurationPropertiesBean bean) {
    if (bean == null || hasBoundValueObject(bean.getName())) {
      return;
    }
    Assert.state(bean.getBindMethod() == BindMethod.JAVA_BEAN, "错误信息");
    // 关键点3：调用binder.bind方法，完成绑定
    this.binder.bind(bean);
  }

  // 这个先不管
  private boolean hasBoundValueObject(String beanName) {
    return this.registry.containsBeanDefinition(beanName) && 
      this.registry.getBeanDefinition(beanName) instanceof ConfigurationPropertiesValueObjectBeanDefinition;
  }

}
```

理解它的关键，在于理解Spring的绑定机制，该机制有点复杂，不是一两句能说清的。简单来说，就是将一堆属性绑定到指定的领域模型。我们只简单地看一下上面涉及到的两个类。具体的，下一篇文章再来看。

#### ConfigurationPropertiesBean

```java
public final class ConfigurationPropertiesBean {

  // 名字，直接就是传入bean的名字
  private final String name;
	// 传入bean的实例
  private final Object instance;
	// 注解在bean实例上的ConfigurationProperties注解实例
  private final ConfigurationProperties annotation;
	// 绑定目标，由传入的bean实例+其它的注解构成
  private final Bindable<?> bindTarget;
	// 绑定方法，枚举，JAVA_BEAN：java bean，使用getter和setter绑定；VALUE_OBJECT：值对象，使用构造方法绑定
  private final BindMethod bindMethod;

  private ConfigurationPropertiesBean(String name, Object instance, ConfigurationProperties annotation, Bindable<?> bindTarget) {
    this.name = name;
    this.instance = instance;
    this.annotation = annotation;
    this.bindTarget = bindTarget;
    this.bindMethod = BindMethod.forType(bindTarget.getType().resolve());
  }

  public static ConfigurationPropertiesBean get(ApplicationContext applicationContext, Object bean, String beanName) {
    // 获取bean的工厂方法，就是我们创建时指定的工厂方法，没有就为null
    Method factoryMethod = findFactoryMethod(applicationContext, beanName);
    return create(beanName, bean, bean.getClass(), factoryMethod);
  }

  private static ConfigurationPropertiesBean create(String name, Object instance, Class<?> type, Method factory) {
    // 获取ConfigurationProperties注解
    ConfigurationProperties annotation = findAnnotation(instance, type, factory, ConfigurationProperties.class);
    if (annotation == null) {
      return null;
    }
    // 获取Validated注解
    Validated validated = findAnnotation(instance, type, factory, Validated.class);
    Annotation[] annotations = (validated != null) ? new Annotation[] { annotation, validated }
    : new Annotation[] { annotation };
    // 解析待绑定类型：工厂方法的返回类型，或者，传入type所指定的类型
    ResolvableType bindType = (factory != null) ? ResolvableType.forMethodReturnType(factory) : ResolvableType.forClass(type);
    Bindable<Object> bindTarget = Bindable.of(bindType).withAnnotations(annotations);
    if (instance != null) {
      bindTarget = bindTarget.withExistingValue(instance);
    }
    return new ConfigurationPropertiesBean(name, instance, annotation, bindTarget);
  }
}
```

- 对`Bindable`和`BindMethod`，可能有一些陌生，暂且不管，后面专门写文章介绍它
- `ConfigurationPropertiesBean`中包含的内容：目标bean实例、`ConfigurationProperties`注解、绑定目标、绑定方式
- 该类为绑定做准备

#### ConfigurationPropertiesBinder

```java
class ConfigurationPropertiesBinder {

  private static final String BEAN_NAME = "org.springframework.boot.context.internalConfigurationPropertiesBinder";
  
  private static final String VALIDATOR_BEAN_NAME = EnableConfigurationProperties.VALIDATOR_BEAN_NAME;

  // 构造方法，初始化了四个属性
  ConfigurationPropertiesBinder(ApplicationContext applicationContext) {
		this.applicationContext = applicationContext;
    // 获取容器的属性源
		this.propertySources = new PropertySourcesDeducer(applicationContext).getPropertySources();
    // 从容器中获取EnableConfigurationProperties.VALIDATOR_BEAN_NAME指定的验证器
		this.configurationPropertiesValidator = getConfigurationPropertiesValidator(applicationContext);
    // 判定是否要遵循jsr303验证规范："javax.validation.Validator", "javax.validation.ValidatorFactory", "javax.validation.bootstrap.GenericBootstrap" 这三个类存在，就需要遵循
		this.jsr303Present = ConfigurationPropertiesJsr303Validator.isJsr303Present(applicationContext);
	}
  
  // 从容器中获取提前初始化OK的binder
  static ConfigurationPropertiesBinder get(BeanFactory beanFactory) {
    return beanFactory.getBean(BEAN_NAME, ConfigurationPropertiesBinder.class);
  }

  // 那个绑定方法
  BindResult<?> bind(ConfigurationPropertiesBean propertiesBean) {
    // 绑定目标
    Bindable<?> target = propertiesBean.asBindTarget();
    // 注解
    ConfigurationProperties annotation = propertiesBean.getAnnotation();
    // 获取处理器
    BindHandler bindHandler = getBindHandler(target, annotation);
    // 调用Binder.bind方法，完成绑定
    return getBinder().bind(annotation.prefix(), target, bindHandler);
  }

  // 获取绑定处理器
  private <T> BindHandler getBindHandler(Bindable<T> target, ConfigurationProperties annotation) {
    // 获取验证器：configurationPropertiesValidator、ConfigurationPropertiesJsr303Validator、自定义验证器
    List<Validator> validators = getValidators(target);
    // 获取处理器：IgnoreTopLevelConverterNotFoundBindHandler
    BindHandler handler = getHandler();
    // 根据不同条件构建不同BindHandler
    handler = new ConfigurationPropertiesBindHander(handler);
    if (annotation.ignoreInvalidFields()) {
      handler = new IgnoreErrorsBindHandler(handler);
    }
    if (!annotation.ignoreUnknownFields()) {
      UnboundElementsSourceFilter filter = new UnboundElementsSourceFilter();
      handler = new NoUnboundElementsBindHandler(handler, filter);
    }
    if (!validators.isEmpty()) {
      handler = new ValidationBindHandler(handler, validators.toArray(new Validator[0]));
    }
    // 一些额外的处理器
    for (ConfigurationPropertiesBindHandlerAdvisor advisor : getBindHandlerAdvisors()) {
      handler = advisor.apply(handler);
    }
    return handler;
  }
  
  // 获取有效的验证器
  private List<Validator> getValidators(Bindable<?> target) {
		List<Validator> validators = new ArrayList<>(3);
		if (this.configurationPropertiesValidator != null) {
      // 容器中的configurationPropertiesValidator
			validators.add(this.configurationPropertiesValidator);
		}
		if (this.jsr303Present && target.getAnnotation(Validated.class) != null) {
      // 新建的ConfigurationPropertiesJsr303Validator
			validators.add(getJsr303Validator());
		}
		if (target.getValue() != null && target.getValue().get() instanceof Validator) {
      // 绑定目标本身也可以是验证器
			validators.add((Validator) target.getValue().get());
		}
		return validators;
	}

  // 创建Binder
  private Binder getBinder() {
    if (this.binder == null) {
      this.binder = new Binder(getConfigurationPropertySources(), getPropertySourcesPlaceholdersResolver(),
                               getConversionServices(), getPropertyEditorInitializer(), null,
                               ConfigurationPropertiesBindConstructorProvider.INSTANCE);
    }
    return this.binder;
  }
}
```

- 最终还是委托给了`Binder`进行调用，`ConfigurationPropertiesBinder`只能算是一个代理，准备好绑定需要的组件，然后调用`Binder`完成绑定
- 我们看到大量的从容器中获取绑定组件的方式，却没看到什么时候在容器中创建了该bean？实际上`getBean()`方法的首次调用就完成了创建和返回两个操作
- 绑定包含了验证的过程，默认会使用两个验证器
  - `ConfigurationPropertiesValidator`
  - `ConfigurationPropertiesJsr303Validator`
- 支持JSR303验证规范的前提条件：同时存在如下三个类型的Bean
  - `Validator`
  - `ValidatorFactory`
  - `GenericBootstrap`

### 事件监听器注册

`ApplicationListenerDetector`，用于检测实现了`ApplicationListener`的Bean，并将其注入容器和时间广播器。

```java
class ApplicationListenerDetector implements DestructionAwareBeanPostProcessor, MergedBeanDefinitionPostProcessor {

  private final transient AbstractApplicationContext applicationContext;

	private final transient Map<String, Boolean> singletonNames = new ConcurrentHashMap<>(256);
  
  @Override
  public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
    // 如果Bean类型是ApplicationListener的子类，则加入缓存
    if (ApplicationListener.class.isAssignableFrom(beanType)) {
      this.singletonNames.put(beanName, beanDefinition.isSingleton());
    }
  }

  @Override
  public Object postProcessAfterInitialization(Object bean, String beanName) {
    if (bean instanceof ApplicationListener) {
      // 缓存里有就注册到容器中
      Boolean flag = this.singletonNames.get(beanName);
      if (Boolean.TRUE.equals(flag)) {
        this.applicationContext.addApplicationListener((ApplicationListener<?>) bean);
      } else if (Boolean.FALSE.equals(flag)) {
        this.singletonNames.remove(beanName);
      }
    }
    return bean;
  }

  @Override
  public void postProcessBeforeDestruction(Object bean, String beanName) {
    // 销毁时移除
    if (bean instanceof ApplicationListener) {
      try {
        ApplicationEventMulticaster multicaster = this.applicationContext.getApplicationEventMulticaster();
        multicaster.removeApplicationListener((ApplicationListener<?>) bean);
        multicaster.removeApplicationListenerBean(beanName);
      } catch (IllegalStateException ex) {
        // ApplicationEventMulticaster not initialized yet - no need to remove a listener
      }
    }
  }

}
```

### 一些注解的实现

- `AutowiredAnnotationBeanPostProcessor`

  `@Autowired`、`@Value`注解

  - 

- `InitDestroyAnnotationBeanPostProcessor`、`CommonAnnotationBeanPostProcessor`

  `@PostConstruct`、`@PreDestroy`

- `RequiredAnnotationBeanPostProcessor`

  `@Required`注解

- `ScheduledAnnotationBeanPostProcessor`

  

### Bean验证相关

- `BeanValidationPostProcessor`



## 总结



### 下一篇写什么

Spring源码剖析 - Bind机制
