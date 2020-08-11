---
title: 依赖注入HttpServletRequest的来源
tags: Java
---

在进行web开发的时候，我们常常需要注入当前请求的上下文，来获取一些特殊的Http请求信息，在SpringMVC或SpringBoot中，我们通常会直接注入HttpServletRequest类。

```java
@RestController
public class WebController {
    
    private final HttpServletRequest request;

    @Autowired
    public WebController(HttpServletRequest request) {
        this.request = request;
    }

    @GetMapping("/hello")
    public String hello() {
        return request.getSession().getId();
    }
}
```

我们使用curl访问这个地址时会发现每次的sessionId都不同，可是我们注入是同一个HttpServletRequest啊，这是怎么回事呢？

```shell
moonto@mac ~ % curl http://localhost:8080/hello
523D6F5A375046204B9F701E36B4C092%   
moonto@mac ~ % curl http://localhost:8080/hello
3D34503F3D7D51944094D5892DD1C130% 
```

<!--more-->

按照我们对Spring的理解，我们的BeanFactory肯定会存在一个HttpServletRequest的Bean，顺着这个脉络去找看一下是怎么回事。

对我们的WebController稍微改造，看看BeanFactory中的Bean。

```java
@RestController
public class WebController implements ApplicationContextAware {

    private final HttpServletRequest request;

    @Autowired
    public WebController(HttpServletRequest request) {
        this.request = request;
    }

    @GetMapping("/hello")
    public String hello() {
        return request.getSession().getId();
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        System.out.println(applicationContext);
    }
}
```

我们在setApplicationContext中设置一个端点，使用Debug启动SpringWeb程序。同时我们使用evaluate执行applicationContext.getBeanFactory()，就能看到BeanFactory的内容。在BeanFactory中可以发现我们的ServletRequest是WebApplicationContextUtils的一个内部类。

![spring-servlet-request-1](https://moonto.org/assets/images/servlet-request/spring-servlet-request-1.png)

跟踪到我们的WebApplicationContextUtils，可以看到RequestFactory其实就是一个ObjectFactory

![spring-request-factory](https://moonto.org/assets/images/servlet-request/spring-request-factory.png)

在这里就可以解开我们的疑问了，为什么注入的是同一个HttpServletRequest，但执行的每次都是不同的对象，因为它是一个ObjectFactory包装的Bean，每次getObject都是从当前上下文取出相关的Request。同样的，这个RequestObjectFactory也是在WebApplicationContextUtils中注册。

![spring-request-bean-register](https://moonto.org/assets/images/servlet-request/spring-request-bean-register.png)

有一个需要注意的地方是它是通过registerResolvableDependency方式注册，因此并不支持beanFactory.getBean方式获取Bean。

