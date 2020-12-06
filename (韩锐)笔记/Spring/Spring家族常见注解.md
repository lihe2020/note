### Spring注解

#### 1.1 生命周期相关

- @Configuration
- @Bean
- @Component
- @ComponentScan
- @Scope
- @Lazy
- @Condition
- @Import 快速导入组件到容器中，支持`ImportSelector`、`ImportBeanDefinitionRegistrar`
- @PostConstruct、@PreDestroy

#### 1.2 属性赋值相关

- @Value
- @PropertySource 指定配置文件位置并加载
- @Autowired、@Qualifier 装配bean，使用@Primary指定首选bean
- @Resource(JSR250)、@Inject(JSR330) Java规范定义的装配注解
- @Service（？）

#### 1.3 环境相关配置

- @Profile

#### 1.4 AOP相关注解

- @Aspect
- @EnableAspectJAutoProxy
- @Before、@After 前置、后置通知
- @AfterReturning 返回通知
- @AfterThrowing 异常通知
- @Around 环绕通知

#### 1.5 声明式事务

- @Transactional
- @EnableTransactionManagement

#### 1.6 事件监听

- @EventListener

### SpringMVC注解

- @Controller
- @

### 参考

1. https://www.bilibili.com/video/BV1gW411W7wy

