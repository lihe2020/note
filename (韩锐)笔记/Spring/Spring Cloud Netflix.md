HystrixCircuitBreakerImpl

#### 信号量配置：

- 默认并发信号量：10
- 

#### 熔断配置：

- 请求数达到20以上才会启用熔断逻辑
- 时间窗口5秒，发生熔断5s后会放一小批流量实验，确认流量是否恢复
- 错误率50%，达到这个比例后将进入熔断处理，错误率是通过Metrics的滑动窗口计算的

#### Metrics配置：

- 滑动窗口总时长：10s
- bucket数量：10

对于上述配置，如果滑动窗口总时长是10s，bucket数是10个，那么每秒钟将会产生一个窗口。

#### 优化选项：

- 请求缓存
- 请求合并



