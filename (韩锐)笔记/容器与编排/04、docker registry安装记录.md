### 1. 安装cert-manager

cert-manager是jetstack提供的Kubernetes原生证书管理控制器。它可以帮助从各种免费证书颁发机构，例如Let's Encrypt、HashiCorp Vault、Venafi等生成证书签名。

官方提供了2种安装方式，此处使用资源清单方式：

```bash
$ kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v0.13.1/cert-manager.yaml
```

由于需要从github上下载文件，所以安装快慢视网络情况而定。安装完成后，通过以下命令验证是否成功：

```bash
$ kubectl get pods --namespace cert-manager
NAME                                      READY   STATUS    RESTARTS   AGE
cert-manager-64b6c865d9-hn2br             1/1     Running   0          82m
cert-manager-cainjector-bfcf448b8-2t8qx   1/1     Running   0          82m
cert-manager-webhook-7f5bf9cbdf-wckvf     1/1     Running   0          82m
```

### 2. 创建资源

cert-manager主要使用两种不同的自定义资源（也称为[CRD](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)）来管理证书：

- `Issuer`  定义了应该从何处获取证书，只能应用于特定的命名空间
- `ClusterIssuer` 与Issuer功能相同，但可以应用于整个集群

为了后续使用方便，我在这里使用`ClusterIssuer`，将以下内容存储到`issuer.yaml`：

保存`issuer.yaml`：

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    # You must replace this email address with your own.
    # Let's Encrypt will use this to contact you about expiring
    # certificates, and issues related to your account.
    email: hanrui3@yiche.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      # Secret resource used to store the account's private key.
      name: letsencrypt-prod
    # Add a single challenge solver, HTTP01 using nginx
    solvers:
    - http01:
        ingress:
          class: nginx
```

接下来运行以下命令来部署和验证结果：

```bash
$ kubectl apply -f issuer.yaml
$ kubectl describe clusterissuer letsencrypt-prod
Name:         letsencrypt-prod
...
Status:
  Acme:
    Last Registered Email:  hanrui3@yiche.com
    Uri:                    https://acme-v02.api.letsencrypt.org/acme/acct/79581395
  Conditions:
    Last Transition Time:  2020-03-03T05:51:28Z
    Message:               The ACME account was registered with the ACME server
    Reason:                ACMEAccountRegistered
    Status:                True
    Type:                  Ready
Events:                    <none>
```

如果状态是Ready且没有报错说明安装成功。当然事情不总是会一帆风顺，如果提示以下信息：

> Warning  ErrVerifyACMEAccount  1s    cert-manager  Failed to verify ACME account: Get https://acme-v02.api.letsencrypt.org/directory: dial tcp: i/o timeout

请首先尝试直接访问上述地址，如果是通的，那么可能是`cert-manager`与`coredns`通信的某个环节出了问题（摊手），删除或重启一个`coredns` pod即可。

### 3. 安装registry

创建htpasswd，用户名密码：admin/yiche，执行以下命令：

```bash
$ sudo docker run --entrypoint htpasswd registry:2 -Bbn admin yiche > ./htpasswd

$ kubectl create namespace docker-registry
```

在helm仓库中搜索并安装`docker-registry`，`values.yaml`配置如下：

```yaml
service: 
  type: "ClusterIP"
secrets:
  htpasswd: "admin:$2y$05$koIK9EnRS2KxFUpFjtDlLORmjUIvVYwW5FbzNGppkPv0Akkn1GClW"
  s3:
    accessKey: minioadmin
    secretKey: minioadmin
storage: s3
s3:  #使用minio作为后端存储
  region: us-east-1
  regionEndpoint: http://10.163.6.155:9000
  bucket: docker
  encrypt: false
  secure: false
ingress:
  enabled: true
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/proxy-body-size: 100g
  hosts:
  - registry.yiche-cloud.com
  tls:
  - hosts:
    - registry.yiche-cloud.com
    secretName: registry.yiche-cloud.com-cert
```

我是用的是rancher ui安装的，直接复制上述内容安装即可。需要注意的是由于条件限制我自行注册了registry.yiche-cloud.com，并做了相应的DNS解析。

安装完成后通过以下2个步骤验证结果：

1. 尝试直接访问：`registry.yiche-cloud.com`，如果能成功访问，代表部署成功
2. 查看证书颁发机构，如果“ Let's Encrypt Authority X3”代表证书安装成功

如果步骤1不成功，请查看pod输出日志；如果证书颁发机构不对请查看证书相关资源是否部署成功。

而我在验证第2步时一直不通过，查看`ingress-nginx`日志：

```bash
$ kubectl get pod -n ingress-nginx
NAME                                    READY   STATUS    RESTARTS   AGE
default-http-backend-67cf578fc4-2gpbf   1/1     Running   0          18h
nginx-ingress-controller-nx8tv          1/1     Running   0          4h56m

$ kubectl logs -f nginx-ingress-controller-nx8tv -n ingress-nginx
```

报错信息：

> 2020/02/29 10:34:30 [error] 8637#8637: *297906 connect() failed (113: No route to host) while connecting to upstream, client: 211.159.160.128, server: registry.yiche-cloud.com, request: "GET /.well-known/acme-challenge/7LyiexUa38KY924wvDiIIaqO8F0ydFIjwQpZRPRAM_0 HTTP/1.1", upstream: "http://10.42.0.16:8089/.well-known/acme-challenge/7LyiexUa38KY924wvDiIIaqO8F0ydFIjwQpZRPRAM_0", host: "registry.yiche-cloud.com"

Let’s Encrypt 使用的是 [ACME 协议](https://letsencrypt.org/zh-cn/how-it-works/)来管理和颁发证书的，在证书颁发前，它会请求域名下指定的URL来验证域名的所有者，而上述日志表面这个URL访问不通。经过仔细排查，发现服务的ip根本不存在，Google了半天，最终发现有人提了几乎一样的[ISSUE](https://github.com/jetstack/cert-manager/issues/2388)，按照上面的说明，重新安装`cert-manager`后就好了。

### 4. 构建镜像&拉取镜像

首先需要登录到registry：

```bash
$ docker login registry.yiche-cloud.com --username admin --password yiche

WARNING! Using --password via the CLI is insecure. Use --password-stdin.
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```

构建镜像并push：

```bash
$ cd ~/horovod_dockerfiles/
$ sudo docker build -t ychorovod:v1 ./
$ sudo docker tag ychorovod:v1 registry.yiche-cloud.com/yiche/horovod:latest
$ sudo docker push registry.yiche-cloud.com/yiche/horovod:latest
```

接下来，为了能在kubernetes中拉取镜像，需要以下操作：

首先，将私有仓库账号信息配置为secret：

```bash
$ kubectl create secret docker-registry private-repo \
	--docker-server=registry.yiche-cloud.com \
	--docker-username=admin \
	--docker-password=yiche \
	--namespace default
```

然后，编辑service帐户并授予其访问secret的权限:

```bash
$ kubectl edit serviceaccount default -n default
```

添加以下内容并保存：

```yaml
..
imagePullSecrets:
- name: private-repo
```

为了验证是否成功，可以尝试拉取镜像：

```bash
#需要先push镜像horovod:0.1-tf1.12到仓库
$ kubectl run --rm -it horovod-test --restart Never --image  registry.yiche-cloud.com/base/horovod:0.1-tf1.12 -- nvidia-smi
```

如果正常输出，说明一切成功，如果报错通过以下命令查看问题：

```bash
$ kubectl get events --sort-by=.metadata.creationTimestamp -n default
```

### 5. 查看证书资源相关命令

整个安装过程中和证书相关的地方比较容易出错，在此记录一些常见的命令协助排查问题。

1. 查看证书列表

   ```bash
   $ kubectl get certificate -n docker-registry
   NAME                            READY   SECRET                          AGE
   registry.yiche-cloud.com-cert   True    registry.yiche-cloud.com-cert   27m
   ```

2. 查看证书详情

   ```bash
   $ kubectl describe certificate registry.yiche-cloud.com-cert -n docker-registry
   Name:         registry.yiche-cloud.com-cert
   Namespace:    docker-registry
   Labels:       app=docker-registry
                 chart=docker-registry-1.9.2
                 heritage=Tiller
                 io.cattle.field/appId=docker-registry
                 release=docker-registry
   ...
   Events:
     Type    Reason        Age   From          Message
     ----    ------        ----  ----          -------
     Normal  GeneratedKey  28m   cert-manager  Generated a new private key
     Normal  Requested     28m   cert-manager  Created new CertificateRequest resource "registry.yiche-cloud.com-cert-351922934"
     Normal  Issued        26m   cert-manager  Certificate issued successfully
   ```

3. 查看order详情

   ```bash
   $ kubectl describe order registry.yiche-cloud.com-cert-351922934 -n docker-registry
   Name:         registry.yiche-cloud.com-cert-351922934-3093567077
   Namespace:    docker-registry
   Labels:       app=docker-registry
                 chart=docker-registry-1.9.2
                 heritage=Tiller
                 io.cattle.field/appId=docker-registry
                 release=docker-registry
   ...
   Events:
     Type    Reason    Age   From          Message
     ----    ------    ----  ----          -------
     Normal  Created   29m   cert-manager  Created Challenge resource "registry.yiche-cloud.com-cert-351922934-3093567077-4280627387" for domain "registry.yiche-cloud.com"
     Normal  Complete  27m   cert-manager  Order completed successfully
   ```

4. 查看challenge

   ```bash
   $ kubectl describe challenge registry.yiche-cloud.com-cert-351922934-3093567077-4280627387 -n docker-registry
   ```

### 参考

1. https://cert-manager.io/docs
2. https://github.com/alexellis/k8s-tls-registry
3. https://github.com/jetstack/cert-manager/issues/2388
4. https://letsencrypt.org/zh-cn/how-it-works/