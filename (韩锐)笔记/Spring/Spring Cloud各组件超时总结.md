本文以Spring Cloud Finchley.RELEASE版本为例。

---

## RestTemplate超时时间

RestTemplate可以通过RestTemplateBuilderl来设置超时时间：

``` java
@Bean
public RestTemplate restTemplate(RestTemplateBuilder restTemplateBuilder) {
    return restTemplateBuilder
            .setConnectTimeout(...)
       .setReadTimeout(...)
       .build();
}
```

## Feign超时时间

文档中没有详细介绍，但部分示例代码中包含了相关配置：

```yaml
feign:
  client:
    config:
      default:
        connectTimeout: 5000 #默认10s
        readTimeout: 5000 #默认60s
        loggerLevel: basic
```

Feign的默认超时时间在[`Request.Options`](https://github.com/OpenFeign/feign/blob/a25423c0233ea1f0b0da8802b60c128133a1e640/core/src/main/java/feign/Request.java#L125)中设置的。

## Ribbon超时时间

``` yaml
#全局超时配置
ribbon:
  ReadTimeout: 1000  #默认1s
  ConnectTimeout: 1000 

#针对具体的服务配置
service-id:
  ribbon:
    ReadTimeout: 1000
    ConnectTimeout: 1000
```

详情[参见文档](https://github.com/Netflix/ribbon/wiki/Programmers-Guide)。

## Hystrix超时时间

``` yaml
hystrix:
  command:
    default:
      execution:
        timeout:
          enabled: true #默认true
        isolation:
          thread:
            timeoutInMilliseconds: 5000 #默认1s
```

Hystrix作为熔断器，通常其他组件一起使用时（如Ribbon）：

``` java
//伪代码调用方式
hystrix(retry(ribbon(http)))
```

此时**需要将超时时间设置成比其他组件的长，否则重试机制将失效**。

## Zuul超时时间

由于Zuul（Spring Cloud）使用了Hystrix和Ribbon，所以它的超时配置是这2个部分的组合。
同时，如果您在Zuul中配置了URL路由，而非通过服务的方式（不经Ribbon），那么还需要配置
`zuul.host.connect-timeout-millis`超时。以下是完整配置：

``` yaml
zuul:
  routes:
    service-id:
      path: /test/**
      serviceId: SERVICE-ID
    admin:
      path: /admin/**
      url: http://test.com/admin
  host: #针对直接发起URL请求的超时配置（不经Ribbon）
    max-total-connections: 5000
    max-per-route-connections: 500

#针对Hystrix的超时配置
hystrix:
  command:
    default:
      execution:
        timeout:
          enabled: true #默认true
        isolation:
          thread:
            timeoutInMilliseconds: 5000 #默认1s

#针对Ribbon的超时配置
ribbon:
  ReadTimeout: 1500  # 1.5s
  ConnectTimeout: 1000 # 1s

```

如果将Hystrix超时配置成2s，而Ribbon配置成5s，那么在每次请求都会看到类似于下面的警告：

```
2019-01-19 17:39:20.981  WARN 36568 --- [io-60362-exec-9] o.s.c.n.z.f.r.s.AbstractRibbonCommand    : The Hystrix timeout of 2000ms for the command SERVICE-ID is set lower than the combination of the Ribbon read and connect timeout, 12000ms.
```

Hystrix的超时时间应当大于Ribbon的超时时间，否则后者的超时时间还没到请求就会被中断

### 参考

[Feign Client 配置](https://xli1224.github.io/2017/09/22/configure-feign/) 
[Spring Cloud各组件超时总结](http://www.itmuch.com/spring-cloud-sum/spring-cloud-timeout/)