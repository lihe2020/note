在很多业务场景中，都需要用到任务调度。Spring作为一站式框架，为任务调度提供抽象，即`TaskScheduler`。

## TaskScheduler接口

`TaskScheduler`提供了多个重载：

``` java
public interface TaskScheduler {
    ScheduledFuture schedule(Runnable task, Trigger trigger);
    ScheduledFuture schedule(Runnable task, Date startTime);
    ScheduledFuture scheduleAtFixedRate(Runnable task, Date startTime, long period);
    ScheduledFuture scheduleAtFixedRate(Runnable task, long period);
    ScheduledFuture scheduleWithFixedDelay(Runnable task, Date startTime, long delay);
    ScheduledFuture scheduleWithFixedDelay(Runnable task, long delay);
}
```

### Trigger

`Trigger`定义了任务执行的触发时机。Spring提供了开箱即用的`PeriodicTrigger`实现，和基于Cron表达式的`CronTrigger`实现。


`PeriodicTrigger`接收一个固定间隔、一个可选的初始延迟，和一个决定周期执行的任务是fixed-rate还是fixed-delay的boolean值。由于`TaskScheduler`本身已经提供了fixed-rate和fixed-delay的方法重载，所以应该尽量使用这些重载方法。


`CronTrigger`支持强大的基于Cron表达式的任务调度。例如，以下计划任务只在工作日内每个整点15分钟后运行：

``` java
scheduler.schedule(task, new CronTrigger("0 15 9-17 * * MON-FRI"));
```

### fixed-delay和fixed-rate

fixed-delay与fixed-rate的区别是：

- fixed-delay间隔时间是从上一次执行完后开始计时的
- fixed-rate间隔时间是从上一次开始执行时就开始计时的

## TaskScheduler的实现

`TaskScheduler`提供了多个实现，其中主要的、也是默认的实现是`ThreadPoolTaskScheduler`，它其实是对的`java.util.concurrent.ScheduledThreadPoolExecutor`包装。
`ThreadPoolTaskScheduler`使用线程池线程执行任务调度，线程池大小默认是1。

下面示例是每隔5秒打印一次当前时间：

``` java
ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
scheduler.initialize();

scheduler.scheduleWithFixedDelay(
        () -> System.out.println("now: " + LocalDateTime.now()),
        TimeUnit.SECONDS.toMillis(5));
```


## @Scheduled注解

Spring任务调度支持基于注解的配置。

### 1. 启用基于注解的任务调度

需要给主配置类加上`@EnableScheduling`注解以启用该功能：

``` java
@Configuration
@EnableScheduling
public class MyApp {
}
```

如果需要更细粒度的控制，如配置TaskScheduler类型、设置线程池大小等，则需要实现`SchedulingConfigurer`接口：

``` java
@Configuration
@EnableScheduling
public class MySchedulingConfigurer implements SchedulingConfigurer {

    @Override
    public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setThreadNamePrefix("poolScheduler");
        scheduler.setPoolSize(10);
        scheduler.initialize();

        taskRegistrar.setTaskScheduler(scheduler);
    }
}
```

上面例子确保所有`@Scheduled`调度使用的线程池大小为10。

### 2. 使用@Scheduled

`@Scheduled`应用于Bean方法级别。例如，下面方法每隔5秒执行一次，且间隔是以上一次执行完后开始计时的：

``` java
@Scheduled(fixedDelay=5000)
public void doSomething() {
    // something that should execute periodically
}
```

下面方法每隔5秒执行一次，且间隔是以上一次开始执行时开始计时的：

``` java
@Scheduled(fixedDelay=5000)
public void doSomething() {
    // something that should execute periodically
}
```

下面方法只在工作日执行：

``` java
@Scheduled(cron="*/5 * * * * MON-FRI")
public void doSomething() {
    // something that should execute on weekdays only
}
```

## 遇到的问题

### 1. @Scheduled方法在同一时间被执行2次

如果`@Scheduled`所在的类同时使用`@Configuration`和`@ConfigurationProperties`，那么调度方法在同一时间会被执行2次：

``` java
@Component
@ConfigurationProperties
public class MyScheduleTask {
    @Scheduled(fixedDelay = 5000)
    public void printCurrentDate() {
        //同一时间会执行2次
    }
}
```

通过逐步排除发现是Spring Cloud引起的，遇到问题的版本是Edgware.SR1，升级到最新版后问题消失。如果您的项目不方便升级，那另一个解决办法将`@ConfigurationProperties`移动到其他地方。

### 2. 调度方法执行不及时

如果使用`@Scheduled`配置了多个调度方法，并执行时间较长，那么这些方法的执行间隔似乎和预期不一致。并且通常是前一个调度方法没执行完，其他的调度方法被阻塞。
这是因为`@Scheduled`使用的线程池默认只有1个线程，所以多个不同的任务会相互影响，解决方法就是加大线程池线程数。

## 示例源码

http://gitlab.bitautotech.com/hanrui3/springboot-scheduler-example

## 参考
https://docs.spring.io/autorepo/docs/spring-framework/3.2.x/spring-framework-reference/html/scheduling.html  
https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-external-config.html  
[Spring Scheduler的使用与坑](http://qinghua.github.io/spring-scheduler/)  
[Spring 寻根究底 - Spring 4.x Task 和 Schedule 概述](http://zzy.cincout.cn/2016/09/30/spring-task-and-schedule-deep-research/)  
[@ConfigurationProperties bean created twice when Bootstrapping with Eureka](https://github.com/spring-cloud/spring-cloud-netflix/issues/2186)  
[Spring Boot, Scheduled task, double invocation](https://stackoverflow.com/questions/38163080/spring-boot-scheduled-task-double-invocation)  
[springboot @PostConstruct methods are called twice](https://stackoverflow.com/questions/47908489/springboot-postconstruct-methods-are-called-twice)