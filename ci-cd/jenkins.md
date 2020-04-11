# Jenkins

## 概述

Jenkins 是一款跨平台的持续集成和持续交付（CI/CD, continuous integration and continuous delivery）应用。

它具备以下特性：

1. 易于安装，只需要运行 `java -jar jenkins.war` 即可。
2. 易于配置，所有配置都能通过 GUI 进行。
3. 丰富的插件生态，提供 [SCM\(Source Control Management，源码控制管理\) 和构建工具](https://wiki.jenkins-ci.org/display/JENKINS/Plugins)。
4. 可扩展，可以很方便地创建新的 Jenkins 插件。
5. 分布式构建，可以将构建或测试负载分发到多台计算机多种操作系统。

## 起步

### 安装

见：[https://jenkins.io/doc/book/installing/](https://jenkins.io/doc/book/installing/)

在 Amazon Linux 中安装 Jenkins，参考：

```text
# install jenkins
sudo wget -O /etc/yum.repos.d/jenkins.repo http://pkg.jenkins.io/redhat/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat/jenkins.io.key
sudo yum install jenkins -y
sudo service jenkins start
```

### 官方教程

见：[https://jenkins.io/doc/pipeline/tour/getting-started/](https://jenkins.io/doc/pipeline/tour/getting-started/)

### 设置环境变量

从 Jenkins GUI 依次点击`系统管理 -> 系统设置 -> 环境变量` ，添加环境变量。键填入：PATH，值填入所需的配置，例如： `/sbin:/usr/sbin:/bin:/usr/bin:/usr/local/bin:/usr/local/go/bin/:/home/ec2-user/bin`。

### 更改 Jenkins 用户

默认的 Jenkins 用户为 `Jenkins` ，很多命令都很受限，无法执行。特别是当在 aws EC2 中使用时，AWS 的服务角色授权是授予 `ec2-user` 的，其他用户（包括 `root`）都无法正常使用 AWS CLI 相关命令。更改 Jenkins 用户的步骤如下（以 `ec2-user` 为例）：

（1）编辑 Jenkins 配置，设置 `JENKINS_USER`。

```text
vim /etc/sysconfig/jenkins
```

找到并修改 `JENKINS_USER="ec2-user"`。

（2）修改 Jenkins 相关文件权限。

```text
sudo chown -R ec2-user:ec2-user /var/lib/jenkins
sudo chown -R ec2-user:ec2-user /var/cache/jenkins
sudo chown -R ec2-user:ec2-user /var/log/jenkins
```

（3）重启服务

```text
sudo service jenkins restart
```

## QA

### Jenkins 的工作目录在哪里？

假定新建了名为 `projectA` 的 Jenkins 任务，当执行 Jenkins 构建时，其工作目录位于 `/var/lib/jenkins/workspace/projectA`。

### 参考

1. [Meet Jenkins – wiki.jenkins.io](https://wiki.jenkins.io/display/JENKINS/Meet+Jenkins)
