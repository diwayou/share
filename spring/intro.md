---
presentation:
  theme: white.css
  width: 1800
  height: 800
  slideNumber: true
---

@import "main.less"

<!-- slide -->

# Spring 系列之 IoC 容器

## 高鹏

<!-- slide -->

## Spring 容器

```puml
top to bottom direction
skinparam defaultTextAlignment center

!include <tupadr3/common>
!include <tupadr3/font-awesome-5/cloud>
!include <tupadr3/font-awesome-5/deviantart>
!include <tupadr3/font-awesome-5/database>
!include <tupadr3/font-awesome-5/ellipsis_h>

!include <tupadr3/devicons/codepen>
!include <tupadr3/devicons/terminal>

!include <tupadr3/material/https>

DEV_CODEPEN(container,容器,node)
FA5_DEVIANTART(aop, AOP, component)
FA5_DATABASE(jdbc, JDBC, component)
MATERIAL_HTTPS(web,Web,component)
DEV_TERMINAL(boot,SpringBoot,component)
FA5_CLOUD(cloud,SpringCloud,component)
FA5_ELLIPSIS_H(ellipsis, ..., component)

aop --> container
jdbc --> container
web --> container
boot --> container
cloud --> container
ellipsis --> container
```

<!-- slide -->

## 依赖注入

```java
@Service
public class StoreManager {

    @Autowired
    private StoreService storeService;

    public Store get(int id) {
        return storeService.get(id);
    }
}
public static void main(String[] args) {
    ApplicationContext ctx = new AnnotationConfigApplicationContext("com.tgou");
    StoreManager storeManager = ctx.getBean(StoreManager.class);
    Store store = storeManager.get(1);
}
```

<!-- slide -->

## 思考几个问题

- Spring 如何获取 Bean 的配置信息？
- Spring 如何管理 Bean 的？
- Bean 的生命周期是什么样的？
- 依赖是什么时间点注入的？
- 如何扩展增强 Spring 功能？
<!-- slide -->

## Spring 核心类

```puml
interface BeanFactory
interface ApplicationContext
class XxxBeanDefinitionReader
class XxxApplicationContext
class DefaultListableBeanFactory

XxxApplicationContext ..> XxxBeanDefinitionReader: 使用
XxxApplicationContext *--> DefaultListableBeanFactory: 组合
ApplicationContext <|.. XxxApplicationContext: 实现
BeanFactory <|.. DefaultListableBeanFactory: 实现
BeanFactory <|-- ApplicationContext: 继承
```

<!-- slide -->

## Bean 创建方式

- Xml - `XmlBeanDefinitionReader`
- Annotation - `AnnotatedBeanDefinitionReader,ClassPathBeanDefinitionScanner`
- Java - `AnnotatedBeanDefinitionReader,ClassPathBeanDefinitionScanner`
<!-- slide -->

## 获取 Bean 配置信息

```java
public class AnnotationConfigApplicationContext extends GenericApplicationContext implements AnnotationConfigRegistry {
  public AnnotationConfigApplicationContext() {
    this.reader = new AnnotatedBeanDefinitionReader(this);
    this.scanner = new ClassPathBeanDefinitionScanner(this);
  }

  public AnnotationConfigApplicationContext(String... basePackages) {
    this();
    scan(basePackages);
    refresh();
  }
  
  public void scan(String... basePackages) {
    Assert.notEmpty(basePackages, "At least one base package must be specified");
    this.scanner.scan(basePackages);
  }
}
```

<!-- slide -->

## Bean 信息存储核心类

```puml
interface BeanDefinition
interface AnnotatedBeanDefinition
class AbstractBeanDefinition
class RootBeanDefinition
class ChildBeanDefinition
class GenericBeanDefinition
class AnnotatedGenericBeanDefinition
class ScannedGenericBeanDefinition

BeanDefinition <|-- AnnotatedBeanDefinition:继承
BeanDefinition <|.. AbstractBeanDefinition:实现
AbstractBeanDefinition <|-- RootBeanDefinition:继承
AbstractBeanDefinition <|-- ChildBeanDefinition:继承
AbstractBeanDefinition <|-- GenericBeanDefinition:继承
GenericBeanDefinition <|-- AnnotatedGenericBeanDefinition:继承
GenericBeanDefinition <|-- ScannedGenericBeanDefinition:继承
AnnotatedBeanDefinition <|.. AnnotatedGenericBeanDefinition:实现
AnnotatedBeanDefinition <|.. ScannedGenericBeanDefinition:实现
```

<!-- slide -->

## Bean 信息存储详情

```puml
interface BeanDefinition
class AbstractBeanDefinition {

 -String[] dependsOn
 -boolean primary = false
 -ConstructorArgumentValues constructorArgumentValues
 -String initMethodName
 -String destroyMethodName

 +void setDependsOn(@Nullable String... dependsOn)
 +String[] getDependsOn()
 +void setPrimary(boolean primary)
 +boolean isPrimary()
 +ConstructorArgumentValues getConstructorArgumentValues()
 +void setInitMethodName(@Nullable String initMethodName)
 +String getInitMethodName()
 +void setDestroyMethodName(@Nullable String destroyMethodName);
 +String getDestroyMethodName();
}

BeanDefinition <|.. AbstractBeanDefinition:实现
```

<!-- slide -->

## 注册 Bean 到容器中(BeanDefinitionReaderUtils)

```java
 public static void registerBeanDefinition(
    BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
    throws BeanDefinitionStoreException {

    // Register bean definition under primary name.
    String beanName = definitionHolder.getBeanName();

    // 这一行就是注册bean的元数据信息
    registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

    // Register aliases for bean name, if any.
    String[] aliases = definitionHolder.getAliases();
    if (aliases != null) {
      for (String alias : aliases) {
        registry.registerAlias(beanName, alias);
      }
    }
 }
```

<!-- slide -->

## 注册 Bean 到容器中(DefaultListableBeanFactory)

```puml
interface BeanDefinitionRegistry
class DefaultListableBeanFactory {
  -final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256)
  -volatile List<String> beanDefinitionNames = new ArrayList<>(256)

  +void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
}

BeanDefinitionRegistry <|.. DefaultListableBeanFactory:实现
```

<!-- slide -->

## 构造 Bean 容器

```java
public class AnnotationConfigApplicationContext extends GenericApplicationContext implements AnnotationConfigRegistry {

  public AnnotationConfigApplicationContext(String... basePackages) {
    this();
    // 扫描bean信息并注册
    scan(basePackages);
    // 构造bean容器
    refresh();
  }
}
public class GenericApplicationContext extends AbstractApplicationContext implements BeanDefinitionRegistry {

  private final DefaultListableBeanFactory beanFactory;

  public GenericApplicationContext() {
    // 在这创建DefaultListableBeanFactory，来帮助管理bean信息
    this.beanFactory = new DefaultListableBeanFactory();
  }
}
```

<!-- slide -->

## refresh()

```java
public abstract class AbstractApplicationContext extends DefaultResourceLoader implements ConfigurableApplicationContext {
  @Override
  public void refresh() throws BeansException, IllegalStateException {
    ...
    prepareBeanFactory(beanFactory);
    registerBeanPostProcessors(beanFactory);
    finishBeanFactoryInitialization(beanFactory);
    ...
  }
  protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    ...
    beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
    beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
    beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
    beanFactory.registerResolvableDependency(ApplicationContext.class, this);
    ...
  }
  protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
    ...
    beanFactory.preInstantiateSingletons();
  }
}
```

<!-- slide -->

## 遍历构造 Bean

```java
public class DefaultListableBeanFactory extends AbstractAutowireCapableBeanFactory implements ConfigurableListableBeanFactory, BeanDefinitionRegistry, Serializable {
  @Override
  public void preInstantiateSingletons() throws BeansException {
    ...
    for (String beanName : beanNames) {
      ...
      getBean(beanName);
    }

    for (String beanName : beanNames) {
      Object singletonInstance = getSingleton(beanName);
      // 扩展点
      if (singletonInstance instanceof SmartInitializingSingleton) {
        smartSingleton.afterSingletonsInstantiated();
      }
    }
  }
}
```

<!-- slide -->

## 构造 Bean

```java
public abstract class AbstractBeanFactory extends FactoryBeanRegistrySupport implements ConfigurableBeanFactory {
  protected <T> T doGetBean(String name, @Nullable Class<T> requiredType, @Nullable Object[] args,boolean typeCheckOnly) throws BeansException {
    ...
    if (mbd.isSingleton()) {
      // 创建Bean并存储
      sharedInstance = getSingleton(beanName, () -> {
        try {
          // 真正去创建Bean
          return createBean(beanName, mbd, args);
        }
        catch (BeansException ex) {
          destroySingleton(beanName);
          throw ex;
        }
      });
      bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
    }
    ...
    return (T) bean;
  }
}
```

<!-- slide -->

## Bean 的存储

```java
public class DefaultSingletonBeanRegistry extends SimpleAliasRegistry implements SingletonBeanRegistry {
  private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
  private final Set<String> registeredSingletons = new LinkedHashSet<>(256);

  public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
    ...
    singletonObject = singletonFactory.getObject();
    ...
    addSingleton(beanName, singletonObject);
  }
  protected void addSingleton(String beanName, Object singletonObject) {
    synchronized (this.singletonObjects) {
      this.singletonObjects.put(beanName, singletonObject);
      this.singletonFactories.remove(beanName);
      this.earlySingletonObjects.remove(beanName);
      this.registeredSingletons.add(beanName);
    }
  }
}
```

<!-- slide -->

## 构造 Bean 1

```java
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory implements AutowireCapableBeanFactory {
  protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
    // 创建bean实例
    BeanWrapper instanceWrapper = null;
    if (mbd.isSingleton()) {
      instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }
    if (instanceWrapper == null) {
      instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
    ...
    // 注入依赖
    populateBean(beanName, mbd, instanceWrapper);
    // 构造bean
    exposedObject = initializeBean(beanName, exposedObject, mbd);
    ...
  }
}
```

<!-- slide -->

## 构造 Bean 2

```java
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory implements AutowireCapableBeanFactory {
  protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
    ...
    int resolvedAutowireMode = mbd.getResolvedAutowireMode();
    if (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
      MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
      // 按照name注入
      if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {
        autowireByName(beanName, mbd, bw, newPvs);
      }
      // 按照类型注入
      if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
        autowireByType(beanName, mbd, bw, newPvs);
      }
      pvs = newPvs;
    }
    ...
    if (pvs != null) {
      // 注入
      applyPropertyValues(beanName, mbd, bw, pvs);
    }
  }
}
```

<!-- slide -->

## 构造 Bean 3

```java
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory implements AutowireCapableBeanFactory {
  protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {
    // 注入XxxAware方法
    invokeAwareMethods(beanName, bean);
    Object wrappedBean = bean;
    if (mbd == null || !mbd.isSynthetic()) {
      // 调用BeanPostProcessor Before操作
      wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    }
    // 调用init方法，包含InitializingBean和自定义的
    invokeInitMethods(beanName, wrappedBean, mbd);
    if (mbd == null || !mbd.isSynthetic()) {
      // 调用BeanPostProcessor After操作
      wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }

    return wrappedBean;
  }
}
```

<!-- slide -->

## ApplicationContextAwareProcessor

```java
class ApplicationContextAwareProcessor implements BeanPostProcessor {
  public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
    ...
    invokeAwareInterfaces(bean);
    return bean;
  }
  private void invokeAwareInterfaces(Object bean) {
    if (bean instanceof EnvironmentAware) {
      ((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
    }
    if (bean instanceof ResourceLoaderAware) {
      ((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
    }
    if (bean instanceof ApplicationContextAware) {
      ((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
    }
  }
}
```

<!-- slide -->

## Bean 生命周期和扩展点

1. 实例化最基本的对象
2. 注入普通 Bean 依赖
3. 注入 BeanNameAware、BeanClassLoaderAware、BeanFactoryAware
4. BeanPostProcessor.postProcessBeforeInitialization
   1. ApplicationContextAwareProcessor
      1. EnvironmentAware
      2. ResourceLoaderAware
      3. ApplicationEventPublisherAware
      4. ApplicationStartupAware
      5. ApplicationContextAware
5. InitializingBean 和自定义的 init 方法
6. BeanPostProcessor.postProcessAfterInitialization
7. SmartInitializingSingleton.afterSingletonsInstantiated
8. DisposableBean 和自定义的 destroy 方法

<!-- slide -->

## 回答这几个问题

- Spring 如何获取 Bean 的配置信息？
- Spring 如何管理 Bean 的？
- Bean 的生命周期是什么样的？
- 依赖是什么时间点注入的？
- 如何扩展增强 Spring 功能？
<!-- slide -->

# 谢谢
