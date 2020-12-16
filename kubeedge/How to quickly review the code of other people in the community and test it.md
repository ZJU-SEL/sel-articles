# How to quickly review the code of other people in the community and test it



# bashrc

https://github.com/GsssC/bashrc

学长的脚本, 便于bashrc配置（环境以及源码下载）



# Windows

Windows主要使用github desktop做代码开发, 假设有下图pr我们希望在本地观察它的源码!

![test-windows-1](./images/test-windows-1.png)



1. 点开右上角Open with, 点开Open in Desktop

   ![test-windows-2](./images/test-windows-2.png)

2. GitHub Desktop会自动启动, 加入原工作区有未保存的变更, 会通知进行stash

![test-windows-3](./images/test-windows-3.png)

3. 选择将工作区stash后, switch branch, 可以看见分支已经切换 

![test-windows-4](./images/test-windows-4.png)

4. 使用git stash list 可以看见刚才stash的内容

![test-windows-5](./images/test-windows-5.png)





# Linux CentOS

1. 安装gh

```bash
yum install dnf dnf-plugins-core

sudo dnf config-manager --add-repo https://cli.github.com/packages/rpm/gh-cli.repo

sudo dnf install gh 
```

2. gh的使用需要github token, 进入settings/developer settings生成github token

   ![test-linux-1](./images/test-linux-1.png)

3. gh使用的最小权限是repo + read:org

![test-linux-2](./images/test-linux-2.png)

4. 生成后注意复制保存秘钥

![test-linux-3](./images/test-linux-3.png)

5. 登录gh

![test-linux-4](./images/test-linux-4.png)

输入刚才生成的token

![test-linux-5](./images/test-linux-5.png)

6. 使用gh

![test-linux-6](./images/test-linux-6.png)

将代码 gh pr checkout 2418 复制后在centos命令行内粘贴输入, 完成切换, 十分方便

![test-linux-7](./images/test-linux-7.png)

 



# MAC 安装：

（1）安装 gh

```bash
brew install gh
```

（2）相关配置

<img src="./images/test-mac-1.png" alt="test-mac-1" style="zoom:75%;" />



（3）使用 gh

```bash
# 如果 base repo 设置的是原始仓库
gh pr checkout 2418

# 如果 base repo 设置的是自己 fork 的仓库，需要指定 base repo
# base repo 的格式为"[HOST/]OWNER/REPO"，这里是kubeedge/kubeedge，
gh pr checkout -R kubeedge/kubeedge 241
```





