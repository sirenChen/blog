# （三）aop的基础，动态代理
什么是aop以及aop的基本概念我就不在这里啰嗦了，网上一搜一大把。稍微记录一下动态代理。对于Spring来说，动态代理有两种实现方式，JDK动态代理和CGLIB动态代理。这里主要是以JDK动态代理为主进行分析。还是直接看例子吧。

# 静态代理
要想说清楚什么是动态代理，先看一个静态代理的例子。首先定义一个接口，规定一个账户服务，需要实现两种方法，如下：

```
// 账户服务接口
public interface AccountService {
    
    // 查询账户
    public void queryAccount();
    
    // 修改账户
    public void updateAccount();
}
```

接着，定义一个实现类
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
如果现在我想在不修改实现类源代码的情况下，改变访问账户的行为，应该怎么办呢？我可以新定义一个实现类，具体实现如下：
```
public class AccountServiceProxy implements AccountService {
    private AccountServiceImpl originService;
    
    // 传入原本的实现类
    public AccountServiceProxy(AccountServiceImpl originService) {
        this.originService = originService;
    }
    
    // 在执行原本的方法之前已经之后，执行写入log的操作
    public void queryAccount() {
        System.out.println("Log: accessing account");
        originService.queryAccount();
        System.out.println("Log: accessing account done");
    }

    // 在执行原本的方法之前已经之后，执行写入log的操作
    public void updateAccount() {
        System.out.println("Log: accessing account");
        originService.updateAccount();
        System.out.println("Log: accessing account done");
    }
}
```
从上面的代码我们可以看到，我们新定义了一个代理类，这个代理类在执行原本的操作之前和之后，会新添加一个写入log的操作。原本的代码也没有被修改。具体使用情况如下：
```
public class Demo {
    public static void main(String[] args) {
        AccountServiceImpl serviceImpl = new AccountServiceImpl();
        AccountServiceProxy serviceProxy = new AccountServiceProxy(serviceImpl);
        
        serviceProxy.queryAccount();
        serviceProxy.updateAccount();
    }
}
```
以上就是静态代理的逻辑。这种方式在不修改源代码的情况下，对原本的操作逻辑进行了修改。但是，这种方式还是存在很多问题的。  
首先就是，如果我们有许多service类，就需要定义很多的ServiceProxy，而且这些代理类的主要作用都是添加写入log的这一种工作，这样就会出现大量的重复代码。第二就是我们可以看到，在代理类中，业务逻辑和写入log混编在了一个。那么现在我们有没有办法能把log操作给抽取出来呢？这就是接下来的动态代理了。

# 动态代理
前面说了，动态代理有两种实现，JDK和CGLIB。这里我就总结一下JDK的动态代理。还是和开始一样，定义service接口和service实现类：
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
接下来，我就不自己编写代理类了，我让jdk自动的动态的生成一个代理类给我。为了让jdk知道，在代理类中，我想如何调用原方法，以及增加何种方法，我需要实现一个接口，如下：
```
//定义一个类，实现InvocationHandler
public class ProxyInvocationHandler implements InvocationHandler {
    private Object target;

    public ProxyInvocationHandler(Object target) {
        this.target = target;
    }

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 调用前写入log
        System.out.println("Log: accessing account");
        // 调用原方法
        method.invoke(target, args);
        //调用后写入log
        System.out.println("Log: accessing account done");

        return null;
    }
}
```
这样子，在访问原来的对象时，jdk会自动生成一个代理类，并执行invoke这个方法。具体情况如下：
```
public class Demo {
    public static void main(String[] args) {
        // 原对象， 也就是需要被代理的对象
        AccountService originService = new AccountServiceImpl();
        
        // handler
        InvocationHandler handler = new ProxyInvocationHandler(originService);

        // 获得代理对象
        AccountService serviceProxy = (AccountService) Proxy.newProxyInstance(originService.getClass().getClassLoader(), originService.getClass().getInterfaces(), handler);
        
        // 执行方法
        serviceProxy.queryAccount();
        serviceProxy.updateAccount();
    }
}
```

# 总结
稍微总结一下动态代理，为下面的Spring AOP原理做准备。使用JDK动态代理，java会自动帮我们生成一个代理类，我们就不需要手工编写这个代理类呢。这个代理类是根据什么东西生成的呢？就是我们自己定义的InvocationHandler。所以，当你拿到了代理类，通过代理类调用方法，最终还是会调用InvocationHandler中的invoke方法。