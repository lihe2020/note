### 1. 安装Rancher Server

```bash
sudo docker run -d --restart=unless-stopped -p 9090:443 rancher/rancher:v2.3.5
```

在部署成功后，访问UI完成接下来的安装配置。以下是相应的yaml配置：

```yaml
# 
# Cluster Config
# 
answers: {}
docker_root_dir: /data/docker/data
enable_cluster_alerting: true
enable_cluster_monitoring: true
enable_network_policy: false
local_cluster_auth_endpoint:
  enabled: true
name: gpu-cluster
# 
# Rancher Config
# 
rancher_kubernetes_engine_config:
  addon_job_timeout: 30
  authentication:
    strategy: x509|webhook
  authorization: {}
  bastion_host:
    ssh_agent_auth: false
  cloud_provider: {}
  ignore_docker_version: true
  ingress:
    provider: nginx
    node_selector:
      # 给节点打上该标签后才能激活
      app: ingress
  kubernetes_version: v1.17.3-rancher1-1
  monitoring:
    provider: metrics-server
  network:
    mtu: 0
    options:
      flannel_backend_type: vxlan
    plugin: canal
  restore:
    restore: false
  services:
    etcd:
      backup_config:
        enabled: true
        interval_hours: 3
        retention: 10
        safe_timestamp: false
      creation: 12h
      extra_args:
        election-timeout: '5000'
        heartbeat-interval: '500'
      gid: 0
      retention: 72h
      snapshot: false
      uid: 0
    kube-api:
      always_pull_images: false
      pod_security_policy: false
      service_node_port_range: 30000-32767
    kube-controller: {}
    kubelet:
      fail_swap_on: false
      generate_serving_certificate: false
    kubeproxy: {}
    scheduler: {}
  ssh_agent_auth: false
```

### 2. 删除Rancher节点

1. 从Rancher集群中删除节点（通过UI操作）

2. 移除容器，**注意：会删除节点所有容器**

   ```bash
   docker rm -f $(docker ps -qa)
   docker rmi -f $(docker images -q)
   docker volume rm $(docker volume ls -q)
   ```

   > 当前rancher server部署在`211.159.161.20`，操作需谨慎。

3. 取消目录挂载

   ```bash
   for mount in $(mount | grep tmpfs | grep '/var/lib/kubelet' | awk '{ print $3 }') /var/lib/kubelet /var/lib/rancher; do umount $mount; done
   ```

4. 目录移除

   ```bash
   rm -rf /etc/ceph \
          /etc/cni \
          /etc/kubernetes \
          /opt/cni \
          /opt/rke \
          /run/secrets/kubernetes.io \
          /run/calico \
          /run/flannel \
          /var/lib/calico \
          /var/lib/etcd \
          /var/lib/cni \
          /var/lib/kubelet \
          /var/lib/rancher/rke/log \
          /var/log/containers \
          /var/log/pods \
          /var/run/calico
   ```

5. 网络移除

   ```bash
   #显示所有网卡
   ip address show
   #移除flannel.1相关网络
   ip link delete <interface_name>
   ```

### 3. 备份Rancher Server

在Rancher Server服务器（`211.159.161.20`）执行以下命令，其中`nifty_agnesi`是容器名称：

```bash
#停止rancher server
docker stop nifty_agnesi

#从停止的rancher server创建data容器
docker create \
  --volumes-from nifty_agnesi \
  --name "rancher-data-$(date '+%Y-%m-%d')" rancher/rancher:v2.3.5

#创建备份
docker run \
  --volumes-from "rancher-data-$(date '+%Y-%m-%d')" \
  -v $PWD:/backup:z \
  busybox tar pzcvf "/backup/rancher-data-backup-v2.3.5-$(date '+%Y-%m-%d').tar.gz" \
  /var/lib/rancher

#删除临时容器
docker rm "rancher-data-$(date '+%Y-%m-%d')"

#启动rancher server
docker start nifty_agnesi
```

### 4. 恢复Rancher Server

在Rancher Server服务器执行以下命令：

```bash
#停止rancher server
docker stop nifty_agnesi

#删除当前rancher server数据，并用备份替换
docker run  --volumes-from nifty_agnesi -v $PWD:/backup \
busybox sh -c "rm /var/lib/rancher/* -rf  && \
tar pzxvf /data/rancher-data-backup-v2.3.5-2020-03-30.tar.gz"

#启动rancher server
docker start nifty_agnesi
```

### 5. 参考

1. [Install Rancher](https://rancher.com/docs/rancher/v2.x/en/quick-start-guide/deployment/quickstart-manual-setup/#2-install-rancher)
2. [Removing Kubernetes Components from Nodes](https://rancher.com/docs/rancher/v2.x/en/cluster-admin/cleaning-cluster-nodes/#cleaning-a-node-manually)
3. [Creating Backups for Rancher Installed with Docker](https://rancher.com/docs/rancher/v2.x/en/backups/backups/single-node-backups/)
4. [Restoring Backups—Docker Installs](https://rancher.com/docs/rancher/v2.x/en/backups/restorations/single-node-restoration/)