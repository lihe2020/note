1. 按照metadata查找资源
2. 

## Pod类

```bash
#按照pod状态查找
kubectl get pods --field-selector status.phase=Failed -n kube-system
```



## 节点类

```bash
kubectl get nodes
```

## CRD

```bash
#获取所有的crd资源
kubectl get crd --all-namespaces
#获取所有prometheus类型的资源
kubectl get prometheus --all-namespaces
#查看资源详情
kubectl describe prometheus cluster-monitoring -n cattle-prometheus
#查看yaml格式资源
kubectl get prometheus cluster-monitoring -n cattle-prometheus -o yaml
```

## 日志类

1. 查看审计日志：

   ```bash
   kubectl get events --sort-by=.metadata.creationTimestamp -n default
   ```

   

2. 22
