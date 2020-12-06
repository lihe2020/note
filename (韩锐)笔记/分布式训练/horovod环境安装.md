horovod本身安装比较简单，但要使用GPU训练的话，相关的组件较多，步骤较繁琐，此文档用于记录完整安装步骤。

> 安装环境：CentOS7，显卡类型：NVIDIA Tesla。
>
> 软件环境：TensorFlow 1.12.0，g++-4.8.5，Tesla Driver 410.129 ，CUDA 9.1.0，cuDNN 7.1.4，NCCL2，OPENMPI 4.0.2

### 安装显卡驱动

打开[英伟达官网下载页][1]，显卡类型是`NVIDIA Tesla  M40 24GB`，按提示选择对应选项，搜索并下载。

安装显卡驱动有多种方式，推荐从官方源安装。如果安装过旧版，需要先卸载：

```bash
#移除旧版，谨慎
sudo yum remove "*nvidia*" 
```

如果有条件，在卸载完成后建议重启一次，否则可能会遇到各种问题。

#### 方法一：离线安装

```bash
sudo yum install gcc\* glibc\*  glibc-\* kernel kernel-devel kernel-headers
wget http://us.download.nvidia.com/tesla/410.129/NVIDIA-Linux-x86_64-410.129-diagnostic.run
sudo sh NVIDIA-Linux-x86_64-410.129-diagnostic.run
```

如果提示找不到内核头文件，请确认内核版本，内核版本和头文件版本是否一致。我在安装的时候一直失败，最后发现是内核版本虽然更新了，但一直没重启，所以一直按旧版去安装，怎么也安装不成功，需要特别注意这个坑。

#### 方法二：从官方源安装

```bash
wget http://us.download.nvidia.com/tesla/410.129/nvidia-diag-driver-local-repo-rhel7-410.129-1.0-1.x86_64.rpm
rpm -i nvidia-diag-driver-local-repo-rhel7-410.129-1.0-1.x86_64.rpm
yum clean all
#查看所有版本的驱动
sudo yum --enablerepo=elrepo --showduplicates list cuda-drivers
#安装410版驱动
sudo yum install cuda-drivers-410.129-1
```

#### 方法三：从ELRepo源中安装

```bash
sudo yum install epel-*
sudo yum install gcc\* glibc\*  glibc-\*
sudo yum install nvidia-detect
nvidia-detect -v
sudo yum install kmod-nvidia
```

安装完成后，输入`nvidia-smi`查看GPU详情：

```bash
#查看驱动版本号
cat /proc/driver/nvidia/version
#查看驱动信息
nvidia-smi
```

如果没有异常，代表安装成功，[安装文档]( https://docs.nvidia.com/datacenter/tesla/tesla-installation-notes/index.html#unique_818599106 )。如果提示：

```
Failed to initialize NVML: Driver/library version mismatch
```

那么有可能是之前安装过旧版，还没完全卸载，重启系统即可。如果无法重启系统，可以按[这个步骤](https://stackoverflow.com/a/45319156)试试。

### 安装CUDA

CUDA(Compute Unified Device Architecture) 是GPU通用并行计算架构，下载地址： https://developer.nvidia.com/cuda-toolkit-archive ，按条件选择后，得到下载链接：

```shell
#移除旧版
sudo yum remove "*cublas*" "cuda*"
wget http://developer.download.nvidia.com/compute/cuda/repos/rhel7/x86_64/cuda-repo-rhel7-9.1.85-1.x86_64.rpm
sudo rpm -i cuda-repo-rhel7-9.1.85-1.x86_64.rpm
sudo yum install -y cuda-9-0

/usr/local/cuda/bin/nvcc --version
#CUDA Version 9.0.176
```

由于不同版本TensorFlow依赖的CUDA版本也[不一样][3]，所以在安装时务必留意版本。

### 安装cuDNN

```shell
wget https://developer.download.nvidia.com/compute/machine-learning/cudnn/secure/v7.1.4/prod/9.0_20180516/cudnn-9.0-linux-x64-v7.1.tgz?8XhaWa88dcDHyJ99UOySnFpzxJmZCNXUfg5wzlKHM94qZb7rzkccwgsfkF6Q-6_vmwMW03R4o4_W5nVVXdo6p38IFmUruRtlKjAauTYIjIwQHYLOKdpNJwzfwse-q_xVRttWE-uGUfkxrdQT4omOGB1ADciRHc2yRDQ14tdZFCJBiKi2UJMWKw04Tfz4QcM3ewyKj6bIrk-JSw
tar -xzvf cudnn-9.0-linux-x64-v7.tgz
sudo cp cuda/include/cudnn.h /usr/local/cuda/include
sudo cp cuda/lib64/libcudnn* /usr/local/cuda/lib64
sudo chmod a+r /usr/local/cuda/include/cudnn.h
#查看版本号
grep CUDNN_MAJOR -A 2 /usr/local/cuda/include/cudnn.h
```

### 安装NCCL

NCCL(NVIDIA Collective Communications Library) 是GPU集群间的通信框架，它遵循并实现了OPENMPI协议，安装步骤：

```bash
wget https://developer.download.nvidia.com/compute/machine-learning/repos/rhel7/x86_64/nvidia-machine-learning-repo-rhel7-1.0.0-1.x86_64.rpm
sudo rpm -i nvidia-machine-learning-repo-rhel7-1.0.0-1.x86_64.rpm
sudo yum install libnccl-2.5.6-1+cuda9.0 libnccl-devel-2.5.6-1+cuda9.0 libnccl-static-2.5.6-1+cuda9.0
```

需要注意的是，由于NCCL依赖于CUDA，安装NCCL同时会安装响应版本CUDA，所以其实安装CUDA步骤可以省略。

### 安装OPENMPI

执行以下命令：

```shell
wget https://download.open-mpi.org/release/open-mpi/v4.0/openmpi-4.0.2.tar.gz
gunzip -c openmpi-4.0.2.tar.gz | tar xf -
cd openmpi-4.0.2
./configure --prefix=/usr/local
make all install

#查看版本
#sudo ldconfig
mpirun -version
```

由于是从源码安装，所以编译时间较长，需要耐心等待。

### 创建用户

为了避免安装Python包版本冲突，新建一个用户：

```shell
useradd -m -d /data/hanrui3 -g root hanrui3
#修改密码
passwd hanrui3
```

执行`visudo`命令，在最后面加入以下内容，为该用户开启sudo权限：

```shell
hanrui3       ALL=(ALL)     ALL
```

切换到该用户，下面的命令都是在该用户环境下执行：

```shell
su hanrui3
```

### 安装Python依赖并执行

依次在每台服务器上安装：

```shell
#升级pip
pip install --upgrade pip --user -i https://pypi.tuna.tsinghua.edu.cn/simple
#安装tensorflow-gpu
python -m pip install tensorflow-gpu==1.12.0 --no-cache-dir --user -i https://pypi.tuna.tsinghua.edu.cn/simple
#安装horovod
HOROVOD_NCCL_INCLUDE=/usr/include HOROVOD_NCCL_LIB=/usr/lib64 HOROVOD_GPU_ALLREDUCE=NCCL HOROVOD_GPU_BROADCAST=NCCL HOROVOD_WITH_TENSORFLOW=1 python -m pip install horovod --verbose --no-cache-dir --user -i https://pypi.tuna.tsinghua.edu.cn/simple
#检测编译结果
horovodrun --check-build --verbose
```

> Horovod+TensorFlow需要`g++-4.8.5`来编译，所以请确保安装了该版本。

执行命令时提示找不到`horovodrun`，那么需要修改`~/.bashrc`，将相关路径加入`PATH`：

```shell
PATH=/usr/local/bin/:$PATH
PATH=~/.local/bin:$PATH
```

修改完成后，使用以下命令更新环境变量：

```shell
source ~/.bashrc
```

### 执行

#### 单机多卡

```shell
horovodrun -np 2 -H localhost:2 --verbose python tensorflow_mnist.py
```

#### 多机多卡

```shell
ssh-keygen
#在10.163.24.200上执行
ssh-copy-id hanrui3@10.163.6.155
#在10.163.6.155上执行
ssh-copy-id hanrui3@10.163.24.200

horovodrun -np 4 -H 10.163.6.155:2,10.163.24.200:2 --verbose python tensorflow2_mnist.py
```

如果执行出错，可以试试`mpirun`：

```shell
mpirun -np 4 \
    -H 10.163.6.155:2,10.163.24.200:2 \
    -bind-to none -map-by slot \
    -x NCCL_DEBUG=INFO -x LD_LIBRARY_PATH -x PATH \
    -mca pml ob1 -mca btl ^openib \
    python tensorflow_mnist.py
```

### [nvidia-docker](https://github.com/NVIDIA/nvidia-docker)

```shell
curl -s -L https://nvidia.github.io/nvidia-docker/centos7/nvidia-docker.repo | sudo tee /etc/yum.repos.d/nvidia-docker.repo
sudo yum install -y nvidia-container-toolkit
sudo systemctl restart docker

#检查容器是否能正确识别GPU
sudo docker run --gpus 2 nvidia/cuda:9.0-base nvidia-smi

#211.159.164.43
sudo docker run --privileged -v /root/.ssh:/root/.ssh -v /etc/ssh/sshd_config:/etc/ssh/sshd_config --network=host --gpus all -v /data/hanrui3/.keras/datasets:/root/.keras/datasets -it horovod/horovod:0.19.0-tf2.0.0-torch1.3.0-mxnet1.5.0-py3.6-gpu
#在容器里执行
export LD_LIBRARY_PATH="/usr/local/cuda/compat/:$LD_LIBRARY_PATH"
horovodrun -np 2 -H localhost:2 python tensorflow2_mnist.py
#上面的命令等同于
#mpirun -np 2 -bind-to none -map-by slot -x NCCL_DEBUG=INFO -x LD_LIBRARY_PATH -x PATH -mca pml ob1 -mca btl ^openib python tensorflow2_mnist.py

horovodrun -np 4 -H 10.163.6.155:2,10.163.24.200:2 --network-interface eth0 -p 9435 --verbose python tensorflow2_mnist.py
#mpirun -np 4 -H 10.163.6.155:2,10.163.24.200:2 --allow-run-as-root -bind-to none -map-by slot -mca plm_rsh_args "-p 9435" -x NCCL_DEBUG=INFO -x LD_LIBRARY_PATH -x PATH  -x NCCL_SOCKET_IFNAME=^lo,docker0 -mca pml ob1 -mca btl ^openib -mca btl_tcp_if_exclude lo,docker0 python tensorflow2_mnist.py
```

为了保证下载的速度，需要配置镜像加速，修改`/etc/docker/daemon.json`配置：

```json
{
    "registry-mirrors":["https://mirror.ccs.tencentyun.com"],
    "data-root":"/data/docker/data"
}
```

遇到的问题：

1. hostname不合法
2. tmp目录无写入权限
3. gcc版本不一致
4. tf依赖的cuda版本不一致（`undefined symbol: __cudaRegisterFatBinaryEnd`）
5. docker需要privileged权限
6. 内核版本不一致，装驱动错误百出，下载速度慢
7. gcc版本不一致，驱动安装失败
8. 网速不行（kerus、nVidia）

### 运行情况统计

单机CPU：3:19~3:29

单机2GPU：2:44~2:54，10分钟

多机4GPU：12:09~12:12，3分钟



### 参考

[1]: 英伟达驱动下载地址：http://www.nvidia.com/Download/Find.aspx
[2]: 腾讯云GPU驱动安装教程：https://cloud.tencent.com/document/product/560/8048

[3]: TensorFlow 各个组件依赖对照表：https://www.tensorflow.org/install/source?hl=en#gpu

[4]: Horovod官方仓库：https://github.com/horovod/horovod