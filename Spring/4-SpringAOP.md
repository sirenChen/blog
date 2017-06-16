# （四）SpringAOP源码分析
看过最基本的JDK动态代理之后，也就大致了解了SpringAOP到底要干什么，主要是为了把一些重复的，比如日志之类的代码，抽取出来，让业务逻辑里面只有业务。我们已经大致知道SpringAOP的底层就是动态代理，那么我们现在就分析一下SpringAOP到底是如何实现的。  
我们设想一下，如果要自己实现一个SpringAOP要解决什么问题？无非就是第一点，
1. 如何通过动态代理，生成代理对象，并返回给用户。
2. 切点和切面，也就是pointcut,aspect怎么实现的。

在看例子之前，还是需要弄懂AOP最基本的几个概念，target, weaving, proxy, aspect, joinpoint, ponitcut, advice。这些东西网上一艘一大把，就不再这里赘述了。但是这几个概念必须要能够理解。

# 来看一个例子
看例子之前说明一下，我这里使用的编程的方式来配置AOP。这其实和用xml配置原理上是一样的，只是用编程的方式，更好分析。和上一节一样，还是定义一个AccountService：
```
// 账户服务接口
public interface AccountService {
    
    // 查询账户
    public void queryAccount();
    
    // 修改账户
    public void updateAccount();
}
```
```
// 账户接口实现类
public class AccountServiceImpl implements AccountService {

    public void queryAccount() {
        System.out.println("querying account...");
    }

    public void updateAccount() {
        System.out.println("updating account...");
    }
}
```
接下来，我们就需要定义一个advice了。这个advice里面，定义的就是横切逻辑。
```
// 在方法执行之前的advice
public class ServiceBeforeAdvice implements MethodBeforeAdvice {
    public void before(Method method, Object[] objects, Object o) throws Throwable {
        System.out.println("Log: accessing account");
    }
}
```
在进一步分析如何把advice织入到代理类中之前，提一下这个advice。其实这个类很明显，就是我们用xml定义时的那个advice。Spring中，主要有如下的Advice：
1. MethodBeforeAdvice: 前置advice，目标方法执行前执行的advice
2. AfterReturningAdvice: 后置adivce，目标方法执行后执行的advice
3. MethodInterceptor: 环绕，目标方法前后执行
4. ThrowsAdvice: 抛出异常，在目标方法抛出异常后执行  

我们有了advice，那我们把这个advice运用到哪个方法呢？也就是，我们如何定义切点呢？有了切点，我们又如何定义切面呢？在Spring中，用advisor来表示切面，为什么不用aspect呢？我也不太清楚，但是advisor中，只能定义一个advice和一个pointcut。我想这大概就是区别把。所以接下来，我们定义一个advisor
```
// 定义一个advisor，让他在queryAccount这个方法前执行
public class ServiceAdvisor extends StaticMethodMatcherPointcutAdvisor {

    public ServiceAdvisor(Advice advice) {
        super(advice);
    }

    public boolean matches(Method method, Class<?> aClass) {
        return "queryAccount".equals(method.getName());
    }
}
```

最后在来看一看测试代码
```

```
那么这么多代码，我们从哪里开始分析呢？首先还是看一下代理对象是如何生成的把。

# 代理对象的生成
从上面的代码很容易看出来，我们的代理对象是由ProxyFactoryBean生成的。所以就好点分析一下这个类。让我们从getObject这个方法看起。整个的调用过程是：
1. ProxyFactoryBean # getObject()
2. ProxyFactoryBean # getSingletonInstance()
3. ProxyFactoryBean # createAopProxy()
4. DefaultAopProxyFactory # createAopProxy()
5. ProxyFactoryBean # getProxy()  

上面第4个方法可以返回一个AopProxy，而这个AopProxy有两个实现类，一个是JdkDynammicAopProxy，一个是Cglib2AopProxy。也就是说，SpringAOP的两种实现方法，就定义在这两个类之中。  
接下来再看getSingletonInstance()这个方法：
```
    private synchronized Object getSingletonInstance() {
        if(this.singletonInstance == null) {
            this.targetSource = this.freshTargetSource();
            if(this.autodetectInterfaces && this.getProxiedInterfaces().length == 0 && !this.isProxyTargetClass()) {
                Class<?> targetClass = this.getTargetClass();
                if(targetClass == null) {
                    throw new FactoryBeanNotInitializedException("Cannot determine target class for proxy");
                }

                // === 1 === //
                this.setInterfaces(ClassUtils.getAllInterfacesForClass(targetClass, this.proxyClassLoader));
            }

            super.setFrozen(this.freezeProxy);
            // === 2 === //
            this.singletonInstance = this.getProxy(this.createAopProxy());
        }

        return this.singletonInstance;
    }
```
源码贴出来，还记得上一节我们手动写JDK动态代理时，要想生成一个代理对象，我们需要什么吗？一个interface，一个InvocationHandler。在注释1处，我们看到了设置interface。然后在注释2处，我们知道createAopProxy会返回一个JdkDynamicAopProxy。那我们的InvocationHandler呢？仔细一看，原来JdkDynamicAopProxy实现了InvocationHandler。那现在啥都有了，代理对象自然也就可以生成了。下面要分析的就是方法执行了。

# 切面切点，方法的执行
前面已经看到是如何生成代理对象的了。那接下来就是方法是如何执行的了。从上一节我们知道，代理对象会调用InvocationHandler的invoke()这个方法。现在我们知道JdkDynamicAopProxy实现了InvocationHandler，那我们去这个类中看看invoke这个方法，这个方法中主要的一段代码如下：
```
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        Object oldProxy = null;
        boolean setProxyContext = false;
        TargetSource targetSource = this.advised.targetSource;
        Class<?> targetClass = null;
        Object target = null;

        .....
        .....
                // === 1 === //
                List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
                
                if(chain.isEmpty()) {
                    Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
                    retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
                } else {
                    // === 2 === //
                    MethodInvocation invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
                    // === 3 === //
                    retVal = invocation.proceed();
                }
        .....
        .....
    }
```
首先看注释1，  
在测试代码中，我们定义了advice，也定义了advisor。但是Spring中并不直接使用他们，Spring会把advice转换成Interceptor，会把advisor封装成InterceptorAndDynamicMethodMatcher。  
也就是说，经过注释1之后，advice变成了interceptor，advisor变成了 InterceptorAndDynamicMethodMatcher。InterceptorAndDynamicMethodMatcher之中包含了interceptor。  
再看注释2，  
因为我们肯定不止仅仅配置一个advisor，所以经过注释1之后，我们得到的是一个list。我们暂且称它为拦截链。得到这个拦截链之后，把它封装到MethodInvocation之中，等待调用。  
最后看注释3，
调用proceed()方法，这个看起来很简单，其实还别叫复杂。所以我们仔细看下proceed这个方法的源码，如下:
```
 public Object proceed() throws Throwable {
        if(this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
            return this.invokeJoinpoint();
        } else {
            Object interceptorOrInterceptionAdvice = this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
            if(interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
                InterceptorAndDynamicMethodMatcher dm = (InterceptorAndDynamicMethodMatcher)interceptorOrInterceptionAdvice;
                
                // === 1 === //
                return dm.methodMatcher.matches(this.method, this.targetClass, this.arguments)?dm.interceptor.invoke(this):this.proceed();
            } else {
                return ((MethodInterceptor)interceptorOrInterceptionAdvice).invoke(this);
            }
        }
    }
```
主要看注释1那个地方，我们开始已经封装了拦截链，这里我们依次查看拦截链中的拦截器。简单的说，如果InterceptorAndDynamicMethodMatcher中封装的advisor符合要求，那么我们调用InterceptorAndDynamicMethodMatcher中Interceptor的invoke()方法，如果不符合要求，再次调用proceed()。  
那么，什么是Interceptor呢？我开始说了，我们配置了Advice，advice会被转换成Interceptor。开始我们有一个MethodBeforeAdvice，所以我们就有一个对应的MethodBeforeAdviceInterceptor，看下源码：
```
public class MethodBeforeAdviceInterceptor implements MethodInterceptor, Serializable {  
  
    private MethodBeforeAdvice advice;  
  
  
    /** 
     * Create a new MethodBeforeAdviceInterceptor for the given advice. 
     * @param advice the MethodBeforeAdvice to wrap 
     */  
    public MethodBeforeAdviceInterceptor(MethodBeforeAdvice advice) {  
        Assert.notNull(advice, "Advice must not be null");  
        this.advice = advice;  
    }  
  
    @Override  
    public Object invoke(MethodInvocation mi) throws Throwable {  
        //在调用方法之前，先执行BeforeAdvice  
        this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis() );  
        return mi.proceed();  
    }  
  
}  
```
可以看到，这个Interceptor之中封装了一个Advice，首先会执行before方法，然后就去执行proceed。所以说，这里就有递归的思想了，通过递归，也就保证了before，after之类的执行顺序。仔细想一想，还真是精妙呀~~~

# 总结
到这里，就基本上清楚了SpringAOP整个执行的大致顺序，当然还有很多细节的地方没能覆盖到。我觉得最难的地方当属于把Advice转换成Intercepor，以及后来通过递归的思想，来保证before，after之类的执行顺序。  
发现有些东西就算理解了，还是很难写出来啊，以后有机会自己实现一遍Spring把。