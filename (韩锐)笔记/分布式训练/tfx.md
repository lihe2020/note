第一个worker：

```bat
TF_CONFIG='{"cluster": {"worker": ["localhost:9761", "localhost:9762"]}, "task": {"index": 0, "type": "worker"}}' python worker.py


TF_CONFIG='{"cluster": {"worker": ["10.163.6.155:9761", "10.163.24.200:9761"]}, "task": {"index": 0, "type": "worker"}}' python worker.py
```

第二个worker

```bat
TF_CONFIG='{"cluster": {"worker": ["10.163.6.155:9761", "10.163.24.200:9761"]}, "task": {"index": 1, "type": "worker"}}' python worker.py
```

```bash
sudo docker run --privileged -v /root/.ssh:/root/.ssh -v /etc/ssh/sshd_config:/etc/ssh/sshd_config -v /data/hanrui3:/data/hanrui3  --network=host --gpus all,capabilities=utility -v /data/hanrui3/.keras/datasets:/root/.keras/datasets -it horovod/horovod:0.19.0-tf2.0.0-torch1.3.0-mxnet1.5.0-py3.6-gpu
```

horovod

docker+horovod

tensorflow distributed

主要的问题是资源分配，代码部署（docker）

资源分配如何解决？

代码部署可以使用docker方式，如此需要部署docker私有镜像仓库



docker方式：

1. 打包docker镜像
2. 运行镜像（worker节点），开放ssh端口（这些worker之间需要相互免密登录），等待请求
3. 开始训练







这段时间，我分别试用了3种方式进行分布式训练：

1. 直接在服务器上部署horovod
2. 使用docker部署horovod
3. 使用tensorflow本身分布式



因为tensorflow和horovod需要依赖很多东西：显卡驱动、cuda套件、cudnn、nccl、openmpi，还有内核、gcc版本等，每个环节都有可能是问题，所以直接在服务器上部署非常繁琐、耗时，搞不好好反复的安装和卸载。



tensorflow自身的分布式在1.x和2.x不一样，我跑了一个2.0的demo，涉及到的概念比较多，cluster、server、job、task等，感觉没horovod方便。



docker这种方式，只需要在服务器上安装显卡驱动，然后配置一下ssh就行了。具体来说有这些步骤，以为3台服务器为例：

1. 在其中2台上启动并进入相应的容器，horovod打包的镜像，并且开放一个ssh端口
2. 在第三台服务器启动容器，执行horovod命令开始训练



docker方式问题：

1. docker的和gpu驱动一一对应问题
2. 部署问题









环境管理+机器管理

底层环境

资源分配

环境是否能同一？

docker+宿主机部署是否都支持？

用不用docker对现有技术方案是否有影响？

docker解决了什么问题？

如果用docker最终的技术方案是什么？













LIBRARY_PATH=/usr/local/cuda/lib64/stubs

```
export LD_LIBRARY_PATH="$LD_LIBRARY_PATH:/usr/local/cuda/lib64/stubs"
```

LD_LIBRARY_PATH=/usr/local/nvidia/lib:/usr/local/nvidia/lib64                                                                                                              

/bin/sh -c ldconfig /usr/local/cuda-9.0/targets/x86_64-linux/lib/stubs &&     HOROVOD_GPU_ALLREDUCE=NCCL HOROVOD_WITH_TENSORFLOW=1 HOROVOD_WITH_PYTORCH=1 HOROVOD_WITH_MXNET=1 pip install --no-cache-dir horovod &&     ldconfig

