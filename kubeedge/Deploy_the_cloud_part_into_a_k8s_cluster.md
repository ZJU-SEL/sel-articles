

## 在k8s集群中部署kubeedge云端

以下教程展示了如何在k8s集群中部署kubeedge云端，所有操作可以在k8s集群master节点上进行，或者在可以通过kubectl操作集群的节点上进行。

github.com/kubeedge/kubeedge/build/cloud 目录下的文件以及脚本需要提前放在节点上，使得之后能够通过 kubectl 命令运行。

#### 准备 cloud image

确保k8s集群可以拉取边缘控制器镜像。如果镜像不存在，可以自己制作并推送到自己的仓库。

```bash
cd $GOPATH/src/github.com/kubeedge/kubeedge
make cloudimage
```



#### 创建 secret (可选)

如果版本小于1.3.0，需要生成 tls certs，并且在此基础上，创建 06-secret.yaml。

```bash
cd build/cloud
../tools/certgen.sh buildSecret | tee ./06-secret.yaml
```



#### 配置 IP 地址（可选）

KubeEdge 1.3.0 开始，可以在 05-configmap.yaml，配置 CloudCore 暴露给边缘节点的 IP 地址（比如 floating IP），相关配置将以cloudcore的证书添加到SANs。

```
modules:
  cloudHub:
    advertiseAddress:
    - 10.1.11.85
```



#### 更新配置

根据 08-service.yaml.example，创建属于你自己的 service 的 yaml 文件 08-service.yaml，将 cloud hub 暴露在k8s集群外部，使得 edge core 能够连接到 cloud hub。

同时，检查每一个文件的内容，确保符合本地的环境。



#### 创建 cloud 资源

按yaml文件名依次创建k8s资源。

```bash
for resource in $(ls *.yaml); do kubectl create -f $resource; done
```


