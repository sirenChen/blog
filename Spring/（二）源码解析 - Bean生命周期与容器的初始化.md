上次稍微梳理了一下Bean的生命周期，但是总有一种不明不白的感觉。正好又看书看到容器初始化的相关知识，发现有点和Bean的初始化过程衔接不上。所以在网上搜各种技术博客，最后还是发现源码才是解决一切问题的答案。在经过自己稍微的一点调试，先总结于此，以免以后忘记。

# Refresh！！！
上次分析了initializeBean这个方法，但是不知道这个方法从哪里来到哪里去，所以让我从头说起。我们都知道，下面这段代码用来启动容器:  
```
ApplicationContext ctx = new ClassPathXmlApplicationContext("beans.xml");
```
这段代码背后发生了什么？事实上，在ApplicationContext的抽象实现类AbstractApplicationContext中，定义了refresh()方法，这个方法就完全就是容器初始化的各种步骤。  
```
    public void refresh() throws BeansException, IllegalStateException {
            Object var1 = this.startupShutdownMonitor;
            synchronized(this.startupShutdownMonitor) {
                this.prepareRefresh();
                //1 Spring将配置文件装入BeanDefinitionRegistry
                ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();
                
                this.prepareBeanFactory(beanFactory);

                try {
                    this.postProcessBeanFactory(beanFactory);
                    
                    //2 调用后工厂处理器
                    this.invokeBeanFactoryPostProcessors(beanFactory);
                    
                    //3 注册Bean后处理器
                    this.registerBeanPostProcessors(beanFactory);
                    
                    //4 初始化消息源
                    this.initMessageSource();
                    
                    //5 初始化应用上下文事件广播器
                    this.initApplicationEventMulticaster();
                    
                    //6 初始化其他特殊的Bean
                    this.onRefresh();
                    
                    //7 注册时间监听器
                    this.registerListeners();
                    
                    //8 初始化所有单实例Bean，
                    this.finishBeanFactoryInitialization(beanFactory);
                    
                    //9 完成刷新并发布容器刷新事件
                    this.finishRefresh();
                } catch (BeansException var9) {
                    if(this.logger.isWarnEnabled()) {
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
上面的九步注释是我从《精通Spring 4.x》这本书上摘抄下来的。由于时间有限，暂时没有每一步都去详细的了解。今天先看与Bean生命周期有关的几个步骤。让我们对着上一篇的Bean的生命周期的那张图来看。

## BeanFactoryPostProcessor
先看第2步。进入invokeBeanFactoryPostProcessors内部，发现如下代码
```
PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, this.getBeanFactoryPostProcessors());
```
可以看出，在这里调用了BeanFactoryPostProcessor。这一步，也就对应了的Bean的生命周期的第一步：调用BeanFactoryPostProcessor的postProcessBeanFactory()方法。  

## BeanPostProcessor
再看第3步。进入registerBeanPostProcessors， 发现代码
```
PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
```
在往下走一步会发现，这一步代码的主要意思就是：从BeanDefinitionRegistry中找出所有实现了BeanPostProcessor接口的Bean，并把他们注册到容器。这一段虽然不对应生命周期，但是这里注册了BeanPostProcessor，算是为后面调用BeanPostProcessor做了铺垫。  

# finishBeanFactoryInitialization(beanFactory);  
这是与Bean的生命周期相关最重要的方法。这个方法特别有意思，它虽然叫finish bean factory Initialization， 但是它做的却是初始化Bean的事情。它既是完成BeanFactory初始化的方法，也是初始化Bean的方法。这一段逻辑不好解释，我直接把几个重要执行步骤列出来：  
1. AbstractApplicationContext # finishBeanFactoryInitialization()
2. DefaultListableBeanFactory # preInstantiateSingletons()
3. AbstractBeanFactory # getBean()
4. AbstractBeanFactory # doGetBean()
4. AbstractAutowireCapableBeanFactory # createBean()
5. AbstractAutowireCapableBeanFactory # doCreateBean()
6. AbstractAutowireCapableBeanFactory # initializeBean()
终于到了这一步，initializeBean这个方法，到这里就对应上了我们上一篇讲的initializeBean这个方法。但是中间还缺了几个步骤，比如实例化之类的。让我么接着看。

## createBean()
同样，源码太多，就不全部贴出来，看一下最重要的几个步骤。这几个方法都在AbstractAutowireCapableBeanFactory这个类中
从createBean()开始：
1. resolveBeforeInstantiation() 
2. applyBeanPostProcessorsBeforeInstantiation()
    在这个方法中调用Object result = ibp.postProcessBeforeInstantiation(beanClass, beanName);  
3. doCreateBean()
4. createBeanInstance()  
5. populateBean()
    调用postProcessAfterInstantiation()  
    调用applyPropertyValues 设置属性值
5. initializeBean()
其实这里不是很好解释，但是去翻源码，从createBean这个方法开始，照着我的顺序一步一步的看，就非常明白了。但是我想说的是，这里对应了生命周期中的这几步：
1. 调用InstantiationAwareBeanPostProcessor的postProcessBeforeInstantiation()
2. 实例化
3. 调用InstantiationAwareBeanPostProcessor的postProcessAfterInstantiation()
4. 设置属性值


## initializeBean()
```
    protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
        if(System.getSecurityManager() != null) {
            AccessController.doPrivileged(new PrivilegedAction<Object>() {
                public Object run() {
                    AbstractAutowireCapableBeanFactory.this.invokeAwareMethods(beanName, bean);
                    return null;
                }
            }, this.getAccessControlContext());
        } else {
            //========1========//
            this.invokeAwareMethods(beanName, bean);
        }

        Object wrappedBean = bean;
        if(mbd == null || !mbd.isSynthetic()) {
            //========2========//
            wrappedBean = this.applyBeanPostProcessorsBeforeInitialization(bean, beanName);
        }

        try {
            //========3========//
            this.invokeInitMethods(beanName, wrappedBean, mbd);
        } catch (Throwable var6) {
            throw new BeanCreationException(mbd != null?mbd.getResourceDescription():null, beanName, "Invocation of init method failed", var6);
        }

        if(mbd == null || !mbd.isSynthetic()) {
            //========4========//
            wrappedBean = this.applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
        }

        return wrappedBean;
    }
```
接下来的步骤就很明显了，结合着上一篇我分析过的，稍微解释一下：
1. 查看invokeAwareMethods源码，发现会调用：
    1. BeanNameAware # setBeanFactory()
    2. BeanFactoryAware # setBeanFactory()
    3. ApplicationContextAware # setApplicationContext()
2. 查看applyBeanPostProcessorsBeforeInitialization(bean, beanName)，调用BeanPostProcessor的postProcessorBeforeInitialization()方法
3. 查看invokeInitMethods(beanName, wrappedBean, mbd)，发现会调用
    1. InitializingBean # afterPropertiesSet()
    2. 配置的init-method 初始化方法
4. 查看applyBeanPostProcessorsAfterInitialization(bean, beanName)，调用BeanPostProcessor的postProcessorAfterInitialization()方法。

# 总结
至此，容器的初始化，和bean的生命周期总算是一一对应上了。一切都是从refresh()这个函数开始的，一个一个的源码去翻，总会搞清楚的。这里我要推荐一片博客，我在读源码的过程中参考了很多他的博客。
http://blog.csdn.net/jintao_ma/article/details/52312152

