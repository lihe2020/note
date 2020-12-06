由于TensorFlow和Horovod依赖的组件较多，安装过程非常繁琐，而基于Docker的方式可以很好的屏蔽这个过程。

### 安装Docker

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
                  docker-engine
                  
#安装依赖
sudo yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2

#安装仓库
sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
    
#执行安装
sudo yum install docker-ce docker-ce-cli containerd.io
```



### 安装[nvidia-docker](https://github.com/NVIDIA/nvidia-docker)

```shell
curl -s -L https://nvidia.github.io/nvidia-docker/centos7/nvidia-docker.repo | sudo tee /etc/yum.repos.d/nvidia-docker.repo
sudo yum install -y nvidia-container-toolkit
sudo systemctl restart docker
```

安装后，验证容器是否能正确识别GPU：

```bash
sudo docker run --gpus 2 nvidia/cuda:9.0-base nvidia-smi
```

此外，为了保证下载的速度，可以需要配置镜像加速，修改`/etc/docker/daemon.json`配置：

```json
{
    "registry-mirrors":["https://mirror.ccs.tencentyun.com"],
    "data-root":"/data/docker/data"
}
```

### 单机多卡训练

Horovod已经提供了诸多版本的镜像，我在这里选择TF2.0.0版本。在`211.159.164.43`上启动容器：

```bash
#211.159.164.43
sudo docker run --privileged \
	-v /root/.ssh:/root/.ssh \
	-v /etc/ssh/sshd_config:/etc/ssh/sshd_config \
	-v /data/hanrui3/.keras/datasets:/root/.keras/datasets \
	--network=host \
	--gpus all \
	-it horovod/horovod:0.19.0-tf2.0.0-torch1.3.0-mxnet1.5.0-py3.6-gpu
```

在容器中输入以下命令开始训练：

```bash
#在容器里执行
export LD_LIBRARY_PATH="/usr/local/cuda/compat/:$LD_LIBRARY_PATH"
horovodrun -np 2 -H localhost:2 python tensorflow2_mnist.py
```

如果运行出错，可以运行等价的mpirun命令来协助排查原因：

```bash
mpirun -np 2 -bind-to none -map-by slot -x NCCL_DEBUG=INFO -x LD_LIBRARY_PATH -x PATH -mca pml ob1 -mca btl ^openib python tensorflow2_mnist.py
```

### 多机多卡训练

多机训练需要提前配置好ssh免密登录，目前我在`211.159.161.20`、`211.159.164.43`已经配置好了。

#### 步骤一：启动worker进程

在`211.159.161.20`启动容器：

```bash
sudo docker run --privileged \
	-v /root/.ssh:/root/.ssh \
	-v /etc/ssh/sshd_config:/etc/ssh/sshd_config \
	-v /data/hanrui3/.keras/datasets:/root/.keras/datasets \
	--network=host \
	--gpus all \
	-it horovod/horovod:0.19.0-tf2.0.0-torch1.3.0-mxnet1.5.0-py3.6-gpu
```

进入容器后，输入以下命令：

```bash
bash -c "/usr/sbin/sshd -p 9435; sleep infinity"
```

开放9435端口，等待接收命令并开始训练。

#### 步骤二：开始训练

在`211.159.164.43`启动容器：

```bash
sudo docker run --privileged -v /root/.ssh:/root/.ssh -v /etc/ssh/sshd_config:/etc/ssh/sshd_config -v /data/hanrui3/.keras/datasets:/root/.keras/datasets --network=host --gpus all,capabilities=utility -it horovod/horovod:0.19.0-tf2.0.0-torch1.3.0-mxnet1.5.0-py3.6-gpu
```

进入容器后，输入以下命令开始训练：

```bash
export LD_LIBRARY_PATH="/usr/local/cuda/compat/:$LD_LIBRARY_PATH"
horovodrun -np 4 -H 10.163.6.155:2,10.163.24.200:2 --network-interface eth0 -p 9435 --verbose python tensorflow2_mnist.py
```

开始训练后，可以通过`nvidia-smi`命令查看这2台服务器的GPU使用情况。

训练大概需要3~5分钟完成。

### 打包镜像

尽管Horovod提供了包含TF各版本的镜像，但我在测试的过程中1.x的版本很多都跑不通，各种莫名错误。考虑到公司内很多的项目都是基于1.x，我编写了Dockerfile，目前在`211.159.164.43`测试通过。文件内容：

```dockerfile
FROM nvidia/cuda:9.0-cudnn7-devel-centos7
CMD ["/bin/bash"]

LABEL MAINTAINER=hanrui<rgshare@qq.com>

#yum加速
ADD CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo
RUN yum makecache
#安装工具
RUN yum install -y wget \
    openssh-client openssh-server \
    perl cpan \
    automake autoconf libtool make \
        && rm -rf /var/cache/yum/*

#安装NCCL
RUN wget https://developer.download.nvidia.com/compute/machine-learning/repos/rhel7/x86_64/nvidia-machine-learning-repo-rhel7-1.0.0-1.x86_64.rpm
RUN rpm -i nvidia-machine-learning-repo-rhel7-1.0.0-1.x86_64.rpm \
        && yum install -y libnccl-2.5.6-1+cuda9.0 libnccl-devel-2.5.6-1+cuda9.0 libnccl-static-2.5.6-1+cuda9.0 \
        && rm -rf nvidia-machine-learning-repo-rhel7-1.0.0-1.x86_64.rpm \
        && rm -rf /var/cache/yum/*

#安装OPENMPI
RUN wget https://download.open-mpi.org/release/open-mpi/v4.0/openmpi-4.0.2.tar.gz \
        && tar zxf openmpi-4.0.2.tar.gz \
        && cd /openmpi-4.0.2 \
        && ./configure --enable-orterun-prefix-by-default --prefix=/usr/local \
        && make -j $(nproc) all \
        && make install \
        && ldconfig \
        && rm -rf /openmpi-4.0.2*
ENV OMPI_MCA_plm_rsh_agent=/bin/false

#设置链接，TF会用到它
RUN ln -s /usr/local/cuda/lib64/stubs/libcuda.so /usr/local/cuda/lib64/stubs/libcuda.so.1 \
    && echo "/usr/local/cuda/lib64/stubs" > /etc/ld.so.conf.d/z-cuda-stubs.conf \
    && ldconfig

#python3
RUN yum install -y python3 python3-devel \
        && ln -sf /usr/bin/python3 /usr/bin/python \
        && ln -sf /usr/bin/pip3.6 /usr/bin/pip \
        && rm -rf /var/cache/yum/*
ADD pip.conf /root/.pip/pip.conf
#安装tf
RUN pip install --no-cache-dir tensorflow-gpu==1.12.0
#RUN LD_DEBUG=libs python -c "import tensorflow as tf;print(tf.test.is_gpu_available())"

#安装horovod
RUN ldconfig /usr/local/cuda/targets/x86_64-linux/lib/stubs && \
    HOROVOD_NCCL_INCLUDE=/usr/include HOROVOD_NCCL_LIB=/usr/lib64 HOROVOD_GPU_ALLREDUCE=NCCL HOROVOD_GPU_BROADCAST=NCCL HOROVOD_WITH_TENSORFLOW=1 \
        pip install --no-cache-dir horovod --verbose && \
    ldconfig

#入口
ADD tensorflow_mnist.py /examples/tensorflow_mnist.py
WORKDIR  /examples
```

构建和运行：

```bash
cd /data/hanrui3/files
#构建
sudo docker build -t test:v001 .
#启动容器
sudo docker run -v /data/hanrui3/.keras/datasets:/root/.keras/datasets \
	--privileged --gpus all \
	-it test:v001 bash
#运行
horovodrun -np 2 -H localhost:2 python tensorflow_mnist.py
```

