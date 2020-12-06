Service的官方定义为：将运行在一组Pods上的应用程序公开为网络服务的抽象方法。Service能够提供负载均衡的能力，但只能提供4层负载均衡能力。

### 1. Service的类型

Service在k8s中有以下四种类型：

- ClusterIp 自动分配一个仅Cluster内部可访问的IP
- NodePort 在ClusterIP基础上为Service在每个Node上绑定一个端口
- LoadBalancer 在NodePort基础上，借助cloud provider创建一个外部负载均衡器，将请求转发到NodePort上
- ExternalName 把集群外部的服务引入到集群内部来，在集群内部直接使用。没有任何类型代理被创建
- Ingress

### 2. VIP和Service代理

在k8s集群中，每个Node运行一个`kube-proxy`进程，它负责为Service实现了一种VIP的形式，而不是ExternalName的形式。

！为何不适用round-robin DNS？

答：因为有DNS缓存

