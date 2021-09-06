虽然kubernetes早在2018年5月就[宣布](https://kubernetes.io/blog/2018/05/24/kubernetes-containerd-integration-goes-ga/)用户可以不用安装docker，直接使用containerd作为CRI运行时，docker依然在生产环境中有很高的装机量，并且在单机开发环境中使用docker相对containerd更为方便。因此即使在2021年docker依然有其存在价值。

2021年8月，debian11终于发布，本文描述如何在debian11上安装docker。主要流程参考官方[文档](https://docs.docker.com/engine/install/debian/)。

1. 安装基础工具  
```sh
 sudo apt-get update
 sudo apt-get install \
	apt-transport-https \
	ca-certificates \
	curl \
	gnupg \
	lsb-release
```
1. 安装docker的gpg key：  
```sh
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```
1. 安装docker源  
```sh
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
上面命令中的`lsb_release -cs`返回`bullseye`，也就是debian11的代号。
1. 安装docker  
```sh
apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
```

至此安装完毕。

在debian系的Linux发行版上，docker会开机启动启动。

如果平时使用非root账户，又不想每次执行docker命令之前都加上sudo，参考docker的[文档](https://docs.docker.com/engine/install/linux-postinstall/#manage-docker-as-a-non-root-user)，可以添加`docker`组，并将非root账户加入到该组中。下面命令创建`docker`组并将当前用户加入`docker`组，执行完成之后重新登陆生效：

```sh
sudo groupadd docker
sudo usermod -aG docker $USER
```
