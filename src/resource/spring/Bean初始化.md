### Bean记载
单例在Spring的同一个容器内只会被创建一次，后续再获取Bean直接从单例缓存中获取，然后再次尝试从SingletonFactories中加载
创建单例Bean可能存在依赖注入情况，创建依赖的时候为了避免循环依赖，**Spring创建原则是不等bean创建完成，就会将创建bean的ObjectFactory提早曝光到缓存中，一旦下一个bean创建时需要依赖上个bean，则直接使用ObjectFactory**
```java
	protected Object getSingleton(String beanName, boolean allowEarlyReference) {
        //检查缓存中是否存在
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
		    //若不存在，锁定全局变量进行处理
			synchronized (this.singletonObjects) {
				singletonObject = this.earlySingletonObjects.get(beanName);
				//如果此时bean正在加载则不处理，否则执行下面
				if (singletonObject == null && allowEarlyReference) {
					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
					if (singletonFactory != null) {
						singletonObject = singletonFactory.getObject();
						//记录在缓存中，earlySingletonObjects 和 singletonFactories 互斥
						this.earlySingletonObjects.put(beanName, singletonObject);
						this.singletonFactories.remove(beanName);
					}
				}
			}
		}
		return (singletonObject != NULL_OBJECT ? singletonObject : null);
	}
```
- 先从singletonObjects 中获取实例
- 如果获取不到从earlySingletonObject里面获取
- 如果还获取不到，再尝试从singletonFactories里面获取beanName对应的ObjectFactory，然后调用getObject来创建bean
- 放到earlySingletonObject,并从singletonFactories里面删除


- singeltonObjects:用于保存beanName和创建bean实例之间的关系
- earlySingletonObjects:也是用于保存beanName和创建bean实例之间的关系，不同在于当一个bean被放到这里面，那么当bean还在创建过程中，就可以通过getBean方法获取到，其目的是用于检测循环引用
- singletonFactories:用户保存beanName和创建bean的工厂之间的关系
- registeredSingleton:保存当前所有已注册的的bean

### 从bean的实例中获取对象


```java
	protected Object getObjectForBeanInstance(
			Object beanInstance, String name, String beanName, RootBeanDefinition mbd) {

		// Don't let calling code try to dereference the factory if the bean isn't a factory.
		//制定bean是工厂相关（&前缀）且beanInstance 又不是FactoryBean 类型则会报错
		if (BeanFactoryUtils.isFactoryDereference(name) && !(beanInstance instanceof FactoryBean)) {
			throw new BeanIsNotAFactoryException(transformedBeanName(name), beanInstance.getClass());
		}

		// Now we have the bean instance, which may be a normal bean or a FactoryBean，是正常bean还是FactoryBean.
		// If it's a FactoryBean, we use it to create a bean instance, unless the，如果是FactoryBean 直接创建实例
		// caller actually wants a reference to the factory.
		if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
			return beanInstance;
		}

		Object object = null;
		if (mbd == null) {
			object = getCachedObjectForFactoryBean(beanName);
		}
		if (object == null) {
			// beanInstance一定是FactoryBean类型
			FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
			// Caches object obtained from FactoryBean if it is a singleton.
			if (mbd == null && containsBeanDefinition(beanName)) {
				mbd = getMergedLocalBeanDefinition(beanName);
			}
			boolean synthetic = (mbd != null && mbd.isSynthetic());
			object = getObjectFromFactoryBean(factory, beanName, !synthetic);
		}
		return object;
	}
```
- 对FactoryBean正确性验证
- 对非FactoryBean不做任何处理
- 对bean进行转换
- 将从Factory 中解析bean的工作委托给getObjectFromFactoryBean，如果bean是单例，就必须保证全局唯一，缓存存储提高性能。


### 获取单例
前面讲的是从缓存中获取单例过程，如果缓存中不存在已经加载的单例bean就需要从头开始bean的加载。
```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory){
		Assert.notNull(beanName, "'beanName' must not be null");
		//全局变量需同步
		synchronized (this.singletonObjects) {
			Object singletonObject = this.singletonObjects.get(beanName);
			//缓存中没有加载此bean，才需要初始化
			if (singletonObject == null) {
				if (this.singletonsCurrentlyInDestruction) {
					throw new BeanCreationNotAllowedException(beanName,
							"Singleton bean creation not allowed while the singletons of this factory are in destruction " +
							"(Do not request a bean from a BeanFactory in a destroy method implementation!)");
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
				}
				beforeSingletonCreation(beanName);
				boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
				if (recordSuppressedExceptions) {
					this.suppressedExceptions = new LinkedHashSet<Exception>();
				}
				try {
					singletonObject = singletonFactory.getObject();
				}
				catch (BeanCreationException ex) {
					if (recordSuppressedExceptions) {
						for (Exception suppressedException : this.suppressedExceptions) {
							ex.addRelatedCause(suppressedException);
						}
					}
					throw ex;
				}
				finally {
					if (recordSuppressedExceptions) {
						this.suppressedExceptions = null;
					}
					afterSingletonCreation(beanName);
				}
				addSingleton(beanName, singletonObject);
			}
			return (singletonObject != NULL_OBJECT ? singletonObject : null);
		}
	}
```
- 检查缓存中是否已经加载过
- 若没有加载，则记录beanName的正在加载状态
- beforeSingleton加载单例前记录加载状态，将正要创建的bean记录在缓存中，对循环依赖进行检测 ——singletonsCurrentlyInCreation.put(beanName, Boolean.TRUE) != null
- 通过调用参数传入ObjectFactory方法初始化bean
- afterSingletonCreation加载单例后处理方法调用,当bean加载结束后需要移除缓存中对该bean的正在加载状态的记录——singletonsCurrentlyInCreation.remove(beanName)
- addSingleton将结果记录至缓存并删除bean过程中所记录的各种辅助状态
- 返回处理结果

### 准备创建bean

```java
	protected Object createBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)
			throws BeanCreationException {

		if (logger.isDebugEnabled()) {
			logger.debug("Creating instance of bean '" + beanName + "'");
		}
		// Make sure bean class is actually resolved at this point.
		resolveBeanClass(mbd, beanName);

		// Prepare method overrides.
		try {
			mbd.prepareMethodOverrides();
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanDefinitionStoreException(mbd.getResourceDescription(),
					beanName, "Validation of method overrides failed", ex);
		}

		try {
			// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
			Object bean = resolveBeforeInstantiation(beanName, mbd);
			if (bean != null) {
				return bean;
			}
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"BeanPostProcessor before instantiation of bean failed", ex);
		}
        //key 真正干活的
		Object beanInstance = doCreateBean(beanName, mbd, args);
		if (logger.isDebugEnabled()) {
			logger.debug("Finished creating instance of bean '" + beanName + "'");
		}
		return beanInstance;
	}
```
- 根据设置的class属性或根据className来解析class
- 



### 创建bean

```java
	protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args) {
		// Instantiate the bean.
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);
		Class<?> beanType = (instanceWrapper != null ? instanceWrapper.getWrappedClass() : null);

		// Allow post-processors to modify the merged bean definition.
		synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
				applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				mbd.postProcessed = true;
			}
		}

        //是否需要提前曝光：单例&允许循环依赖&当前bean正在创建中
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isDebugEnabled()) {
				logger.debug("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
			//避免后期循环依赖，可以在bean初始化完成前将实例的ObjectFactory加入工厂
			addSingletonFactory(beanName, new ObjectFactory<Object>() {
				public Object getObject() throws BeansException {
					return getEarlyBeanReference(beanName, mbd, bean);
				}
			});
		}

		// Initialize the bean instance.
		Object exposedObject = bean;
		try {
		    //对bean 进行填充，将各个属性注入，可能存在依赖于其他bean的属性，会递归初始依赖bean
			populateBean(beanName, mbd, instanceWrapper);
			if (exposedObject != null) {
				exposedObject = initializeBean(beanName, exposedObject, mbd);
			}
		}
		catch (Throwable ex) {
			if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
				throw (BeanCreationException) ex;
			}
			else {
				throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
			}
		}

		if (earlySingletonExposure) {
			Object earlySingletonReference = getSingleton(beanName, false);
			if (earlySingletonReference != null) {
				if (exposedObject == bean) {
					exposedObject = earlySingletonReference;
				}
				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
					String[] dependentBeans = getDependentBeans(beanName);
					Set<String> actualDependentBeans = new LinkedHashSet<String>(dependentBeans.length);
					for (String dependentBean : dependentBeans) {
						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
							actualDependentBeans.add(dependentBean);
						}
					}
					if (!actualDependentBeans.isEmpty()) {
						throw new BeanCurrentlyInCreationException(beanName,
								"Bean with name '" + beanName + "' has been injected into other beans [" +
								StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
								"] in its raw version as part of a circular reference, but has eventually been " +
								"wrapped. This means that said other beans do not use the final version of the " +
								"bean. This is often the result of over-eager type matching - consider using " +
								"'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
					}
				}
			}
		}

		// Register bean as disposable.
		try {
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
		}

		return exposedObject;
	}
```

### 关于Bean
IoC就是一个对象定义其依赖关系而不创建它们的的过程。
不用new，让Spring控制new过程。
在Spring中，我们基本不需要 new 一个类，这些都是让 Spring 去做的。
Spring 启动时会把所需的类实例化成对象，如果需要依赖，则先实例化依赖，然后实例化当前类。
因为依赖必须通过构建函数传入，所以实例化时，当前类就会接收并保存所有依赖的对象。这一步也就是所谓的依赖注入。
类的实例化、依赖的实例化、依赖的传入都交由Spring Bean容器控制。而不是用new方式实例化对象、通过非构造函数方法传入依赖