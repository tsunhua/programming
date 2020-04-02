# Kubernetes

在认识 Kubernetes 之前，我们需了解下容器，在了解容器之前，我们得先知道什么是虚拟机。

## 什么是虚拟机？

虚拟机（VM, Virtual Machine）是计算机系统的仿真，以便隔离真实计算机硬件，运行多个不同的操作系统。虚拟化技术是硬件隔离的，运行在虚拟机（客体机）中的程序不直接与真实机器（宿主机）交互。

常见的虚拟机软件有：

| 软件 | 初次发布时间 | 早期主导的公司 | 操作系统 | 编写语言 |
| :--- | :--- | :--- | :--- | :--- |
| VMware Workstation | 1999 | VMware | Windows, Linux | C, C++, F\# |
| VirtualBox | 2007.1.17 | Oracle | Windows, macOS, Linux and Solaris | C, C++, x86 Assembly |
| OpenStack | 2010.10.21 | Rackspace Hosting, NASA | 跨平台 | Python |

虚拟机因为要隔离出单独的一个个系统，相比容器，占用内存和磁盘空间较大，运行起来很笨重。

## 什么是容器？

容器（Container），通常是指 Linux 容器（LXC, Linux Containers ），是轻量级的虚拟化技术（我们把虚拟机技术称之为重量级的虚拟化技术）。它不隔离出一个个子系统，而是直接运行在真机上，隔离的是进程，使用的底层技术其实是 Linux Namespaces（Linux 命名空间）。什么是 Linux Namespaces 呢？

在计算机领域，Namespace 是用来表明标识符的可见范围。在 XML 定义中就用 xmlns（XML Namespace）来确定标签的含义，隔离命名冲突，例如：

```markup
<html>
<h:table xmlns:h="http://www.w3.org/TR/html4/">
   <h:tr>
   <h:td>Apples</h:td>
   <h:td>Bananas</h:td>
   </h:tr>
</h:table>
<f:table xmlns:f="http://www.w3school.com.cn/furniture">
   <f:name>African Coffee Table</f:name>
   <f:width>80</f:width>
   <f:length>120</f:length>
</f:table>
</html>
```

学过 Android 的，可以联想下 Android XML 布局文件中默认的 xmlns

```markup
xmlns:android=``"http://schemas.android.com/apk/res/android"
```

以及我们自定义组件属性时，使用的 xmlns

```text
xmlns:app=``"http://schemas.android.com/apk/res-auto"
```

学过 C++的还可以联想下其关键字 namespace。

言归正传，Linux Namespaces 是 Linux 内核中对容器的支持，不同 Namespace 之间内核资源是隔离的。Linux Namespaces 最早出现在 Linux 2.4，到 4.6 时已经有 7 种 Namespaces了，每种 Namespace 隔离不同种类的内核资源，具体列表如下：

| Namespace | Since | Constant | Isolates |
| :--- | :--- | :--- | :--- |
| Cgroup | Linux 2.6.24 → Linux 4.5 | CLONE\_NEWCGROUP | Cgroup root directory（Cgroup 的根目录） |
| IPC | Linux 2.6.19 | CLONE\_NEWIPC | System V IPC, POSIX message queues（信号量、消息队列和共享内存） |
| Network | Linux 2.6.24 → Linux 2.6.29 | CLONE\_NEWNET | Network devices, stacks, ports, etc.（网络设备、栈和端口等） |
| Mount | Linux 2.4.19 | CLONE\_NEWNS | Mount points（文件系统挂载点） |
| PID | Linux 2.6.24 | CLONE\_NEWPID | Process IDs（进程 ID） |
| User | Linux 2.6.23 → Linux 3.8 | CLONE\_NEWUSER | User and group IDs（用户和组 ID） |
| UTS | Linux 2.6.19 | CLONE\_NEWUTS | Hostname and NIS domain name（主机名和 NIS 域名） |

重要的 6 种 Namespace 在 2010 年就已经出现并多数编码完成，Linux 的容器支持愈加完善，Docker 的出现正逢其时。

## Docker

我们前面讲到 Docker，始于 2010 年 dotCloud 开发的基于Linux 容器的虚拟化技术，即 Docker 技术。这项技术于 2013 年 3 月在 Github 上开源并发布 v0.1 版本，吸引大量人气，甚至 Google、Redhat、IBM 和Amazon 等大公司也表示全力支持。到2014年6月9日，Docker v1.0 正式发布。如前述虚拟机软件，整理 Docker 基本信息如下：

| 软件 | 初次发布时间 | 早期主导的公司 | 操作系统 | 编写语言 |
| :--- | :--- | :--- | :--- | :--- |
| Docker | 2013.3 | dotCloud（Docker Inc.） | Linux, Windows, MacOS | Go |

初步认识 Docker，只需要了解其中的 3 个概念：

1. 镜像（Image），这里的镜像不是我们平常所熟知的 ISO 压缩文件，而是一种使用 Union FS 分层存储技术存储的 root 文件系统，包含容器运行时所需的程序、库、资源和配置等文件。
2. 容器（Container），Docker 语义下的容器是镜像运行时的实体，其实质就是一个 Docker 进程。可以对容器执行创建、启动、停止、删除和暂停操作。容器的存储方式也是分层存储，以镜像为基础层，在其上创建一个当前容器的存储层，称之为容器存储层。（注：容器存储层的生命周期与容器同步，故写入文件时应使用数据卷或绑定宿主目录）
3. 登记处（Registry），一个登记处可以登记多个仓库，每个仓库可以包含多个标签，每个标签对应一个镜像。登记处便利了镜像的分发和管理。

## Kubernetes

随着 Docker 技术的流行，人们对统一部署、扩展和运行容器集群的需求日益强烈。于是，Kubernetes 应运而生。

Kubernetes （源自古希腊语，意为领航员、舵手，简称 K8S，8 代表中间的 8 个字母）由 Google 公司于 2014 年推出，并于 2015 年 7 月 21 日发布 v1.0 。Kubernetes 系统深受 Google 内部另一个大型集群管理系统 Borg 的影响，其 logo 上的七个轮辐就是在向电影《星际迷航》中的 Borg 人 Seven 致意。

### 架构

Kubernetes 采用主从式架构，一个集群控制节点（Master）和多个工作节点（Node）。包含以下核心组件：

![](../.gitbook/assets/image%20%286%29.png)

图：Kubernetes 核心组件

![](../.gitbook/assets/image%20%287%29.png)

图：Kubernetes 架构（官方）

![](../.gitbook/assets/image%20%282%29.png)

图：Kubernetes 架构（AWS）

![](../.gitbook/assets/image%20%284%29.png)

图：Kubernetes 架构（Slide from [this](https://schd.ws/hosted_files/kccnceu18/00/What%20does%20“production%20ready”%20really%20mean%20for%20a%20Kubernetes%20cluster.pdf) presentation from Lucas Käldström \(@kubernetesonarm\)）

![](../.gitbook/assets/image.png)

图：Kubernetes 声明式 API 设计

![](../.gitbook/assets/image%20%285%29.png)

图：Kubernetes 架构

### Kubernetes 对象

Kubernetes 采用声明式设计，通过 API 对象声明期望的状态（desired state），然后系统会通过控制面（control plane）最终达到期望。

![](../.gitbook/assets/image%20%283%29.png)

图：Kubernetes 对象

### 基础设施抽象

![](../.gitbook/assets/image%20%281%29.png)

图：基础设施抽象

## 创建集群

### 安装 kubectl

kubectl 是 Kubernetes 的命令行工具，可以通过跑命令来控制整个 Kubernetes 集群。

> 注意：kubectl 的版本要确保与 Kubernetes 最多上下相差一个小版本。安装最新的版本可以无视此项规定。

#### 安装 kubectl 到 Linux

**安装二进制版本**

```text
curl -LO https:``//storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl``chmod +x ./kubectl``sudo mv ./kubectl /usr/local/bin/kubectl``kubectl version
```

**使用本地包管理器**

**Ubuntu, Debian or HypriotOS**

```text
sudo apt-get update && sudo apt-get install -y apt-transport-https``curl -s https:``//packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -``echo ``"deb https://apt.kubernetes.io/ kubernetes-xenial main"` `| sudo tee -a /etc/apt/sources.list.d/kubernetes.list``sudo apt-get update``sudo apt-get install -y kubectl
```

**CentOS, RHEL or Fedora**

```text
cat < /etc/yum.repos.d/kubernetes.repo``[kubernetes]``name=Kubernetes``baseurl=https:``//packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64``enabled=``1``gpgcheck=``1``repo_gpgcheck=``1``gpgkey=https:``//packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg``EOF``yum install -y kubectl
```

#### 安装 kubectl 到 macOS

**安装二进制版本**

```text
curl -LO https:``//storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/darwin/amd64/kubectl``chmod +x ./kubectl``sudo mv ./kubectl /usr/local/bin/kubectl``kubectl version
```

**使用 Homebrew 安装**

```text
brew install kubernetes-cli
```

#### 安装 kubectl 到 Windows

**安装二进制版本**

```text
curl -LO https:``//storage.googleapis.com/kubernetes-release/release/v1.15.0/bin/windows/amd64/kubectl.exe``// 添加二进制文件到环境变量 PATH 中``kubectl version
```

#### 添加命令自动补全支持

**Zsh**

自动补全的脚步可以通过 kubectl completion zsh 查看。下面是配置步骤：

在 ~/.zshrc 文件中添加

```text
source <(kubectl completion zsh)
```

如果出现 complete:13: command not found: compdef 之类的错误，就在 ~/.zshrc 文件开头添加

```text
autoload -Uz compinit``compinit
```

### 安装 Minikube

Minikube 可以在本地环境中的虚拟机上快速部署一个单节点的 Kubernetes 集群。

#### 确保系统支持虚拟化技术

**Linux**

执行以下命令，当输出不为空时即可。

```text
egrep --color ``'vmx|svm'` `/proc/cpuinfo
```

**macOS**

执行以下命令，当输出不为空时即可。

```text
sysctl -a | grep machdep.cpu.features | grep VMX
```

**Windows**

执行以下命令

```text
systeminfo
```

当输出以下内容时，系统支持虚拟化技术。

```text
Hyper-V Requirements:   VM Monitor Mode Extensions: Yes``             ``Virtualization Enabled In Firmware: Yes``             ``Second Level Address Translation: Yes``             ``Data Execution Prevention Available: Yes
```

当输出以下内容时，系统支持虚拟化技术，并且 Hypervisor 已经安装了，可以忽略下个步骤。

```text
Hyper-V Requirements:   A hypervisor has been detected. Features required ``for` `Hyper-V will not be displayed.
```

#### 安装 Hypervisor

**Linux**

当系统没有安装 Hypervisor 时，选择下列其中的一个安装即可。

* [KVM](https://www.linux-kvm.org/), which also uses QEMU
* [VirtualBox](https://www.virtualbox.org/wiki/Downloads)

**macOS**

当系统没有安装 Hypervisor 时，选择下列其中的一个安装即可。

* [HyperKit](https://github.com/moby/hyperkit)
* [VirtualBox](https://www.virtualbox.org/wiki/Downloads)
* [VMware Fusion](https://www.vmware.com/products/fusion)

**Windows**

当系统没有安装 Hypervisor 时，选择下列其中的一个安装即可。

* [Hyper-V](https://msdn.microsoft.com/en-us/virtualization/hyperv_on_windows/quick_start/walkthrough_install) （只支持 Win10 企业版本、专业版和教育版）
* [VirtualBox](https://www.virtualbox.org/wiki/Downloads)

#### 安装 Minikube

**Linux**

```text
curl -Lo minikube https:``//storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 \`` ``&& chmod +x minikube``sudo install minikube /usr/local/bin
```

**Mac**

使用 Homebrew

```text
brew cask install minikube
```

使用二进制文件

```text
curl -Lo minikube https:``//storage.googleapis.com/minikube/releases/latest/minikube-darwin-amd64 \`` ``&& chmod +x minikube``sudo mv minikube /usr/local/bin
```

**Windows**

使用安装器

```text
下载并运行 minikube-installer.exe
```

## 启动集群

运行 minikube start ，如提示 machine does not exist，则运行 minikube delete 清理本地状态。

## 启动 Dashboard

运行 minikube dashboard 会自动启动一个网页，在里面可以管理整个集群。

## 部署一个应用

以下过程可以通过 [Module 2 - Deploy an app](https://kubernetes.io/docs/tutorials/kubernetes-basics/deploy-app/deploy-interactive/)，在线交互式地完成。

### 查看所有节点及其状态

```text
kubectl get nodes
```

### 部署 Hello World 应用

```text
kubectl run kubernetes-bootcamp --image=gcr.io/google-samples/kubernetes-bootcamp:v1 --port=``8080
```

通过以下查看部署的应用及状态，等待状态变为 Ready 。

```text
kubectl get deployments
```

### 创建代理，直连 Pod

可以通过以下命令创建一个代理，直接连接部署的 Pod 进行访问，这是暂时的，后面我们会使用 NodePort、LoadBalancer 等方式暴露服务。

```text
kubectl proxy
```

通过以下命令查看Kubernetes 版本，测试代理是否成功。

```text
curl http:``//localhost:8001/version
```

### 访问部署的应用

```text
export POD_NAME=$(kubectl get pods -o go-template --template ``'{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}'``)``echo Name of the Pod: $POD_NAME``curl http:``//localhost:8001/api/v1/namespaces/default/pods/$POD_NAME/proxy/
```

## 探索你的应用

以下过程可以通过[ Module 3 - Explore your app](https://kubernetes.io/docs/tutorials/kubernetes-basics/explore/explore-interactive/)，在线交互式地完成。

### 获取集群 Pod 信息

通过以下命令，可以获取集群中所有 Pod 的信息，包含名称、名称空间、优先级、节点、状态、IP、容器、卷、事件等等。

```text
kubectl describe pods
```

### 查看容器日志

```text
export POD_NAME=$(kubectl get pods -o go-template --template ``'{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}'``)``echo Name of the Pod: $POD_NAME``kubectl logs $POD_NAME
```

### 在容器中执行命令

使用 kubectl 的 exec 指令，可以在容器中执行命令。因为我们之前部署的是单容器的 Pod，因此可以不用指定容器，使用 Pod Name 即可。如果多容器的话可以通过 -c 指定 Container。

```text
kubectl exec $POD_NAME env
```

还可以直接打开容器内的 bash 进行交互，如下。

```text
kubectl exec -ti $POD_NAME bash``ls``cat server.js``curl localhost:``8080``exit
```

## 对外暴露你的应用

以下过程可以通过 [Module 4 - Expose your app publicly](https://kubernetes.io/docs/tutorials/kubernetes-basics/expose/expose-interactive/)，在线交互式地完成。

### 通过 NodePort 方式创建服务

```text
kubectl get pods``kubectl get services``// type 可选值：ClusterIP、NodePort、LoadBalancer``kubectl expose deployment/kubernetes-bootcamp --type=``"NodePort"` `--port ``8080``kubectl get services``export NODE_PORT=$(kubectl get services/kubernetes-bootcamp -o go-template=``'{{(index .spec.ports 0).nodePort}}'``)``curl $(minikube ip):$NODE_PORT
```

### 打标签

使用 kubectl label 可以给 deployment、pod 以及 service 等等打标签，随后可以通过 kubectl get \[deployment \| pods \| services \| svc\] -l 过滤带有指定标签的部署、Pod 或者服务等等。

```text
// 给 Pod 打标签``export POD_NAME=$(kubectl get pods -o go-template --template ``'{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}'``)``echo Name of the Pod: $POD_NAME``kubectl label pod $POD_NAME app=v1``// 描述 Pod``kubectl describe pods $POD_NAME``// 使用标签过滤 Pod``kubectl get pods -l app=v1
```

### 删除一个服务

```text
// 删除服务``kubectl delete service -l run=kubernetes-bootcamp``// 列出现有服务``kubectl get services``// 外部不可访问``curl $(minikube ip):$NODE_PORT``// 内部可以访问``kubectl exec -ti $POD_NAME curl localhost:``8080
```

## 扩充你的应用

以下过程可以通过 [Module 5 -Scaling Your App](https://kubernetes.io/docs/tutorials/kubernetes-basics/scale/scale-interactive/)，在线交互式地完成。

### 扩充 Deployment

```text
kubectl get deployments``kubectl scale deployments/kubernetes-bootcamp --replicas=``4``kubectl get deployments``kubectl get pods -o wide``kubectl describe deployments/kubernetes-bootcamp
```

### 负载均衡

扩充后，Service 是负载均衡的，测试如下：

```text
kubectl describe services/kubernetes-bootcamp``export NODE_PORT=$(kubectl get services/kubernetes-bootcamp -o go-template=``'{{(index .spec.ports 0).nodePort}}'``)``echo NODE_PORT=$NODE_PORT``curl $(minikube ip):$NODE_PORT``curl $(minikube ip):$NODE_PORT``curl $(minikube ip):$NODE_PORT
```

### 收缩 Deployment

```text
kubectl scale deployments/kubernetes-bootcamp --replicas=``2``kubectl get deployments``kubectl get pods -o wide
```

## 更新你的应用

以下过程可以通过 [Module 6 -Updating Your App](https://kubernetes.io/docs/tutorials/kubernetes-basics/update/update-interactive/)，在线交互式地完成。

### 更新应用版本

更新应用时，系统会扩充 Deployment，然后部署新镜像，旧的 Deployment 进入Terminating 状态。流量仅会在可用的 Deployments 之间负载均衡。

```text
kubectl get deployments``kubectl get pods``kubectl describe pods``kubectl set image deployments/kubernetes-bootcamp kubernetes-bootcamp=jocatalin/kubernetes-bootcamp:v2``kubectl get pods
```

### 验证更新

```text
// 通过响应内容验证``kubectl describe services/kubernetes-bootcamp``export NODE_PORT=$(kubectl get services/kubernetes-bootcamp -o go-template=``'{{(index .spec.ports 0).nodePort}}'``)``echo NODE_PORT=$NODE_PORT``curl $(minikube ip):$NODE_PORT` `// 通过 rollout status 查看升级进度``kubectl rollout status deployments/kubernetes-bootcamp` `kubectl describe pods
```

> 注意：当且仅当 Deployment 中的 pod template（例如.spec.template） 中的 label 更新或者镜像更改时触发 rollout。

### 回滚更新

```text
kubectl rollout undo deployments/kubernetes-bootcamp
```

## 删除你的应用

### 删除服务

```text
kubectl delete service kubernetes-bootcamp
```

### 删除部署

```text
kubectl delete deployment kubernetes-bootcamp
```

## 停用集群

```text
minikube stop
```

## 删除集群

```text
minikube delete
```

