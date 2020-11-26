

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





## CloudCore 的高可用（部署在 k8s 集群中）

**注意：**

有很多方法实现 cloudcore 的高可用性（HA），比如，ingress、keepalived等。这里我们通过 keepalived 实现，ingress 方法会在后续实现。

#### 配置 CloudCore 的虚拟 IP 地址

配置 CloudCore 暴露给边缘节点的 VIP 地址。这里推荐使用 keepalived 实现。当使用 keepalived 时， 最好通过 nodeSelector 将 pods 调度到具体数量的 nodes 上。同时，在 CloudCore 运行的每个节点上，都需要安装 keepalived。keepalived 的配置文件在文章结尾。这里将 VIP 设置为 10.10.102.242。

nodeSelector 使用方式如下：

```bash
kubectl label nodes [nodename] [key]=[value]  # 给 cloudcore 将要运行的 node，打上标签
```

修改 nodeselector 属性：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudcore
spec:
  template:
    spec:
      nodeSelector: # 设置 nodeSelector
        [key]: [value]
```



#### 创建 k8s 资源

github.com/kubeedge/kubeedge/build/cloud/ha 目录下的文件以及脚本需要提前放在节点上，使得之后能够通过 kubectl 命令运行（你需要根据本地环境，对文件/脚本进行一些修改）。

首先，确保k8s集群可以拉取边缘控制器镜像。如果镜像不存在，可以自己制作并推送到自己的仓库。

```bash
cd $GOPATH/src/github.com/kubeedge/kubeedge
make cloudimage
```

按yaml文件名依次创建k8s资源。创建之前，**检查每一个文件的内容，确保符合本地的环境。**

**注意：**目前为止，以下文件还不支持 kubectl logs 命令。如果需要，必须手动进行更多的配置。



#### 02-ha-configmap.yaml

通过 advertiseAddress 字段，配置 CloudCore 暴露给边缘节点的 VIP 地址。该配置将以cloudcore的证书添加到SANs。比如：

```yaml
modules:
  cloudHub:
    advertiseAddress:
    - 10.10.102.242
```

**注意：**如果想要重启 CloudCore，在创建 k8s 资源之前，先运行以下命令：

```bash
kubectl delete namespace kubeedge
```

然后创建 k8s 资源：

```bash
cd build/cloud/ha
for resource in $(ls *.yaml); do kubectl create -f $resource; done
```



#### keepalived

以下是我们推荐的 keepalived 配置文件。你也可以根据需要调整。

**keepalived.conf:**

- master:

```bash
! Configuration File for keepalived

global_defs {
  router_id lb01
  vrrp_mcast_group4 224.0.0.19
}
# CloudCore
vrrp_script CloudCore_check {
  script "/etc/keepalived/check_cloudcore.sh" # health check 的脚本
  interval 2
  weight 2
  fall 2
  rise 2
}
vrrp_instance CloudCore {
  state MASTER
  interface eth0 # 根据你的主机
  virtual_router_id 167
  priority 100
  advert_int 1
  authentication {
    auth_type PASS
    auth_pass 1111
  }
  virtual_ipaddress {
    10.10.102.242/24 # VIP
  }
  track_script {
    CloudCore_check
 }
}
```

- backup:

```bash
! Configuration File for keepalived

global_defs {
  router_id lb02
  vrrp_mcast_group4 224.0.0.19
}
# CloudCore
vrrp_script CloudCore_check {
  script "/etc/keepalived/check_cloudcore.sh" # health check 的脚本
  interval 2
  weight 2
  fall 2
  rise 2
}
vrrp_instance CloudCore {
  state BACKUP
  interface eth0 # 根据你的主机
  virtual_router_id 167
  priority 99
  advert_int 1
  authentication {
    auth_type PASS
    auth_pass 1111
  }
  virtual_ipaddress {
    10.10.102.242/24 # VIP
  }
  track_script {
    CloudCore_check
 }
}
```

check_cloudcore.sh:

```bash
#!/usr/bin/env bash
http_code=`curl -k -o /dev/null -s -w %{http_code} https://127.0.0.1:10002/readyz`
if [ $http_code == 200 ]; then
    exit 0
else
    exit 1
fi
```

