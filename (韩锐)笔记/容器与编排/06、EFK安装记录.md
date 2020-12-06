### 1. 安装

从helm中安装`efk`即可

### 2.配置

#### nginx-ingress配置

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: ingress-kibana
  namespace: cattle-logging
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
  - http:
      paths:
      - path: /kibana
        backend:
          serviceName: efk-kibana
          servicePort: 5601
```

在kibana中配置：<kbd>管理</kbd>，<kbd>远程集群</kbd>，在“种子地址”一栏中输入：

```text
elasticsearch-master.cattle-logging.svc.cluster.local:9300
```

`elasticsearch-master.cattle-logging.svc.cluster.local`表示的是elasticsearch服务的dns名字。

### 3. Elasticsearch相关命令

```bash
#查看节点信息
curl -X GET es-cls:9200/_nodes/process?pretty
```

### 4. 安装fluentd

由于docker安装目录不统一，有的在`/data/docker/data`，而有的在`/data02/docker/data`，所以为了方便卷映射，我（自作聪明）创建了一个链接目录：

```bash
ln -s /data(02)/docker/data /var/lib/docker
```

以为这样就能高枕无忧了，结果后面各种尝试就是收集不上来日志。

最开始我用的是Rancher自带的日志收集功能，它其实是集成了Fluentd并进行了调整，安装非常简单，很快就装完了，但一直报错：

> /var/lib/rancher/rke/log/kube-proxy_feb9aa5da6f2e3d8291a71ce72cd7c7ad2b9d51d7e834a87a4a3bf912ac18f6f.log unreadable. It is excluded and would be examined next time.

提示的是文件没找到，所以我怀疑目录挂载是否正确，查看配置：

```yaml
fluentd: 
  cluster: 
    dockerRoot: "/data/docker/data"
    ...
```

虽然不全是这个目录，但有几个节点是的，但看kibana就是没日志。反复卸载安装了多次也没用，官方也没提供详细的文档描述，问题似乎无从查起！所以我转而使用[fluent-bit](https://hub.helm.sh/charts/stable/fluent-bit)，安装过程也很简单，相关配置描述也非常详细：

这里我使用[fluent-bit](https://hub.helm.sh/charts/stable/fluent-bit)安装：

```yaml
#安装debug版本方便调试
image:
  fluent_bit:
    tag: "1.3.7-debug"
    
backend:
  type: es
  es:
    host: elasticsearch-master.cattle-logging.svc.cluster.local
    port: 9200
    index: gpu-cluster
    type: flb_type
    logstash_prefix: gpu-cluster
extraVolumes:
- name: datalog
  hostPath:
    path: /data/docker/data/
- name: data01log
  hostPath:
    path: /data01/docker/data/
- name: data02log    
  hostPath:
    path: /data02/docker/data/  
extraVolumeMounts:
- mountPath: /data/docker/data/
  name: datalog
- mountPath: /data01/docker/data/
  name: data01log
- mountPath: /data02/docker/data/
  name: data02log
```

安装完成后查看日志，虽然仅寥寥数行，但ES连接正常，似乎一切OK，可是日志同样没收集上来😭，进入容器一探究竟：

> [ info] [storage] initializing...
> [ info] [storage] in-memory
> [ info] [storage] normal synchronization mode, checksum disabled, max_chunks_up=128
> [ info] [engine] started (pid=1)
> [ info] [filter_kube] https=1 host=kubernetes.default.svc port=443
> [ info] [filter_kube] local POD info OK
> [ info] [filter_kube] testing connectivity with API server...
> [ info] [filter_kube] API server connectivity OK
> [ info] [sp] stream processor started

可是日志同样没收集上来😭，进入容器一探究竟：

```bash
kubectl exec -it <container-id> sh -n cattle-logging
```

首先查看并确认各项配置，一切OK；接下来查看目录挂载情况，目录结构也都很完整，一个都没少。

```yaml
extraVolumeMounts 

extraVolumes 
```

### 3. 总结

在安装的过程中最大的问题就是无法访问kibana的dashboard。原因很复杂：

1. 只有备案的域名才能访问腾讯云主机，无论是dns解析还是配置host的方式
2. nginx-ingress只有一个公开的HTTP端口
3. 

如果出现：

> Kibana did not load properly. Check the server output for more information.

请确认是否是通过域名访问的。