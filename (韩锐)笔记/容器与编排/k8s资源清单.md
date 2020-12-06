### 1. 资源类型

k8s中所有的内容都抽象为资源，资源实例化之后，叫做对象。资源类型包括：

- 工作负载型资源（workload）：Pod、ReplicaSet、Deployment、StatefulSet、DaemonSet、Job、CronJob
- 服务发现及负载均衡型资源（ServiceDiscovery LoadBalance）：Service、Ingress
- 配置与存储型资源：Volume（存储卷）、CSI（容器存储接口）
- 特殊类型的存储卷：ConfigMap、Secret、Downward API
- 集群级资源：Namespace、Node、Role、ClusterRole、RoleBinding、ClusterRoleBinding
- 元数据型资源：HPA、PodTemplate、LimitRange

### Pod容器概述

#### 2.1 Init容器

#### 探针

探针是由kubelet对容器执行的定期诊断。要执行诊断，kubelet调用由容器实现的Handler。有3中类型的处理程序：

1. ExecAction：在容器内执行指定命令，如果命令退出码为0则认为诊断成功
2. TCPSocketAction：检测指定端口，如果端口打开，则认为诊断成功
3. HTTPGetAction：请求指定接口，根据状态码判断是否诊断为成功

每次探测都将获得以下三种状态之一：

1. 成功
2. 失败
3. 未知：认为诊断失败，因此不会采取任何行动

探测方式：

1. livenessProbe：存活探测，指示容器是否正在运行
2. readinessProbe：就绪探测，探测容器是否准备好服务请求

#### 2.3 Pod控制器

##### 2.3.1 控制器类型

- ReplicaSet  用于确保副本数始终保持在预定值，容器有异常退出时，会自动创建新的pod代替
- Deployment 为Pod和ReplicaSet提供声明式定义方法，替代ReplicationController
- DaemonSet 在每个Node上运行**一个**Pod副本。典型应用场景包括Node日志收集、监控等
- StateFulSet 解决有状态服务的问题(Deployment, ReplicaSet是为无状态设计的)
- Job/CronJob 负责批处理任务，仅执行一次或基于cron的定期执行
- Horizontal Pod Autoscaling 适用于deployment和replicaset的弹性扩展

网络通信模式
    •    pod内多个容器：lo
    •    各pod间通信：Overlay Network（分为同一个节点和不同节点）
    •    Pod与Service之间：各节点的iptables规则