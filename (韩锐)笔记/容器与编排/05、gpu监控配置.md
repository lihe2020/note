本文档记录了安装gpu监控过程。

在安装rancher时勾选“监控指标”后会自动安装prometheus。虽然安装很简单，但心中有些疑问，在此记录。

- prometheus-operator
- kube-prometheus

### 1. 启动GPU支持

```bash
kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/1.0.0-beta4/nvidia-device-plugin.yml
```

### 2. 部署dcgm-exporter

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  namespace: cattle-prometheus
  name: dcgm-exporter
spec:
  selector:
    matchLabels:
      app: dcgm-exporter
  template:
    metadata:
      labels:
        app: dcgm-exporter
      name: dcgm-exporter
    spec:
      nodeSelector:
        hardware-type: NVIDIAGPU
      containers:
      - image: quay.io/prometheus/node-exporter:v0.16.0
        name: node-exporter
        args:
        - "--web.listen-address=0.0.0.0:9101"
        - "--path.procfs=/host/proc"
        - "--path.sysfs=/host/sys"
        - "--collector.textfile.directory=/run/prometheus"
        - "--no-collector.arp"
        - "--no-collector.bcache"
        - "--no-collector.bonding"
        - "--no-collector.conntrack"
        - "--no-collector.cpu"
        - "--no-collector.diskstats"
        - "--no-collector.edac"
        - "--no-collector.entropy"
        - "--no-collector.filefd"
        - "--no-collector.filesystem"
        - "--no-collector.hwmon"
        - "--no-collector.infiniband"
        - "--no-collector.ipvs"
        - "--no-collector.loadavg"
        - "--no-collector.mdadm"
        - "--no-collector.meminfo"
        - "--no-collector.netdev"
        - "--no-collector.netstat"
        - "--no-collector.nfs"
        - "--no-collector.nfsd"
        - "--no-collector.sockstat"
        - "--no-collector.stat"
        - "--no-collector.time"
        - "--no-collector.timex"
        - "--no-collector.uname"
        - "--no-collector.vmstat"
        - "--no-collector.wifi"
        - "--no-collector.xfs"
        - "--no-collector.zfs"
        ports:
        - name: metrics
          containerPort: 9101
          hostPort: 9101
        resources:
          requests:
            memory: 30Mi
            cpu: 100m
          limits:
            memory: 50Mi
            cpu: 200m
        volumeMounts:
        - name: proc
          readOnly:  true
          mountPath: /host/proc
        - name: sys
          readOnly: true
          mountPath: /host/sys
        - name: collector-textfiles
          readOnly: true
          mountPath: /run/prometheus
      - image: nvidia/dcgm-exporter:1.0.0-beta
        name: nvidia-dcgm-exporter
        securityContext:
          runAsNonRoot: false
          runAsUser: 0
        volumeMounts:
        - name: collector-textfiles
          mountPath: /run/prometheus

      hostNetwork: false
      hostPID: true

      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
      - name: collector-textfiles
        emptyDir:
          medium: Memory
```

### 3. 配置Prometheus

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: expose-gpu-metrics
  namespace: cattle-prometheus
  labels:
    app: expose-gpu-metrics
spec:
  type: ClusterIP
  clusterIP: None
  ports:
  - name: http-metrics
    port: 9101
  selector:
    app: dcgm-exporter
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: exporter-node-gpu
  namespace: cattle-prometheus
  labels:
    app: exporter-node-gpu
    release: cluster-monitoring
spec:
  selector:
    matchLabels:
      app: expose-gpu-metrics
  jobLabel: expose-gpu-metrics-monitor
  endpoints:
  - port: http-metrics
    path: /metrics
    scheme: http
```

### 4. 安装Grafana UI

开箱即用：https://grafana.com/grafana/dashboards/11578

### 5. 参考

1. https://github.com/coreos/prometheus-operator/blob/master/Documentation/user-guides/getting-started.md
2. https://github.com/coreos/prometheus-operator/blob/master/Documentation/api.md