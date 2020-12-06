### 1. 事务传播行为

spring特有的事务传播行为，spring支持7种事务传播行为：

- PROPAGATION_REQUIRED 当前方法必须在事务中执行，如果没有事务会创建一个，否则使用进行中的事务
- PROPAGATION_SUPPORTS 当前方法不一定得在事务中执行，但是如果有进行中的事务那么将会使用该事务
- PROPAGATION_MANDATORY 当前方法必须在事务中执行，如果没有事务直接抛出异常
- PROPAGATION_NESTED 如果有事务则创建一个嵌套事务，否则创建一个事务
- PROPAGATION_NEVER 当前方法不能在事务中运行，如果有事务直接抛出异常
- PROPAGATION_REQUIRES_NEW 当前方法必须在自己的事务中运行。正在运行中的其他事务将会被挂起
- PROPAGATION_NOT_SUPPORTED 当前方法不能在事务中运行，正在运行中的其他事务将会被挂起

### 2. 使用