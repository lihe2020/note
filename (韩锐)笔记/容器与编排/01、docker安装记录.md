### 1. 安装Docker

安装docker官方教程：https://docs.docker.com/install/linux/docker-ce/centos/

```bash
#卸载旧版
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine \
                  docker-ce docker-ce-cli containerd.io
                  
#安装依赖
sudo yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2

#安装仓库
sudo wget -O /etc/yum.repos.d/docker-ce.repo https://download.docker.com/linux/centos/docker-ce.repo
sudo sed -i 's+download.docker.com+mirrors.tuna.tsinghua.edu.cn/docker-ce+' /etc/yum.repos.d/docker-ce.repo

#查看所有版本
sudo yum list --showduplicates docker-ce
#执行指定安装
sudo yum install docker-ce-18.09.7 docker-ce-cli-18.09.7
```

### 2. 安装NVIDIA-Docker

移除旧版

```bash
docker volume ls -q -f driver=nvidia-docker | xargs -r -I{} -n1 docker ps -q -a -f volume={} | xargs -r docker rm -f
sudo yum remove nvidia-docker nvidia-docker2 nvidia-container-toolkit libnvidia-container*
```

安装

```bash
#安装仓库
curl -s -L https://nvidia.github.io/nvidia-docker/centos7/nvidia-docker.repo | sudo tee /etc/yum.repos.d/nvidia-docker.repo
#安装
sudo yum install nvidia-docker2
sudo pkill -SIGHUP dockerd
#重启docker
sudo systemctl restart docker
```

### 3. 配置镜像加速

修改`/etc/docker/daemon.json`配置，添加镜像加速地址，以及调整默认运行时：

```bash
{
    "registry-mirrors":["https://mirror.ccs.tencentyun.com"],
    "data-root":"/data/docker/data",
    "default-runtime": "nvidia",
    "runtimes": {
        "nvidia": {
            "path": "/usr/bin/nvidia-container-runtime",
            "runtimeArgs": []
        }
    }
}
```

调整完成后重启docker。
