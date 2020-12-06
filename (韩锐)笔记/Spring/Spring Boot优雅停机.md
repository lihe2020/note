Spring Boot优雅停机
===

在微服务架构中，应用程序的更新、重启导致访问异常是不可接受的。

在当前Spring Boot版本（2.0.x）中，应用停止时，所有正在处理的连接将被强行中断，从而导致正在连接的的客户端出现连接被重置异常：

```
2018-11-26 17:12:55.969 ERROR 20722 --- [nio-8080-exec-7] o.a.c.c.C.[.[.[/].[dispatcherServlet]    : Servlet.service() for servlet [dispatcherServlet] in context with path [] threw exception [Request processing failed; nested exception is org.springframework.web.client.ResourceAccessException: I/O error on GET request for "http://127.0.0.1:8670/": Connection reset; nested exception is java.net.SocketException: Connection reset] with root cause

java.net.SocketException: Connection reset
	at java.net.SocketInputStream.read(SocketInputStream.java:210) ~[na:1.8.0_172]
	at java.net.SocketInputStream.read(SocketInputStream.java:141) ~[na:1.8.0_172]
	at java.io.BufferedInputStream.fill(BufferedInputStream.java:246) ~[na:1.8.0_172]
	at java.io.BufferedInputStream.read1(BufferedInputStream.java:286) ~[na:1.8.0_172]
	at java.io.BufferedInputStream.read(BufferedInputStream.java:345) ~[na:1.8.0_172]
	at sun.net.www.http.HttpClient.parseHTTPHeader(HttpClient.java:735) ~[na:1.8.0_172]
	at sun.net.www.http.HttpClient.parseHTTP(HttpClient.java:678) ~[na:1.8.0_172]
	at sun.net.www.http.HttpClient.parseHTTPHeader(HttpClient.java:848) ~[na:1.8.0_172]
	at sun.net.www.http.HttpClient.parseHTTP(HttpClient.java:678) ~[na:1.8.0_172]
	at sun.net.www.protocol.http.HttpURLConnection.getInputStream0(HttpURLConnection.java:1587) ~[na:1.8.0_172]
	at sun.net.www.protocol.http.HttpURLConnection.getInputStream(HttpURLConnection.java:1492) ~[na:1.8.0_172]
	at java.net.HttpURLConnection.getResponseCode(HttpURLConnection.java:480) ~[na:1.8.0_172]
...
```

官方成员回应修复这个问题可能会导致副作用，详情[参见这个issue](https://github.com/spring-projects/spring-boot/issues/4657)。同时，他表示开发者可以自行解决这个问题，并给出了一段可用于生产环境的示例代码。

``` java
@Configuration
public class GracefulShutdownConfig {
    @Bean
    public GracefulShutdown gracefulShutdown() {
        return new GracefulShutdown();
    }

    @Bean
    public WebServerFactoryCustomizer tomcatCustomizer() {
        return (WebServerFactoryCustomizer<ConfigurableServletWebServerFactory>) container -> {
            if (container instanceof TomcatServletWebServerFactory) {
                ((TomcatServletWebServerFactory) container)
                        .addConnectorCustomizers(gracefulShutdown());
            }
        };
    }

    private static class GracefulShutdown implements TomcatConnectorCustomizer,
            ApplicationListener<ContextClosedEvent> {

        private static final Logger log = LoggerFactory.getLogger(GracefulShutdown.class);

        private volatile Connector connector;

        @Override
        public void customize(Connector connector) {
            this.connector = connector;
        }

        @Override
        public void onApplicationEvent(ContextClosedEvent event) {
            this.connector.pause();
            Executor executor = this.connector.getProtocolHandler().getExecutor();
            if (executor instanceof ThreadPoolExecutor) {
                try {
                    ThreadPoolExecutor threadPoolExecutor = (ThreadPoolExecutor) executor;
                    threadPoolExecutor.shutdown();
                    if (!threadPoolExecutor.awaitTermination(30, TimeUnit.SECONDS)) {
                        log.warn("Tomcat thread pool did not shut down gracefully within "
                                + "30 seconds. Proceeding with forceful shutdown");
                    }
                } catch (InterruptedException ex) {
                    Thread.currentThread().interrupt();
                }
            }
        }
    }
}
```

（稍作调整，因为示例代码针对的是1.x.x版本，部分对象不可用）  

这段代码的要点是，在应用停止时，暂停Tomcat连接器，拒绝新的连接请求，等待线程池关闭后才能继续销毁应用程序上下文。

## 参考
https://github.com/spring-projects/spring-boot/issues/4657
https://github.com/gesellix/graceful-shutdown-spring-boot
https://www.gesellix.net/post/zero-downtime-deployment-with-docker-stack-and-spring-boot/
http://blog.marcosbarbero.com/graceful-shutdown-spring-boot-apps/