## 基础概念
- 并发(Concurrent) 处理请求A的同时响应请求B，这些任务有可能会交替执行
- 并行(Parallel) 让多个CPU同时处理多个任务，这些任务同时执行
- 竞态(Race Condition) 指计算的正确性依赖于相对时间顺序或者线程的交错
   1. read-modify-write
   2. check-then-act
- 线程安全(Thread safety) 
- 原子性(Atomicity) 要么执行已经结束，要么尚未发生，其他线程访问不到中间状态


## 参考
- https://tech.meituan.com/2018/11/15/java-lock.html