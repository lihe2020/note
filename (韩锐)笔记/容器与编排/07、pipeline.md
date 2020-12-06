使用docker启动gitlab：

```bash
sudo docker run --detach \
  --hostname gitlab.ailab.io \
  --publish 8925:22 \
  --publish 8929:8929 \
  --env GITLAB_OMNIBUS_CONFIG="external_url 'http://211.159.161.20:8929/';gitlab_rails['gitlab_shell_ssh_port'] = 8925;" \
  --name gitlab \
  --restart always \
  --volume /data/gitlab/config:/etc/gitlab \
  --volume /data/gitlab/logs:/var/log/gitlab \
  --volume /data/gitlab/data:/var/opt/gitlab \
  gitlab/gitlab-ce:12.4.7-ce.0
```

如果需要修改gitlab配置：

```bash
#进入容器
sudo docker exec -it gitlab /bin/bash
#修改配置
vim /etc/gitlab/gitlab.rb
#重新载入配置
gitlab-ctl reconfigure
```

```bash
#登录到私有仓库
docker login registry.yiche-cloud.com --username admin --password yiche
#构建镜像
docker build -t horovod:0.1-tf1.12 .
#镜像打标签
docker tag horovod:0.1-tf1.12 registry.yiche-cloud.com/base/horovod:0.1-tf1.12
#推送到仓库
docker push registry.yiche-cloud.com/base/horovod:0.1-tf1.12
```

安装完成后访问：

http://211.159.161.20:8929/

需要修改root账号密码为：yiche@123，修改成功后用`root`账号登录

### 遇到的问题

1. 一直报错：

   > java.net.UnknownHostException: kubernetes.default: Name or service not known
   > 	at java.net.Inet4AddressImpl.lookupAllHostAddr(Native Method)
   > 	at java.net.InetAddress$2.lookupAllHostAddr(InetAddress.java:929)
   > 	at java.net.InetAddress.getAddressesFromNameService(InetAddress.java:1324)
   > 	at java.net.InetAddress.getAllByName0(InetAddress.java:1277)
   > 	at java.net.InetAddress.getAllByName(InetAddress.java:1193)
   > 	at java.net.InetAddress.getAllByName(InetAddress.java:1127)
   > 	at okhttp3.Dns$1.lookup(Dns.java:40)

   似乎和DNS有关

   ```bash
   #进入busybox
   kubectl run busybox --tty busybox --image=busybox:1.28 -- sh
   #容器内执行，提示超时
   nslookup kubernetes
   ```


### 参考

1. https://docs.gitlab.com/omnibus/docker/