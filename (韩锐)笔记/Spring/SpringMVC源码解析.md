```java
// TomcatServletWebServerFactory.java
protected void configureContext(Context context,
      ServletContextInitializer[] initializers) {
   TomcatStarter starter = new TomcatStarter(initializers);
   if (context instanceof TomcatEmbeddedContext) {
      TomcatEmbeddedContext embeddedContext = (TomcatEmbeddedContext) context;
      embeddedContext.setStarter(starter);
      embeddedContext.setFailCtxIfServletStartFails(true);
   }
   // 添加Servlet容器初始化器
   context.addServletContainerInitializer(starter, NO_CLASSES);
   for (LifecycleListener lifecycleListener : this.contextLifecycleListeners) {
      context.addLifecycleListener(lifecycleListener);
   }
   for (Valve valve : this.contextValves) {
      context.getPipeline().addValve(valve);
   }
   for (ErrorPage errorPage : getErrorPages()) {
      new TomcatErrorPage(errorPage).addToContext(context);
   }
   for (MimeMappings.Mapping mapping : getMimeMappings()) {
      context.addMimeMapping(mapping.getExtension(), mapping.getMimeType());
   }
   configureSession(context);
   new DisableReferenceClearingContextCustomizer().customize(context);
   for (TomcatContextCustomizer customizer : this.tomcatContextCustomizers) {
      customizer.customize(context);
   }
}
```

























首先spring会注册ZuulHandlerMapping来处理我们配置的路由请求

> ZuulHandlerMapping继承自AbstractUrlHandlerMapping，而后者实现了HandlerMapping接口。DispatcherServlet会利用这些HandlerMapping来处理请求。



```
ZuulController
```







Semaphores默认最大值100





















