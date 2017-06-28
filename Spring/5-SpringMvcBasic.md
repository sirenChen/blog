# 基本概念
首先我们说，SpringMVC是一个web层的框架。那么作为一个web层的框架，它主要的目的就是三点：
1. 接收客户端传过来的请求参数。
2. 把参数交给服务层进行处理。
3. 把处理过的参数，返回给客户端。

SpringMVC的工作原理以及工作流程，网上很轻松就能搜到一大把，我在这里稍微简述一下：
1. 客户端发出HTTP请求，Web服务器（Tomcat）接收到请求。Web容器把请求交给DispatcherServlet处理。
2. 更具请求信息（主要是路径），由HandlerMapping返回一个Handler给DispatcherServlet。
3. DispatcherServlet得到handler后，通过HandlerAdapter对Handler进行封装。调用此handler，返回一个ModelAndView。
4. 通过ViewResolver对ModelAndView进行解析，得到View试图。最后对这个view进行渲染。

![](SpringMVC.jpg) 

# 入门实例
既然DispatcherServlet是一个servlet，那么想要配置这个servlet自然是去web.xml中了。如下：
```
    <servlet>
        <servlet-name>MyApp</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath:/config/SpringMvcContext.xml</param-value>
        </init-param>
    </servlet>

    <servlet-mapping>
        <servlet-name>MyApp</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>
```