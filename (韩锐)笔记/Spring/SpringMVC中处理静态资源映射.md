将`DispatcherServlet`映射到应用程序根路径(`/`)后，所有请求都将会被拦截，这正是我们想要的，但除了静态资源😜。如果不加额外的配置，直接访问静态资源将会出现404。

解决该问题有三种方式：

## 方式一：将静态资源交给默认Servlet处理
默认Servlet(名字叫`default`)是专用于处理静态资源和目录的，详情参见[Tomcat文档](https://tomcat.apache.org/tomcat-7.0-doc/default-servlet.html)。由于更具体的url模式将会覆盖根路径，所以只需将资源路径映射到默认Servlet即可，对应web.xml配置：
``` xml
<servlet-mapping>
    <servlet-name>default</servlet-name>
    <!--对应路径 ~/webapp/resources/* -->
    <url-pattern>/resources/*</url-pattern>
</servlet-mapping>
```
或者在`WebApplicationInitializer`中以编程的方式实现：
``` java
public class DemoStartInitializer
        extends AbstractAnnotationConfigDispatcherServletInitializer {

    @Override
    public void onStartup(ServletContext servletContext) throws ServletException {
        servletContext.getServletRegistration("default").addMapping("/resources/*");
        super.onStartup(servletContext);
    }
}
```

## 方式二：由DispatcherServlet将对静态资源的请求转发到默认Servlet
这种方式只需要启用`DefaultServletHttpRequestHandler`，它会将所有匹配`/**`(这是一个[Ant-style路径](https://stackoverflow.com/questions/2952196/learning-ant-path-style)，相对来说优先级最低)的请求转发给Servlet容器中默认的Servlet上，相应的Spring配置：
``` xml
<mvc:default-servlet-handler />
```
或者以Java配置的方式，例如：
``` java
public class WebConfig extends WebMvcConfigurerAdapter {
    // ...

    @Override
    public  void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer){
        configurer.enable();
    }

}
```

## 方式三：由DispatcherServlet处理静态资源
这种方式是最灵活的，可以将静态资源映射到webapp目录、classpath，也可控制静态资源缓存等。如，将`/resources/**`映射到`/WEB-INF/resources/`，需要如下配置：
``` xml
<mvc:resources mapping="/resources/**" location="/WEB-INF/resources/" />
```
或者以Java配置的方式，例如：
``` java
@Configuration
@EnableWebMvc
public class WebConfig extends WebMvcConfigurerAdapter {

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/resources/**").addResourceLocations("/WEB-INF/resources/");
    }
}
```

详细等介绍参见[官方文档](https://docs.spring.io/spring/docs/current/spring-framework-reference/html/mvc.html#mvc-config-static-resources)