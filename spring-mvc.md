# Spring MVC的介绍

## 跟踪Spring MVC的请求

当用户发送请求, 请求从代理服务器转发到应用服务器容器如Tomcat,从此开启了Spring MVC之旅. 

首先第一站是DispatcherServlet,你可以把他想象成一个管理并派发请求的门卫,我们都知道HTTP请求在JAVA的web程序中是采用的BIO模型, 并发的请求上来以后会启动多线程分别处理请求,也就是每个servlet会负责一个请求, 那这个门卫呢就是负责将请求指派给某个servlet去处理的家伙. DispatcherServlet会查询一个或者多个处理器映射来确定请求下一站在哪.

我们来详细的讲讲这个DispatcherServlet.在Spring MVC中,servlet的声明被放在了web.xml中.
```
<servlet>
        <servlet-name>projectName</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <load-on-startup>1</load-on-startup>
    </servlet>
```
这个servlet-name很重要, 在默认情况下,DispatcherServlet加载时会从一个基于这个Servlet名字的XML文件中加载Spring应用上下文, 他会尝试加载一个名为projectName-serlvet.xml的文件来加载应用上下文.

```
<context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>
            classpath*:/spring-config/applicationContext-*.xml
        </param-value>
</context-param>
```
你还可以通过context-param来加载其他需要加载的配置文件,一般是service,dao层等.

