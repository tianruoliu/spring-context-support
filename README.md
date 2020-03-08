项目地址： https://github.com/alibaba/spring-context-support

> 本篇笔记中的Spring Framework基于5.2 ，spring-context-support版本1.0.3

# 使用场景

## 工具类  -`com.alibaba.spring.util`

> 为`spring-core`提供工具类的补充

`spring-core`中提供了大量工具类

- `StringUtils`

  > `StringUtils`对子字符串的支持不是很好 , 本项目对其进行了补充

+ `ClassUtils`

  ```java
  public static ClassLoader getDefaultClassLoader() {
  		ClassLoader cl = null;
  		try {
              // 获取当前线程的ClassLoader
  			cl = Thread.currentThread().getContextClassLoader();
  		}
  		catch (Throwable ex) {
  			// Cannot access thread context ClassLoader - falling back...
  		}
  		if (cl == null) {
  			// No thread context class loader -> use class loader of this class.
  			cl = ClassUtils.class.getClassLoader();
  			if (cl == null) {
  				// getClassLoader() returning null indicates the bootstrap ClassLoader
  				try {
  					cl = ClassLoader.getSystemClassLoader();
  				}
  				catch (Throwable ex) {
  					// Cannot access system ClassLoader - oh well, maybe the caller can live with null...
  				}
  			}
  		}
  		return cl;
  	}
  ```




## Spring Beans相关扩展   - `com.alibaba.spring.beans`

### Spring Beans生命周期后置处理`BeanPostProcessor`(`org.springframework.beans.factory.config`)

#### 基本语义

+ 处理Bean初始化生命周期，包括before和after
+ 在回调方法处理时，可能会对对象变化，但是多数情况不需要变化类型，即改变当前Bean参数对象中的状态即可

```java
public interface BeanPostProcessor {
	@Nullable
	default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}
    
	@Nullable
	default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}

}
```

自定义`BeanPostProcessor`

```java
class MyBeanPostProcessor implements BeanPostProcessor {
        
        @Override
        public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
            if (ClassUtils.isAssignable(bean.getClass(), CharSequence.class)) { // 凡是Bean类型为CharSequence子类型
                // 被转化为String
                return bean.toString();
            }
            return bean;
        }
    }
```

#### 实现细节

在Spring Framework 5.0以前，我们需要完全实现`BeanPostProcessor`接口的方法:

+ 初始化前   -` postProcessBeforeInitialization`
+ 初始化后   - `postProcessAfterInitialization`

使用不便的地方：

+ 对bean类型通常需要自行判断

在Spring Framework 5.0+ 开始，`BeanPostProcessor`提供了默认实现，即没有操作直接返回。

> 这个接口为什么还可以return呢

`com.alibaba.spring.beans.factory.config.GenericBeanPostProcessorAdapter`

```java
public abstract class GenericBeanPostProcessorAdapter<T> implements BeanPostProcessor {

    private final Class<T> beanType;

    public GenericBeanPostProcessorAdapter() {
        ParameterizedType parameterizedType = (ParameterizedType) getClass().getGenericSuperclass();
        Type[] actualTypeArguments = parameterizedType.getActualTypeArguments();
        this.beanType = (Class<T>) actualTypeArguments[0];
    }

    @Override
    public final Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        if (ClassUtils.isAssignableValue(beanType, bean)) { 
            return doPostProcessBeforeInitialization((T) bean, beanName);
        }
        return bean;
    }

    @Override
    public final Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        if (ClassUtils.isAssignableValue(beanType, bean)) {
            return doPostProcessAfterInitialization((T) bean, beanName);
        }
        return bean;
    }
```



#### 风险规避

`BeanPostProcessor`会被所有的bean所回调

>  N  Spring Beans    -> M `BeanPostProcessor`   -> 操作次数  M*N

```java
// AbstractBeanFactory#addBeanPostProcessor  
@Override
	public void addBeanPostProcessor(BeanPostProcessor beanPostProcessor) {
		Assert.notNull(beanPostProcessor, "BeanPostProcessor must not be null");
		// Remove from old position, if any
		this.beanPostProcessors.remove(beanPostProcessor);
		// Track whether it is instantiation/destruction aware
		if (beanPostProcessor instanceof InstantiationAwareBeanPostProcessor) {
			this.hasInstantiationAwareBeanPostProcessors = true;
		}
		if (beanPostProcessor instanceof DestructionAwareBeanPostProcessor) {
			this.hasDestructionAwareBeanPostProcessors = true;
		}
		// Add to end of list
		this.beanPostProcessors.add(beanPostProcessor);
	}
```

`AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsBeforeInitialization`

```java
   @Override
	public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
			throws BeansException {

		Object result = existingBean;
		for (BeanPostProcessor processor : getBeanPostProcessors()) {
			Object current = processor.postProcessBeforeInitialization(result, beanName);
            // 假设我们自定义的BeanPostProcessor回调方法return null ，我们依旧返回当前bean，如果这一步不做处理，那么任由他返回null ，就会影响到整个BeanFactory中Bean的使用
			if (current == null) {
				return result;
			}
			result = current;
		}
		return result;
	}
```

#### 与`BeanFactory`的关系

关联关系：`BeanPostProcessor`与`BeanFactory` 是N对1的关系，即一个`BeanFactory`实例可以关联N个`BeanPostProcessor`。

`BeanPostProcessor`的来源:

+ 可以是显示地插入,如`ConfigurableBeanFactory#addBeanPostProcessor()`；
+ 也可以将`BeanPostProcessor`定义成普通的Spring Bean,即`AbstractApplicationContext#prepareBeanFactory`

`AbstractApplicationContext#refresh`

```java
public void refresh() throws BeansException, IllegalStateException {
        synchronized(this.startupShutdownMonitor) {
            this.prepareRefresh();
            ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();
            // 这里要看一下
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



`AbstractApplicationContext#prepareBeanFactory`

```java
  protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        beanFactory.setBeanClassLoader(this.getClassLoader());
        beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
        beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, this.getEnvironment()));
        // 这个很熟悉吧
        beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
        beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
        beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
        beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
        beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
        beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
        beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
        beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
        beanFactory.registerResolvableDependency(ResourceLoader.class, this);
        beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
        beanFactory.registerResolvableDependency(ApplicationContext.class, this);
       // 这里也有
        beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));
        if (beanFactory.containsBean("loadTimeWeaver")) {
            beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
            beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
        }

        if (!beanFactory.containsLocalBean("environment")) {
            beanFactory.registerSingleton("environment", this.getEnvironment());
        }

        if (!beanFactory.containsLocalBean("systemProperties")) {
            beanFactory.registerSingleton("systemProperties", this.getEnvironment().getSystemProperties());
        }

        if (!beanFactory.containsLocalBean("systemEnvironment")) {
            beanFactory.registerSingleton("systemEnvironment", this.getEnvironment().getSystemEnvironment());
        }

    }
```



### Spring BeanFactory生命周期后置处理器 - BeanFactoryPostProcessor

#### 基本语义

自定义处理`ConfigurableListableBeanFactory`，当它已经被基本处理完成，即`org.springframework.context.support.AbstractApplicationContext#prepareBeanFactory`方法处理完成。

```java
@FunctionalInterface
public interface BeanFactoryPostProcessor {

	/**
	 * Modify the application context's internal bean factory after its standard
	 * initialization. All bean definitions will have been loaded, but no beans
	 * will have been instantiated yet. This allows for overriding or adding
	 * properties even to eager-initializing beans.
	 * @param beanFactory the bean factory used by the application context
	 * @throws org.springframework.beans.BeansException in case of errors
	 */
	void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;

}

```

#### 生命周期回调方法 -`postProcessBeanFactory`

+ 方法参数  -  ` ConfigurableListableBeanFactory`

+ 回调时机  - `AbstractApplicationContext#invokeBeanFactoryPostProcessors`

  ```java
  protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
         PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, this.getBeanFactoryPostProcessors());
         if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean("loadTimeWeaver")) {
             beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
             beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
         }
   
     }
  
  ```

  + 回调  `BeanFactoryPostProcessor`

  + 回调   `BeanDefinitionRegistryPostProcessor`   -注册`BeanDefinition`

    ```java
    public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {
    
    	/**
    	 * Modify the application context's internal bean definition registry after its
    	 * standard initialization. All regular bean definitions will have been loaded,
    	 * but no beans will have been instantiated yet. This allows for adding further
    	 * bean definitions before the next post-processing phase kicks in.
    	 * @param registry the bean definition registry used by the application context
    	 * @throws org.springframework.beans.BeansException in case of errors
    	 */
    	void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;
    
    }
    
    ```

#### 理解回调细节

+ `AbstractApplicationContext#invokeBeanFactoryPostProcessors`

  + `PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, this.getBeanFactoryPostProcessors());`

    ```java
    public static void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {
            Set<String> processedBeans = new HashSet();
            ArrayList regularPostProcessors;
            ArrayList registryProcessors;
            int var9;
            ArrayList currentRegistryProcessors;
            String[] postProcessorNames;
            if (beanFactory instanceof BeanDefinitionRegistry) {
                BeanDefinitionRegistry registry = (BeanDefinitionRegistry)beanFactory;
                regularPostProcessors = new ArrayList();
                registryProcessors = new ArrayList();
                Iterator var6 = beanFactoryPostProcessors.iterator();
    
                while(var6.hasNext()) {
                    BeanFactoryPostProcessor postProcessor = (BeanFactoryPostProcessor)var6.next();
                    if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
                        BeanDefinitionRegistryPostProcessor registryProcessor = (BeanDefinitionRegistryPostProcessor)postProcessor;
                        // 回调
                        registryProcessor.postProcessBeanDefinitionRegistry(registry);
                        registryProcessors.add(registryProcessor);
                    } else {
                        regularPostProcessors.add(postProcessor);
                    }
                }
    
                currentRegistryProcessors = new ArrayList();
                postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
                String[] var16 = postProcessorNames;
                var9 = postProcessorNames.length;
    
                int var10;
                String ppName;
                for(var10 = 0; var10 < var9; ++var10) {
                    ppName = var16[var10];
                    if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
                        currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                        processedBeans.add(ppName);
                    }
                }
    
                sortPostProcessors(currentRegistryProcessors, beanFactory);
                registryProcessors.addAll(currentRegistryProcessors);
                invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
                currentRegistryProcessors.clear();
                postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
                var16 = postProcessorNames;
                var9 = postProcessorNames.length;
    
                for(var10 = 0; var10 < var9; ++var10) {
                    ppName = var16[var10];
                    if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
                        currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                        processedBeans.add(ppName);
                    }
                }
    
                sortPostProcessors(currentRegistryProcessors, beanFactory);
                registryProcessors.addAll(currentRegistryProcessors);
                invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
                currentRegistryProcessors.clear();
                boolean reiterate = true;
    
                while(reiterate) {
                    reiterate = false;
                    postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
                    String[] var19 = postProcessorNames;
                    var10 = postProcessorNames.length;
    
                    for(int var26 = 0; var26 < var10; ++var26) {
                        String ppName = var19[var26];
                        if (!processedBeans.contains(ppName)) {
                            currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                            processedBeans.add(ppName);
                            reiterate = true;
                        }
                    }
    
                    sortPostProcessors(currentRegistryProcessors, beanFactory);
                    registryProcessors.addAll(currentRegistryProcessors);
                    invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
                    currentRegistryProcessors.clear();
                }
    
                invokeBeanFactoryPostProcessors((Collection)registryProcessors, (ConfigurableListableBeanFactory)beanFactory);
                invokeBeanFactoryPostProcessors((Collection)regularPostProcessors, (ConfigurableListableBeanFactory)beanFactory);
            } else {
                // 回调
                invokeBeanFactoryPostProcessors((Collection)beanFactoryPostProcessors, (ConfigurableListableBeanFactory)beanFactory);
            }
    
            String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);
            regularPostProcessors = new ArrayList();
            registryProcessors = new ArrayList();
            currentRegistryProcessors = new ArrayList();
            postProcessorNames = postProcessorNames;
            int var20 = postProcessorNames.length;
    
            String ppName;
            for(var9 = 0; var9 < var20; ++var9) {
                ppName = postProcessorNames[var9];
                if (!processedBeans.contains(ppName)) {
                    if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
                        regularPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
                    } else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
                        registryProcessors.add(ppName);
                    } else {
                        currentRegistryProcessors.add(ppName);
                    }
                }
            }
    
            sortPostProcessors(regularPostProcessors, beanFactory);
            invokeBeanFactoryPostProcessors((Collection)regularPostProcessors, (ConfigurableListableBeanFactory)beanFactory);
            List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList(registryProcessors.size());
            Iterator var21 = registryProcessors.iterator();
    
            while(var21.hasNext()) {
                String postProcessorName = (String)var21.next();
                orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
            }
    
            sortPostProcessors(orderedPostProcessors, beanFactory);
            invokeBeanFactoryPostProcessors((Collection)orderedPostProcessors, (ConfigurableListableBeanFactory)beanFactory);
            List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList(currentRegistryProcessors.size());
            Iterator var24 = currentRegistryProcessors.iterator();
    
            while(var24.hasNext()) {
                ppName = (String)var24.next();
                nonOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
            }
    
            invokeBeanFactoryPostProcessors((Collection)nonOrderedPostProcessors, (ConfigurableListableBeanFactory)beanFactory);
            beanFactory.clearMetadataCache();
        }
    
    ```

    `invokeBeanFactoryPostProcessors`

    ```java
    private static void invokeBeanFactoryPostProcessors(Collection<? extends BeanFactoryPostProcessor> postProcessors, ConfigurableListableBeanFactory beanFactory) {
            Iterator var2 = postProcessors.iterator();
    
            while(var2.hasNext()) {
                BeanFactoryPostProcessor postProcessor = (BeanFactoryPostProcessor)var2.next();
                postProcessor.postProcessBeanFactory(beanFactory);
            }
    
        }
    
    ```

#### 与 ApplicationContext 的关系

关联关系 -`BeanFactory`与`ApplicationContext`是**一对一**的关系(绝大多数情况`BeanFactory`的实现是`DefaultListableBeanFactory`，并且`ApplicationContext`的实现类是`AbstractApplicationContext`的子类）

佐证：

```java
// AbstractApplicationContext#getAutowireCapableBeanFactory 
public AutowireCapableBeanFactory getAutowireCapableBeanFactory() throws IllegalStateException {
        return this.getBeanFactory();
    }

```

`BeanFactoryPostProcessor`与`ApplicationContext`的关系是N对1

> // AbstractApplicationContext
>
> `private final List<BeanFactoryPostProcessor> beanFactoryPostProcessors;`// 有序，有一个排序过程

`BeanFactoryPostProcessor`的来源

+ 显示地插入 ，即调用`AbstractApplicationContext#addBeanFactoryPostProcessor`

  ```java
  // AbstractApplicationContext
  public void addBeanFactoryPostProcessor(BeanFactoryPostProcessor postProcessor) {
       Assert.notNull(postProcessor, "BeanFactoryPostProcessor must not be null");
       this.beanFactoryPostProcessors.add(postProcessor);
  }
  
  ```

+ `BeanFactoryPostProcessor`定义成普通的Spring Bean , 即`AbstractApplicationContext#invokeBeanFactoryPostProcessors`方法处理

  ```java
  // PostProcessorRegistrationDelegate#PostProcessorRegistrationDelegate
  .....
   postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
  			for (String ppName : postProcessorNames) {
  				if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
  					currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
  					processedBeans.add(ppName);
  				}
  			}
  sortPostProcessors(currentRegistryProcessors, beanFactory);
  
  ```

  



## `BeanDefinition`

### 基本语义

`BeanDefinition`是Bean声明元数据的一种描述接口.

+ 主要元数据指标
  + Bean类型    - `getBeanClassName();`
  + Parent Bean名称   -` getParentName();`
  + 工厂Bean名称   -`getFactoryBeanName()`
  + 工厂方法名称   - `getFactoryMethodName`()
  + Bean生命周期范围 - `getScope()`
+ 次要元数据指标
  + Bean是否懒加载   -`isLazyInit();
  + 是否为Primary    - `isPrimary()`

### 黑科技

+ 特殊元数据指标
  + Bean定义的来源（xml /Java Config )  - `setSource(Object source)`  用于区分不同Bean定义的一种手段
  + Bean定义属性上下文  -`setAtribute(String,Object) ` 临时存储在当前Bean定义的属性，它不影响Bean实例化/初始化，然而可以辅助Bean初始化

## `BeanFactory`

  子接口

+ `ListableBeanFactory `  - 在`BeanFactory`基础上，主要职责是扩展对`BeanDefinition`的管理以及Bean对象的集合返回。

  ```java
  public interface ListableBeanFactory extends BeanFactory {
      boolean containsBeanDefinition(String beanName);
      int getBeanDefinitionCount();
      String[] getBeanNamesForType(ResolvableType type);
      ...
  }
  
  ```

+ `HierarchicalBeanFactory`    - 层次性的`BeanFactory`，类似于`ClassLoader`的双亲委派机制（**并不是继承关系**）

  `ClassLoader#getParent()` 

  ```java
  public interface HierarchicalBeanFactory extends BeanFactory {
  
  	@Nullable
  	BeanFactory getParentBeanFactory();
  
      // bean 是你自己本地的 ，还是你父亲的
  	boolean containsLocalBean(String name);
  
  }
  
  ```

+ `ConfigurableBeanFactory`  - 可配置（可写）的`BeanFactory` ,相对于其他`BeanFactory` `只读的特性

> 为什么BeanFactory有多个，因为我们可以是xml文件配置的，也可以是注解驱动的，也可以是显示代码写的。我们可以从Spring MVC 的两个ApplicationContext理解，这个无需多言。



## 常见问题

### BeanFactory与FactoryBean的区别和联系

`FactoryBean`   - 创建特定Bean的工厂 ，其对象是`BeanFactory`的一个普通Bean

 主要特性

- 指定Bean类型    - `getObjectType`
- 获取/创建Bean   - `getObjectType`
- 创建Bean 是否为单例  – `isSingleton`

```java
public interface FactoryBean<T> {
	String OBJECT_TYPE_ATTRIBUTE = "factoryBeanObjectType";
    
	@Nullable
	T getObject() throws Exception;
    @Nullable
	Class<?> getObjectType();
    
	default boolean isSingleton() {
		return true;
	}
}

```

举例：`UserFactoryBean` 创建User Bean  ,通常应用关心User Bean ,而非关心`UserFactoryBean` `.

`BeanFactory`是Spring容器（接口），也管理`FactoryBean `



## 思考

我们在定义一个接口的时候，对其方法的入参，要尽可能抽象一点；但是在具体实现接口方法时，可以调整入参为更具体的