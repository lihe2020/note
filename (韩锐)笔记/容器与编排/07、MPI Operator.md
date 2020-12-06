1. 安装crd：

   mpi-operator资源定义文件路径：https://github.com/kubeflow/mpi-operator/raw/master/deploy/mpi-operator.yaml

   由于（mpi-operator所属的）kubeflow依赖的k8s版本过低，所以直接使用上述文件可能会报错，请按照错误提示修正yaml文件即可。

   > 错误大都是openAPIV3Schema下的对象缺少type描述，加上即可

2. 



