# 基于 EKS 的 Go 服务部署及监控方案

## 概述

本方案依服务的部署管理到流量管理的次序编写，行文结构上先给出方案，再说明方案的原理和实现过程。

### 方案概述

| 需求 | 方案 |
| :--- | :--- |
| 构建集群 | 采用 Amazon EKS（Elastic Kubernetes Service）构建 Kubernetes 集群。 |
| 容器化 | 采用 Alpine Linux 作为基础镜像，构建镜像推送到 Amazon ECR 仓库，然后再使用预置的 Kubernetes 部署文件部署到 Kubernetes 集群。 |
| CI/CD | 采用 Jenkins + Git Parameter Plugin + Gitlab 进行 CI/CD，当通过 Jenkins Dashboard 发出构建请求后，打包机会执行上述容器化过程（即构建应用、构建镜像、发布镜像和部署服务）。 |
| 服务发现 | Kubernetes 1.13 版本默认集成了 CoreDNS 作为内部服务发现的组件，对集群内部的服务我们可以直接通过 "\[命名空间.\]服务名" 方式访问，对外部的服务我们可以通过设置 ExternalName 或者定义存根域和上游 DNS 的方式实现。 |
| 配置管理 | 采用 Kubernetes 的 ConfigMap 资源对象定义配置，随后通过挂载卷方式读取。配置变更通过 Gitlab + Jenkins 实现自动化部署。 |
| 负载均衡 | 采用 Istio Gateway 实现 HTTP 和 gRPC 请求的服务端负载均衡。 |
| 灰度发布 | 采用 Istio Gateway 对外暴露服务，然后修改 VirtualService 资源，控制 http rout 中各个 destination 的 weight 即可。 |
| 服务熔断 | 采用 Istio + Envoy 的服务网格方案，在原先的 Pod 中注入 Envoy 代理（即所谓的边车注入），从而达到管理流量的目的，包括服务熔断。 |
| 日志收集 | 采用 EFK 方案，在集群的每个节点部署一个 Fluent Bit，负责收集日志转发到 Fluentd，Fluentd 处理完数据后路由到多个不同的输出节点（比如 ElasticSearch、S3）。 |
| 服务调用追踪 | 采用 Istio 插件 jaeger tracing 进行服务调用追踪和展示。 |
| 服务监控 | 采用 Prometheus 进行服务监控，随后通过企业微信群机器人进行告警通知 |

### 面板概述

1. JUMPER 占位，表示跳板机。
2. JUMPER\_IP 占位，表示跳板机的公网 IP，可以给该公网 IP 设置安全组，以便在公司内部可访问，而在外部不可访问达到安全的目的。

#### Jenkins

功能：CI/CD

访问方式：访问 [http://JUMPER\_IP:8080](http://JUMPER_IP:8080)

#### Kubernetes

功能：管理部署

访问方式：

a. 将从任意地址请求到Amazon EC2 实例 6443 端口的请求转发到 Kubernetes Dashboard 的 443 端口

```text
kubectl port-forward --address 0.0.0.0 svc/kubernetes-dashboard -n kube-system 6443:443 &
```

b. 检索 eks-admin 服务账户的身份验证令牌。从输出中复制  值。您可以使用此令牌连接到控制面板

```text
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep eks-admin | awk '{print $1}')
```

c. 访问 [https://JUMPER\_IP:6443，输入令牌。](https://jumper_ip:6443，输入令牌。/)

#### Kiali

功能：查看流量

访问方式：

a. 在 JUMPER 中执行以下命令，将从任意地址请求到 JUMPER\_IP:20001 的转发到 Kiali Dashboard 的 20001 端口。

```text
kubectl -n istio-system port-forward --address 0.0.0.0 $(kubectl -n istio-system get pod -l app=kiali -o jsonpath='{.items[0].metadata.name}') 20001:20001 &
```

b. 访问 [https://JUMPER\_IP:20001。](https://jumper_ip:20001。/)

#### Jaeger

功能：服务调用追踪

访问方式：

a. 在 JUMPER 中执行以下命令，将从任意地址请求到 JUMPER\_IP:15032 的转发到 Jaeger Dashboard 的 16686 端口。

```text
kubectl -n istio-system port-forward --address 0.0.0.0  $(kubectl -n istio-system get pod -l app=jaeger -o jsonpath='{.items[0].metadata.name}') 15032:16686 &
```

b. 访问 [https://JUMPER\_IP:15032。](https://jumper_ip:15032。/)

#### Grafana

功能：查看服务器性能

访问方式：

a. 在 JUMPER 中执行以下命令，将从任意地址请求到 JUMPER\_IP:3000 的转发到 Grafana Dashboard 的 3000 端口。

```text
kubectl -n grafana port-forward --address 0.0.0.0 $(kubectl -n grafana get pod -l app=grafana -o jsonpath='{.items[0].metadata.name}') 3000:3000 &
```

b. 访问 [https://JUMPER\_IP:3000。](https://JUMPER_IP:3000。)

#### Kibana

功能：查看日志

访问方式：

a. 在 JUMPER 中执行以下命令，将从任意地址请求到 JUMPER\_IP:5601 的转发到 Kibana Dashboard 的 5601 端口。

```text
kubectl -n logging port-forward --address 0.0.0.0 $(kubectl -n logging get pod -l app=kibana -o jsonpath='{.items[0].metadata.name}') 5601:5601 &
```

b. 访问 [https://JUMPER\_IP:5601。](https://jumper_ip:5601。/)

#### Prometheus

a. 在 JUMPER 中执行以下命令，将从任意地址请求到 JUMPER\_IP:9090的转发到 Prometheus Dashboard 的 9090 端口。

```text
kubectl -n prometheus port-forward --address 0.0.0.0 $(kubectl -n prometheus get pod -l app=prometheus -o jsonpath='{.items[0].metadata.name}') 9090:9090 &
```

b.访问[https://JUMPRRT\_IP:9090](https://JUMPRRT_IP:9090)

## 构建集群

### 方案概述

采用 Amazon EKS（Elastic Kubernetes Service）构建 Kubernetes 集群。

### 相关原理

#### Kubernetes

Kubernetes 是一个可移植的、可扩展的开源平台，用于管理容器化工作负载和服务，便于声明式配置和自动化。Kubernetes 1.0 版本于 2015 年 7 月发布。Kubernetes 横向采用 C/S 架构，Client 通过 kubectl 工具发送声明式的 API 对象配置到 Server 端的 API Server 进行部署管理，Server 端包含 Master 和 Node 节点组，具体架构如下图所示：

![](../.gitbook/assets/image%20%2818%29.png)

图：Kubernetes 横向架构

Kubernetes 的出现方便了我们对容器化应用进行编排。

#### Amazon EKS

Amazon EKS 是一项托管服务，可跨多个可用区运行 Kubernetes 控制层面实例以确保高可用性。Amazon EKS 可以自动检测和替换运行状况不佳的控制层面实例，并为它们提供自动版本升级和修补。

Amazon EKS 还与许多 AWS 服务集成以便为您的应用程序提供可扩展性和安全性，包括：

* 用于容器镜像的 Amazon ECR
* 用于负载分配的 Elastic Load Balancing
* 用于身份验证的 IAM
* 用于隔离的 Amazon VPC

目前 EKS 支持的 Kubernetes 最新版本是 1.13.7。

### 实现过程

本过程采用 eksctl 创建集群。eksctl 是 EKS 官方提供的创建和更新集群的命令行工具。

#### （1）预备环境

* 软件环境
* * python 版本 &gt;= 2.7.9
  * aws cli 版本 &gt;= 1.16.156
  * eksctl 版本 &gt;= 0.1.37
  * kubectl 版本要求最新版本或者不低于 Kubernetes 版本 1 个次要版本号
* [角色权限](https://github.com/weaveworks/eksctl/issues/204#issuecomment-450280786)（如有管理员账号请忽略以下权限配置）
* * cloudformation 完全权限
  * eks 完全权限
  * ec2 大量权限
  * autoscaling 部分权限
  * iam 部分权限

（1）如 aws cli 版本过低，请参照以下命令更新：

```text
pip install awscli --upgrade --user
```

（2）安装 eksctl 的方式如下：

```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/download/latest_release/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

（3）安装 kubectl 的方式如下：

```bash
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
kubectl version
```

（4）所需权限列表如下：

```javascript
// Cloudformation 完全权限
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "cloudformation:*"
      ],
      "Resource": "*"
    }
  ]
}
// EKS 读写权限
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "VisualEditor0",
      "Effect": "Allow",
      "Action": [
        "eks:ListClusters",
        "eks:CreateCluster"
      ],
      "Resource": "*"
    },
    {
      "Sid": "VisualEditor1",
      "Effect": "Allow",
      "Action": [
        "eks:UpdateClusterVersion",
        "eks:ListUpdates",
        "eks:DescribeUpdate",
        "eks:DescribeCluster",
        "eks:ListClusters",
        "eks:CreateCluster"
      ],
      "Resource": "arn:aws:eks:*:*:cluster/*"
    }
  ]
}
// EC2 相关权限
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "VisualEditor0",
      "Effect": "Allow",
      "Action": [
        "ec2:CreateInternetGateway",
        "ec2:CreateVpc",
        "ec2:Describe*",
        "ec2:createTags",
        "ec2:ModifyVpcAttribute",
        "ec2:CreateSubnet",
        "ec2:CreateSubnet",
        "ec2:CreateRouteTable",
        "ec2:CreateSecurityGroup",
        "ec2:DeleteSecurityGroup",
        "ec2:AttachInternetGateway",
        "ec2:CreateRoute",
        "ec2:AuthorizeSecurityGroupIngress",
        "ec2:AuthorizeSecurityGroupEgress",
        "ec2:RevokeSecurityGroupEgress",
        "ec2:RevokeSecurityGroupIngress",
        "ec2:AssociateRouteTable",
        "ec2:CreateNatGateway",
        "ec2:AllocateAddress",
        "ec2:DeleteInternetGateway",
        "ec2:DeleteNatGateway",
        "ec2:DeleteRoute",
        "ec2:DeleteRouteTable",
        "ec2:DeleteSubnet",
        "ec2:DeleteTags",
        "ec2:DeleteVpc",
        "ec2:DescribeInternetGateways",
        "ec2:DescribeNatGateways",
        "ec2:DescribeRouteTables",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeSubnets",
        "ec2:DescribeTags",
        "ec2:DescribeVpcAttribute",
        "ec2:DetachInternetGateway",
        "ec2:DisassociateRouteTable",
        "ec2:RunInstances",
        "ec2:ReleaseAddress"
      ],
      "Resource": "*"
    }
  ]
}
// cloudwatch 相关权限
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "cloudwatch:ListMetrics",
        "cloudwatch:GetMetricStatistics",
        "cloudwatch:Describe*"
      ],
      "Resource": "*"
    },
  ]
}
// autoscaling 相关权限
{
  "Version": "2012-10-17",
  "Statement": [
      {
          "Effect": "Allow",
          "Action": [
                "autoscaling:CreateAutoScalingGroup",
                "autoscaling:DeleteAutoScalingGroup",
                "autoscaling:DeleteLaunchConfiguration",
                "autoscaling:DescribeAutoScalingGroups",
                "autoscaling:DescribeLaunchConfigurations",
                "autoscaling:DescribeScalingActivities",
                "autoscaling:UpdateAutoScalingGroup"
            ],
          "Resource": "*"
      }
  ]
}
// elasticloadbalancing 相关权限
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "elasticloadbalancing:Describe*",
      "Resource": "*"
    }
  ]
}
// iam 相关权限
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "VisualEditor0",
      "Effect": "Allow",
      "Action": [
        "iam:CreateRole",
        "iam:AttachRolePolicy",
        "iam:DetachRolePolicy",
        "iam:GetRole",
        "iam:PassRole",
        "iam:CreateInstanceProfile",
        "iam:AddRoleToInstanceProfile",
        "iam:RemoveRoleFromInstanceProfile",
        "iam:GetInstanceProfile",
        "iam:PutRolePolicy",
        "iam:DeleteRolePolicy",
        "iam:GetRolePolicy",
        "iam:ListInstanceProfiles",
        "iam:CreateServiceLinkedRole",
        "iam:ListInstanceProfilesForRole"
      ],
      "Resource": "*"
    }
  ]
}
```

> 最近在 aws 上创建 1.15 的 k8s 时发现还需要以下权限：
>
> * iam:listAttachedRolePolicies
> * ssm:GetParameter

#### （2）使用构建配置文件构建

**2.1）准备如下配置文件**

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: ecp-cafe
  region: us-west-2

vpc:
  id: vpc-0a5c6646beaff0f5d
  subnets:
    private:
      us-west-2a: { id: subnet-0805f1e738ecc03c6 }
      us-west-2b: { id: subnet-0c772d05b8ad5af94 }
      us-west-2c: { id: subnet-056ad3335f8152658 }
    public:
      us-west-2a: { id: subnet-07a0f12e5f3530a74 }
      us-west-2b: { id: subnet-0a1b8bcc3c1b2d37e }
      us-west-2c: { id: subnet-0bab65e705b561952 }
iam:
  serviceRoleARN: "arn:aws:iam::295038636998:role/EKSRole"

nodeGroups:
  - name: workers
    labels: { role: workers }
    instanceType: t3.medium
    desiredCapacity: 2
    minSize: 1
    maxSize: 5
    privateNetworking: true
    securityGroups:
      withShared: true
      withLocal: true
      attachIDs: ['sg-0a32468df4b82bbc4']
```

**2.2）执行安装**

```text
eksctl create cluster -f cluster.yaml
```

#### （3）配置 IAM 授权

```bash
curl -o aws-iam-authenticator https://amazon-eks.s3-us-west-2.amazonaws.com/1.13.7/2019-06-11/bin/linux/amd64/aws-iam-authenticator
chmod +x ./aws-iam-authenticator
mkdir -p $HOME/bin && cp ./aws-iam-authenticator $HOME/bin/aws-iam-authenticator && export PATH=$HOME/bin:$PATH
echo 'export PATH=$HOME/bin:$PATH' >> ~/.bashrc
```

#### （4）配置 kubeconfig

```bash
// 生成 kubeconfig
aws eks --region <your region> update-kubeconfig --name <cluster name>
// 查看 kubeconfig
cat ~/.kube/config
```

#### （5）安装 Dashboard

**5.1）部署 Dashboard**

```yaml
// 将 Kubernetes 控制面板部署到集群
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
// 部署 heapster 以在集群上启用容器集群监控和性能分析
kubectl apply -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/heapster.yaml
// 将 heapster 的 influxdb 后端部署到集群
kubectl apply -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/influxdb.yaml
// 为控制面板创建 heapster 集群角色绑定
kubectl apply -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/rbac/heapster-rbac.yaml
// 创建一个具有新集群管理权限的新服务账户
cat > eks-admin-service-account.yaml << EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: eks-admin
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: eks-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: eks-admin
  namespace: kube-system
EOF
// 将此服务账户和集群角色绑定应用到您的集群
kubectl apply -f eks-admin-service-account.yaml
```

要使得 Kubernetes Dashboard 可以查看 CPU 和内存指标，需要额外启用 Heapster 的 `insecure=true` 选项（不推荐在线上使用，可能有安全问题），或者使用 metrics-server。部署 metrics-server 的方式如下：

```text
helm install stable/metrics-server --namespace kube-system
```

参看：

1. [Support metrics API \#2986 - github.com](https://github.com/kubernetes/dashboard/issues/2986)
2. [教程：部署 Kubernetes Web UI \(控制面板\) - aws](https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/dashboard-tutorial.html)

**5.2）获取登录令牌**

```bash
# 检索 eks-admin 服务账户的身份验证令牌。从输出中复制 <authentication_token> 值。您可以使用此令牌连接到控制面板
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep eks-admin | awk '{print $1}')
```

**5.3）请求转发**

```bash
# 将从任意地址请求到Amazon EC2 实例 6443 端口的请求转发到 Kubernetes Dashboard 的 443 端口
kubectl port-forward --address 0.0.0.0 svc/kubernetes-dashboard -n kube-system 6443:443 &
```

**5.4）访问 Dashboard**

通过 [https://JUMPER\_IP:6443](https://jumper_ip:6443/) 访问。

#### （6）节点扩充

```text
eksctl scale nodegroup --cluster=ecp-cafe --nodes=3 --name=ecp-cafe-workers
```

#### （7）删除集群

```text
eksctl delete cluster -f cluster.yaml
```

## 容器化

### 方案概述

**采用 Alpine Linux 作为基础镜像，构建镜像推送到 Amazon ECR 仓库，然后再使用预置的 Kubernetes 部署文件部署到 Kubernetes 集群**，整个过程可以用以下活动图表示：

![](../.gitbook/assets/image%20%284%29.png)

### 相关原理

#### 容器

在讨论容器化方案之前，我们得先理清什么是容器？

所谓容器通常指 Linux 容器（LXC, Linux Containers ），这是一项轻量级的虚拟化技术。虚拟即是隔离，隔离系统资源使得不同容器中的应用互不感知，互不影响。Linux 容器与传统虚拟机技术不同的是，Linux 容器是直接运行在宿主机上的，容器间隔离的是进程，底层的实现技术是 Linux 内核的 cgroup 和 namespace 技术。

cgroup（control group）：Linux 内核中用来限制、控制与分离一个进程集的资源（包括 CPU、内存、磁盘输入输出等）。

namespace：Linux 内核中用来对全局系统资源的封装隔离。让处在不同命名空间的进程集之间资源互不可见。

#### Docker

Docker 是著名的容器封装，其 1.0 版本于 2014 年 6 月 9 日发布，目前支持 Linux 容器和 Windows 容器（Beta）。其采用 C/S 架构，由 Client、Engine 和 Registry 组成，示意图如下：

![](../.gitbook/assets/image%20%2820%29.png)

图：Docker 横向架构

Docker 因开源和 "Build, Ship, Run, Any App Anywhere" 理念的缘故，支持大量的应用，拥有庞大的开源镜像。

#### Alpine Linux

[Alpine Linux](https://alpinelinux.org/about/) 是一个基于 [musl libc](https://www.musl-libc.org/faq.html) 和 [busybox](https://www.busybox.net/FAQ.html#goals) 的安全的轻量级的 Linux 发行版，其容器镜像大小不超过 5 MB，[最新版本](https://hub.docker.com/_/alpine/scans/library/alpine/3.10.2)仅 2.7 MB。Alpine Linux 使得我们构建 Linux 容器化应用的体积大大缩小。

musl libc：是一个通用的 C 库（libc，ISO C 和 POSIX 标准中描述的标准库）实现，轻量、快速、简单、自由并专注于标准的一致性和安全。

busybox：标准 Linux 命令行工具的一种实现，小、简单和正确。

#### Amazon ECR

Amazon ECR\(Amazon Elastic Container Registry \) 是完全托管的安全 Docker 容器注册表，可使开发人员轻松存储、管理和部署 Docker 容器映像。关于 ECR 的更多知识请访问 [Amazon ECR 用户指南](https://docs.aws.amazon.com/zh_cn/AmazonECR/latest/userguide/what-is-ecr.html)。

### 实现过程

#### （1）构建 Go 应用可执行文件

如未安装 Go，请参照以下命令在 EC2 中安装：

```bash
# install go
wget https://dl.google.com/go/go1.12.7.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.12.7.linux-amd64.tar.gz

# config env in file `~/.bash_profile`
export PATH=$PATH:/usr/local/go/bin
```

随后执行以下命令构建 Go 应用。

```bash
export GO111MODULE=on
export GOOS=linux
export GOARCH=amd64
export CGO_ENABLED=0
export BUILD_TAG=dev
go build -o main -tags ${BUILD_TAG}
```

#### （2）构建 Docker 镜像

如未安装 Docker，请参照以下命令在 EC2 中安装：

```bash
# install docker，见：https://docs.aws.amazon.com/zh_cn/AmazonECS/latest/developerguide/docker-basics.html
# ！需要使用 Amazon Linux AMI 2 镜像的实例
sudo yum update -y
sudo amazon-linux-extras install docker
sudo service docker start
sudo usermod -a -G docker ec2-user
# 退出重新登录，然后执行以下命令
docker info
```

具备 Docker 环境后，准备以下 Dockerfile（以 ecp-cafe/mocha 服务为例）：

```bash
# Dockerfile References: https://docs.docker.com/engine/reference/builder/

# Start from the latest golang base image
# golan dockerhub: https://hub.docker.com/_/golang
FROM golang:1.12.7-alpine3.10  AS builder

# Set the Current Working Directory inside the container
WORKDIR /app

# Copy go mod and sum files
COPY go.mod go.sum main ./

######## Start a new stage from scratch #######
FROM alpine:3.10

WORKDIR /root/

# Copy the Pre-built binary file from the previous stage
COPY --from=builder /app/main .

# Expose port 8080 to the outside world
EXPOSE 8080

# Command to run the executable
CMD ["./main"]
```

随后执行以下命令，进行 Docker 镜像构建。

```bash
export APP=mocha
docker build -t ${APP} . --no-cache=true
```

#### （3）推送到镜像仓库

这里使用 Amazon ECR 作为我们的镜像仓库，在推送镜像前，需要确保以下条件：

1. 在 ECR 控制台创建了相应的 Repository，这里预建了名为 ecp-cafe/mocha 的 Repository。
2. 具备 ECR 的必要权限。
3. 需要使用 aws cli 工具登录 ecr。

ECR 的必要权限包括：

```text
"ecr:GetAuthorizationToken",
"ecr:GetDownloadUrlForLayer",
"ecr:BatchGetImage",
"ecr:BatchCheckLayerAvailability",
"ecr:PutImage",
"ecr:InitiateLayerUpload",
"ecr:UploadLayerPart",
"ecr:CompleteLayerUpload"
```

执行以下命令完成镜像打标签和推送。

```bash
export DOCKER_REPO="295038636998.dkr.ecr.us-west-2.amazonaws.com/ecp-cafe
# 登录 ECS
eval $(aws ecr get-login --region us-west-2 --no-include-email)
# 镜像打标签
IMAGE_TAG="`date +%Y%m%d%H%M`"
docker tag ${APP} ${DOCKER_REPO}/${APP}:${IMAGE_TAG}
# 推送镜像
docker push ${DOCKER_REPO}/${APP}:${IMAGE_TAG}
```

#### （4）部署到 Kubernetes 集群

准备以下部署文件（以 ecp-cafe/mocha 为例），假定放置于 ci/deploy 文件夹中。

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ecp-cafe
---
kind: Service
apiVersion: v1
metadata:
  name: mocha
  namespace: ecp-cafe
  labels:
    run: mocha
spec:
  ports:
  - protocol: TCP
    port: 8080
    targetPort: 8080
  selector:
    run: mocha
  type: LoadBalancer
  sessionAffinity: None
  externalTrafficPolicy: Cluster
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: mocha
  namespace: ecp-cafe
  labels:
    run: mocha
spec:
  replicas: 1
  selector:
    matchLabels:
      run: mocha
  template:
    metadata:
      labels:
        run: mocha
    spec:
      containers:
      - name: mocha
        image: 295038636998.dkr.ecr.us-west-2.amazonaws.com/ecp-cafe/mocha:latest
        ports:
        - containerPort: 8080
          protocol: TCP
        imagePullPolicy: Always
      restartPolicy: Always
      terminationGracePeriodSeconds: 60
      dnsPolicy: ClusterFirst
      schedulerName: default-scheduler
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 25%
      maxSurge: 25%
```

随后执行以下命令，服务就可以部署到 Kubernetes 集群中了。

```yaml
# 应用配置
kubectl apply -f ci/deploy/
# 更新镜像
kubectl set image deployments/${APP} ${APP}=${DOCKER_REPO}/${APP}:${IMAGE_TAG} --namespace=ecp-cafe
```

## CI/CD

### 方案概述

**采用 Jenkins +** [**Git Parameter Plugin**](https://wiki.jenkins.io/display/JENKINS/Git+Parameter+Plugin) **+ Gitlab 进行 CI/CD，当通过 Jenkins Dashboard 发出构建请求后，打包机会执行上述容器化过程（即构建应用、构建镜像、发布镜像和部署服务）。**整个过程可用下面的时序图表示：

![](../.gitbook/assets/image%20%2835%29.png)

### 相关原理

#### Jenkins

Jenkins 是一款跨平台的持续集成和持续交付（CI/CD, continuous integration and continuous delivery）应用，是一个 Java 应用，依赖 JVM 环境。Jenkins 的默认工作目录是：/var/lib/jenkins/workspace/，默认用户是：Jenkins。

[Git Parameter Plugin](https://wiki.jenkins.io/display/JENKINS/Git+Parameter+Plugin) 是 Jenkins 的一款选择 Git Branch 和 Tag 的插件，以便对指定的 Branch 或 Tag 进行构建。

有关 Jenkins 的更多知识，见[官方教程](https://jenkins.io/doc/pipeline/tour/getting-started/)。

### 实现过程

#### （1） 配置 Gitlab 用户

执行以下命令完成 Gitlab 用户的配置。

```text
GIT_USER=
GIT_PSW=
echo 'https://${GIT_USER}:${GIT_PSW}@git.mobisummer.com' >> ~/.git-credentials
git config --global credential.helper store
```

#### （2）安装 Jenkins 并进行必要配置

**2.1） 确保安装 JDK8 或以上**

如未安装 JDK8，请参照以下命令：

```bash
# install jdk8
sudo yum install java-1.8.0-openjdk-devel
# remove jdk7
sudo yum remove java-1.7.0-openjdk
```

**2.2）安装并启动 Jenkins**

执行以下命令安装并启动 Jenkins：

```bash
# install jenkins
sudo wget -O /etc/yum.repos.d/jenkins.repo http://pkg.jenkins.io/redhat/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat/jenkins.io.key
sudo yum install jenkins -y
# start jenkins
sudo service jenkins start
```

**2.3） 访问 Jenkins Dashboard**

访问 [http://JUMPER\_URL:8080](http://localhost:8080/)，选择标准安装即可。

**2.4） 配置 Jenkins 插件**

最后，通过 Jenkins → 插件管理 → 搜索框，查找并安装插件[Extended Choice Parameter Plug-In](http://wiki.jenkins-ci.org/display/JENKINS/Extended+Choice+Parameter+plugin)。

**2.5\) 配置环境变量**

系统管理-&gt;系统设置，添加以下环境变量

LANG=zh.CH.UTF-8

PYTHONIOENCODING=UTF8

PATH=/[sbin:/usr/sbin:/bin:/usr/bin:/usr/local/bin:/usr/local/go/bin/:/home/ec2-user/bin](http://sbin/usr/sbin:/bin:/usr/bin:/usr/local/bin:/usr/local/go/bin/:/home/ec2-user/bin)

**2.6） 配置 Jenkins 用户**

默认的 Jenkins 用户为 Jenkins ，很多命令都很受限，无法执行。特别是当在 aws EC2 中使用时，AWS 的服务角色授权是授予 ec2-user 的，其他用户（包括 root）都无法正常使用 AWS CLI 相关命令。更改 Jenkins 用户的步骤如下（以 ec2-user 为例）：

首先，编辑 Jenkins 配置，设置 JENKINS\_USER：

```bash
vim /etc/sysconfig/jenkins
```

找到并修改 JENKINS\_USER="ec2-user"。

接着，修改 Jenkins 相关文件权限：

```bash
sudo chown -R ec2-user:ec2-user /var/lib/jenkins
sudo chown -R ec2-user:ec2-user /var/cache/jenkins
sudo chown -R ec2-user:ec2-user /var/log/jenkins
```

最后，重启服务：

```bash
sudo service jenkins restart
```

#### （3）配置 Jenkins 任务

Jenkins → 新建任务 → 输入一个任务名称 → 点选 "构建一个自由风格的软件项目" → 点击 "确定"，随后会进入任务配置页面。

1. 在 "General" TAG，选中 "参数化构建过程" → 在 "添加参数" 下拉选框中选中 "Git Parameter" → Name 处输入 "BRANCH"，"Parameter Type" 下拉选框选中 "Branch"。"Default Value" 处输入 "origin/master"。
2. 在 "源码管理" TAB，选择 Git，输入项目的仓库地址并添加 Git 用户。
3. 在 "构建" TAB，在 "增加构建步骤" 下拉选框中选中 "执行 shell"，输入以下命令：

   ```bash
   # 这里是 ecp-cafe/mocha 项目需要，其它项目根据需要变更
   export BUILD_TAG=dev
   # 以下脚本在步骤(4) 中实现，位于项目的根目录
   ./jenkins.sh
   ```

#### （4）配置打包脚本

jenkins.sh 文件内容如下：

```bash
#!/bin/bash
############### 应用配置 ###############
APP=`cat ci/gradle.properties | grep ARTIFACT | awk -F'=' '{ print $2 }'`
echo "APP=$APP"
# 构建标签, 需在环境变量中设置
if [ $BUILD_TAG ];then
    BUILD_TAG=${BUILD_TAG}
else
    echo "BUILD_TAG 未设置，请在环境变量中设置，可选值为：local | dev | test | prod"
  exit 1
fi
echo "BUILD_TAG=$BUILD_TAG"

# 镜像标签,读取 ci/gradle.properties 下的 VERSION
VERSION="`date +%Y%m%d%H%M`"
IMAGE_TAG=`cat ci/gradle.properties | grep VERSION | awk -F'=' '{ print $2 }'`"-"$VERSION
echo "IMAGE_TAG=$IMAGE_TAG"

############### 部署 ###############
export DOCKER_REPO="295038636998.dkr.ecr.us-west-2.amazonaws.com/ecp-cafe"
export GO111MODULE=on
export GOOS=linux
export GOARCH=amd64
# 以下如不设置，则在运行时会报：exec user process caused "no such file or directory"
export CGO_ENABLED=0

## 清理未使用的本地镜像
docker image prune -a -f

## 构建镜像
go build -o main -tags ${BUILD_TAG}
docker build -t ${APP} . --no-cache=true
## 登录 ECS
eval $(aws ecr get-login --region us-west-2 --no-include-email)
## 推送镜像
docker tag ${APP} ${DOCKER_REPO}/${APP}:${IMAGE_TAG}
docker push ${DOCKER_REPO}/${APP}:${IMAGE_TAG}

docker tag ${APP} ${DOCKER_REPO}/${APP}:latest
docker push ${DOCKER_REPO}/${APP}:latest

## 确保 ecp-cafe 命名空间
kubectl apply -f ci/deploy/ecp-cafe-ns.yaml

## 对 ecp-cafe 自动 sidecar 注入
kubectl label namespace ecp-cafe istio-injection=enabled

## 应用部署配置
kubectl apply -f ci/deploy/${BUILD_TAG}
## 更新镜像
kubectl set image deployments/${APP} ${APP}=${DOCKER_REPO}/${APP}:${IMAGE_TAG} --namespace=ecp-cafe
```

## 服务发现

### 方案概述

**Kubernetes 1.13 版本默认集成了 CoreDNS 作为内部服务发现的组件，对集群内部的服务我们可以直接通过 "\[命名空间.\]服务名" 方式访问，对外部的服务我们可以通过设置 ExternalName 或者定义存根域和上游 DNS 的方式实现。**集群的 DNS 解析过程如下：

![](../.gitbook/assets/image%20%288%29.png)

### 相关原理

#### 服务发现

**服务发现就是一种提供服务发布和查找的服务**，是基于服务架构（SOA）的核心服务，需具备以下关键特性：

1. 注册（Registration），新增服务到服务列表；
2. 目录（Directory），即服务列表；
3. 查找（Lookup），通过服务名找到服务。

**服务发现的关键在于服务元数据（metadata）的存储**，包括服务名、服务 IP、服务端口等信息。

Kubernetes 支持两种服务发现方式，环境变量和 DNS。

#### Kubernetes 服务环境变量

Kubernetes Service 环境变量形如（假定服务名为 latte，且访问端口为 8080）：

```text
LATTE_SERVICE_HOST=10.100.251.57
LATTE_SERVICE_PORT=8080
```

Docker Link 环境变量形如（假定服务名为 latte，且访问端口为 8080）：

```text
LATTE_PORT_8080_TCP_ADDR=10.100.251.57
LATTE_PORT_8080_TCP_PORT=8080
LATTE_PORT_8080_TCP_PROTO=tcp
LATTE_PORT=tcp://10.100.251.57:8080
LATTE_PORT_8080_TCP=tcp://10.100.251.57:8080
```

此种方式的服务发现缺点很明显：

1. 先前的服务必须先运行起来，否则 Pod 无法发现；
2. 如依赖的服务宕机或绑定新地址，Pod 无法发现，仍然持有旧的地址。

#### Linux 中的 DNS 查询原理

在 Linux 的 `/etc/` 目录中，存在 3 个我们需要关注的文件，分别是（参考：[http://man7.org/linux/man-pages/man5/host.conf.5.html）：](http://man7.org/linux/man-pages/man5/host.conf.5.html）：)

1. `/etc/host.conf`：DNS 解析器配置，包含 trim、multi、order、reorder 和 nospoof 等等配置。
2. `/etc/hosts`：本地 hosts 数据库，存放本地的域名到 IP 的配置。
3. `/etc/resolv.conf`：DNS 解析器配置，包含 nameserver、domain、search、sortlist 和 options 等配置。

下面是一台 Linux 服务器中 3 个相关文件的内容：

```text
# /etc/host.conf
multi on
# /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost6 localhost6.localdomain6
# /etc/resolv.conf
; generated by /usr/sbin/dhclient-script
search us-west-2.compute.internal
options timeout:2 attempts:5
nameserver 192.168.0.2
```

`/etc/resolv.conf` [配置](http://man7.org/linux/man-pages/man5/resolv.conf.5.html)解释如下：

| 配置项 | 功能 | 备注 |
| :--- | :--- | :--- |
| nameserver | DNS 服务器 | 值必须是 IP 地址 |
| domain | 本地域名 | 域中的查询可以使用相对于本地域名的短名称 |
| search | 主机名查询列表 | 默认只包含本地域名。阈值为 6 个域名，256 个字符。 |
| options | 选项 | 修改内部 DNS 解析器变量值。 |

options 中常见的配置项有：

| 配置项 | 功能 |
| :--- | :--- |
| ndots | 所有查询中，如果`.`的个数少于给定的数值，则会根据`search`中配置的列表依次在对应域中先进行搜索，如果没有返回，则最后再直接查询域名本身。阈值为 15。 |
| timeout | 等待 DNS 服务器响应的超时时间，单位为秒。阈值为 30 s。 |
| attempts | 向同一个 DNS 服务器发起重试的次数，单位为次。阈值为 5。 |

#### Kubernetes 中的 DNS 查询原理

**Kubernetes 通过修改每个 Pod 中每个容器的域名解析配置文件 `/etc/resolv.conf` 来达到服务发现的目的**。

在笔者创建的集群中获取其中一个容器的域名解析配置文件如下：

```text
# /etc/resolv.conf
nameserver 10.100.0.10
search cafe.svc.cluster.local svc.cluster.local cluster.local us-west-2.compute.internal
options ndots:5
```

其含义是：DNS 服务器为 10.100.0.10，当查询关键词中 `.` 的数量少于 5 个，则根据 search 中配置的域名进行查询，当查询都没有返回正确响应时再尝试直接查询关键词本身。

例如，执行 `host -v cn.bing.com` 后我们将会看到：

```text
Trying "cn.bing.com.cafe.svc.cluster.local"
Trying "cn.bing.com.svc.cluster.local"
Trying "cn.bing.com.cluster.local"
Trying "cn.bing.com.us-west-2.compute.internal"
Trying "cn.bing.com"
...
```

**Kubernetes 的 DNS 服务（简称为 kube-dns）支持 Service 的 A 记录、 SRV 记录和 CNAME 记录**。可以通过以下方式查询：

```yaml
注：查询 DNS 记录的方法
（1）安装 dig 工具
将下面的部署配置保存到文件 dnsutils.yaml，然后执行 kubectl apply -f dnsutils.yaml 部署。
apiVersion: v1
kind: Pod
metadata:
name: dnsutils
namespace: default
spec:
containers:
  - name: dnsutils
 image: tutum/dnsutils
 command:
      - sleep
      - "3600"
 imagePullPolicy: IfNotPresent
restartPolicy: Always

（2）使用 dig 工具获取 DNS 记录
# 用法
dig @<DNS服务器> <记录类型> <域名> <可选值>
# 示例
## 1）获取 DNS 服务地址
kubectl get svc kube-dns -n kube-system
NAME       TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)         AGE
kube-dns   ClusterIP   10.100.0.10   <none>        53/UDP,53/TCP   8d
## 2）进入 dnsutils 的 shell 终端
kubectl exec -ti dnsutils  sh
## 3）查询 latte 服务的 A 记录
dig @10.100.0.10 A  latte.cafe.svc.cluster.local  +noall +answer
```

### 实现过程

#### （1）将外部服务 ExternalName 化

假定集群使用了一个外部的数据库服务，这时我们可以使用以下部署配置将其 ExternalName 化：

```yaml
kind: Service
apiVersion: v1
metadata:
  name: dev-mongo
  namespace: cafe
spec:
  type: ExternalName
  externalName: cafe.cluster-cps9klrdqize.us-west-2.docdb.amazonaws.com
```

```text
ExternalName 要求以下的字符规则： '[a-z0-9]([-a-z0-9]*[a-z0-9])?(\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)*'
```

随后 kube-dns 会为其生成以下的 CNAME 记录：

```text
dev-mongo.cafe.svc.cluster.local. 10 IN CNAME cafe.cluster-cps9klrdqize.us-west-2.docdb.amazonaws.com.
```

集群内部服务只需要使用 dev-mongo 即可调用到外部服务。

## 配置管理

### 方案概述

**采用 Kubernetes 的 ConfigMap 资源对象定义配置，随后通过挂载卷方式读取。配置变更通过 Gitlab + Jenkins 实现自动化部署**，时序图如下**：**

![](../.gitbook/assets/image%20%286%29.png)

### 相关原理

#### ConfigMap

ConfigMap 是 Kubernetes 内置的资源对象，用于保存配置数据的键值对，可以用来保存单个属性，也可以用来保存配置文件。

更新 ConfigMap 后：

* 使用该 ConfigMap 挂载的 Env 不会同步更新
* 使用该 ConfigMap 挂载的 Volume 中的数据需要一段时间（实测大概10秒）才能同步更新

ENV 是在容器启动的时候注入的，启动之后 kubernetes 就不会再改变环境变量的值，且同一个 namespace 中的 pod 的环境变量是不断累加的。为了更新容器中使用 ConfigMap 挂载的配置，需要通过滚动更新 pod 的方式来强制重新挂载 ConfigMap。

#### GitLab CI/CD

Gitlab 支持读取项目根目录的 .gitlab-ci.yml 文件，然后当代码变更时自动执行构建。更多知识请访问 [Getting started with GitLab CI/CD](https://docs.gitlab.com/ee/ci/quick_start/)。

#### Gitlab Webhooks

Gitlab Webhooks 支持项目内的事件推送，包括 Push Event、Tag Push Event、Comments 等等，通过 Gitlab → Settings → Integrations 进行配置，更多支持请访问：[Webhooks - gitlab.com](https://docs.gitlab.com/ee/user/project/integrations/webhooks.html)。

#### Jenkins Pipeline

Jenkins Pipeline 支持读取项目根目录下的 Jenkinsfile 文件，然后执行构建。更为关键的是，Jenkins 支持远程触发构建。更多知识请访问[ How to trigger a Jenkins job remotely](https://arghya.xyz/articles/triggering-jenkins-job-remotely/) 及 [创建您的第一个Pipeline](https://jenkins.io/zh/doc/pipeline/tour/hello-world/)。

### 实现过程

#### （1）准备配置文件（假定为 mocha-cm.yaml）

```text
AWS_REGION: us-west-2
APP: mocha
```

#### （2）创建 ConfigMap 对象

```text
kubectl create cm mocha-cm --from-file=./mocha-cm.yaml -n ecp-cafe
```

#### （3）查看创建的 ConfigMap

运行以下命令来查看刚才创建的 ConfigMap：

```text
kubectl get cm mocha-cm -n ecp-cafe -o yaml
```

可得如下配置：

```yaml
apiVersion: v1
data:
  mocha-cm.yaml: |
    AWS_REGION: us-west-2
    APP: mocha
kind: ConfigMap
metadata:
  name: mocha-cm
  namespace: ecp-cafe
```

#### （4）挂载到 Deployment

修改应用（此处以 mocha 为例）的 Deployment 文件，如下，将配置文件挂载到 /config 目录，随后应用中可以通过 /config/mocha-cm.yaml 访问挂载的配置文件。

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    run: mocha
  name: mocha
  namespace: ecp-cafe
spec:
  replicas: 1
  selector:
    matchLabels:
      run: mocha
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        run: mocha
    spec:
      containers:
      - name: mocha
        image: 295038636998.dkr.ecr.us-west-2.amazonaws.com/ecp-cafe/mocha:0.1.4-201908290353
        imagePullPolicy: Always
        volumeMounts:
        - name: config-volume
          mountPath: /config
        ports:
        - containerPort: 8080
          protocol: TCP
        resources: {}
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      volumes:
        - name: config-volume
          configMap:
            name: mocha-cm
```

#### （4）更新配置

修改配置文件内容，然后执行以下命令：

```text
kubectl create configmap --dry-run -o yaml -n ecp-cafe --from-file=./mocha-cm.yaml mocha-cm | kubectl apply -n ecp-cafe -f -
```

#### （5）触发应用更新

当更新配置时，应用并不感知，因此需要通过修改 Deployment 注解的方式，触发 Deployment Rolling Update，从而达到更新应用配置的目的。简单来说，执行以下命令就好了。

```text
kubectl patch deployment mocha -n ecp-cafe --patch '{"spec": {"template": {"metadata": {"annotations": {"version/config": "'$(date +%s)'" }}}}}'
```

#### （6）使用 Jenkins 部署配置

Jenkins → 新建任务 → 输入一个任务名称 → 点选 "构建一个自由风格的软件项目" → 点击 "确定"，随后会进入任务配置页面。

1. 在 "General" TAG，选中 "参数化构建过程" → 在 "添加参数" 下拉选框中选中 "Git Parameter" → Name 处输入 "BRANCH"，"Parameter Type" 下拉选框选中 "Branch"。"Default Value" 处输入 "origin/master"。
2. 在 "源码管理" TAB，选择 Git，输入配置项目的仓库地址并添加 Git 用户。
3. 在 "构建触发器"，选择 "触发远程构建（例如，使用脚本）"，输入身份验证令牌，随后我们可以通过 [http://JUMPER\_IP:8080/job/mocha-config-deploy/buildWithParameters?token=](http://jumper_ip:8080/job/mocha-config-deploy/buildWithParameters?token=)TOKEN\_NAME 执行远程构建。
4. 在 "构建" TAB，在 "增加构建步骤" 下拉选框中选中 "执行 shell"，输入以下命令：

   ```text
   python gitlab_deploy_config.py
   ```

   gitlab\_deploy\_config.py 放置在配置项目的根目录，其内容为：

   ```text
   from __future__ import print_function
   import sys
   import yaml
   import sys
   import os
 
 
   def eprint(*args, **kwargs):
       print(*args, file=sys.stderr, **kwargs)
 
 
   def main():
       for _, _, files in os.walk("./config"):
           print ("Found following files:")
           for filename in files:
               print (filename)
               result = yaml_verify("./config/" + filename)
               if result is False:
                   sys.exit(1)
                   return
       cmd = 'kubectl create configmap --dry-run -o yaml -n ecp-cafe --from-file=./config/mocha-cm.yaml mocha-cm | kubectl replace -n ecp-cafe -f -'
       os.system(cmd)
       cmd = 'kubectl patch deployment mocha -n ecp-cafe --patch \'{"spec": {"template": {"metadata": {"annotations": {"version/config": "\'$(date +%s)\'" }}}}}\''
       os.system(cmd)
 
 
   def yaml_verify(filename):
       try:
           fp = open(filename)
       except IOError as ioerr:
           eprint("ERROR: Failed to open [%s] [%s]" %
                  (filename, ioerr))
           return False
 
       try:
           yaml.safe_load(fp)
           return True
       except yaml.error.YAMLError as yamlerr:
           eprint("ERROR: Yaml parsing failed [file: %s] [%s]" %
                  (filename, yamlerr))
           return False
       return True
 
 
   if __name__ == '__main__':
       main()
   ```

#### （7）使用 Gitlab Webhooks 自动构建

在项目下找到 Settings → Integrations ，然后按以下步骤配置：

1. 在 URL 处输入 Jenkins 的远程构建链接，示例为 "[http://JUMPER\_IP:8080/job/mocha-config-deploy/buildWithParameters?token=TOKEN\_NAME](http://JUMPER_IP:8080/job/mocha-config-deploy/buildWithParameters?token=TOKEN_NAME)"。
2. 在 Trigger 处勾选 "Push events"，输入触发的分支为 "master" 。
3. 点击 "Save changes"。

为了使 Gitlab 可以正常调用需要满足以下条件：

1. 确保 Gitlab 可以正常访问 Jenkins 的远程端口。
2. 确保Jenkins 上的 "系统设置 → 全局安全配置 → 跨站请求伪造保护" 处于关闭状态。

随后，master 分支上的配置修改就会触发 Jenkins 的远程构建了。

## 负载均衡

### 方案概述

**采用 Istio Gateway 实现 HTTP 和 gRPC 请求的服务端负载均衡。**假定部署了 mocha 和 latte 服务，mocha 服务对外暴露了 HTTP 接口，mocha 与 latte 之间通过 gRPC 进行通信，其数据流图如下：

![](../.gitbook/assets/image%20%287%29.png)

注意：这里即使只有一个 Pod 连接 latte，latte 仍能做到请求的负载均衡而非连接的负载均衡。

### 相关原理

#### 负载均衡（Load Balancing）

负载均衡是指能将客户端的请求均衡地负载到服务的各个节点（计算、网络或存储资源），可分为客户端的负载均衡和服务端的负载均衡，还可分为按连接的负载均衡和按请求的负载均衡。

客户端的负载均衡依赖于对客户端的信任，一旦客户端暴走，服务端很可能出现节点过载。按连接的负载均衡一旦连接建立后，后续的请求都会交由特定的节点进行处理，请求暴增时，服务端也很可能出现节点过载。因此我们希望做的是**服务端的按请求的负载均衡**。

#### Istio

Istio 是服务网格的经典实现，用来管理集群中的流量。

#### Istio 负载均衡

Istio 中的 Pilot 组件使用来自服务注册（由 Kubernetes 提供）的信息，并提供与平台无关的服务发现接口。网格中的 Envoy 实例执行服务发现，并相应地动态更新其负载均衡池。

![](../.gitbook/assets/image%20%281%29.png)

图片来源：[流量管理 - istio.io/zh](https://archive.istio.io/v1.2/zh/docs/concepts/traffic-management/)

实际使用时需要配置 Gateway 资源对象。要使用 Istio 的负载均衡功能，需要把我们的流量从 Gateway 中流入到设定的 VirtualService，从而导向我们的微服务。理清下 Istio Gateway 相关的几个 CRD 的含义：

1. VirtualService：设定路由规则；
2. DestinationRule：设定流量策略，包括熔断、限流和负载均衡策略；
3. ServiceEntry：注册网格外服务；
4. Gateway：网关，负载均衡器，指定要开放的端口、协议和 SNI 配置（SSL/TLS 证书绑定相关）。

在整个服务网格的视点，我们发现整个服务网格的入口由 Gateway 定义，出口由 ServiceEntry 定义，内部的网格路由由 VirtualService 定义，内部的网格流量由 DestinationRule 定义。

Istio 支持 3 种负载均衡算法，分别是：

1. 轮询（ROUND\_ROBIN）
2. 最小连接（LEAST\_CONN）
3. 随机（RANDOM）

#### gRPC

gRPC 是 Google 推出的一款基于 HTTP/2 协议支持异构系统 RPC 通信的依赖库，其使用 [protocol buffers](https://developers.google.com/protocol-buffers/docs/overview) 作为 RPC 消息和方法的定义语言。

protocol buffers 目前有两个版本，v2 和 v3，在没有历史包袱的情况下，我们选用 v3 。

关于 proto3（即 protocol buffers v3） 的语法，见 [https://developers.google.com/protocol-buffers/docs/reference/proto3-spec](https://developers.google.com/protocol-buffers/docs/reference/proto3-spec)。

关于如何在快速在 Go 中使用 gRPC，见 [https://grpc.io/docs/tutorials/basic/go/](https://grpc.io/docs/tutorials/basic/go/)。

### 实现过程

#### （1）httpbin 服务的 HTTP 请求负载均衡

**1.1）部署 httpbin 应用**

```text
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.2/samples/httpbin/httpbin.yaml
```

**1.2）确定入口 IP 和端口**

```text
kubectl get svc istio-ingressgateway -n istio-system
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].port}')
```

**1.3）配置 Gateway**

以下配置定义了一个名为 httpbin-gateway 的 Gateway 资源，采用 Istio 的 gateway 默认实现，暴露 host 为 "[httpbin.example.com](http://httpbin.example.com/)" 端口为 80 的 VirtualService。

```yaml
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: httpbin-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "httpbin.example.com"
EOF
```

**1.4）配置 VirtualService**

以下配置定义了一个名为 httpbin 的 VirtualService，其绑定到名为 httpbin-gateway 的 Gateway，绑定名为 "[httpbin.example.com](http://httpbin.example.com/)" 的 host，将 host 的 /status 和 /delay 路径路由到集群中名为 httpbin，端口为 8000 的服务。

```yaml
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
  - "httpbin.example.com"
  gateways:
  - httpbin-gateway
  http:
  - match:
    - uri:
        prefix: /status
    - uri:
        prefix: /delay
    route:
    - destination:
        port:
          number: 8000
        host: httpbin
EOF
```

**1.5）使用 curl 访问 httpbin 服务**

在配置了 DNS 解析的情况下我们通过 [http://httpbin.example.com/status/200](http://httpbin.example.com/status/200) 即可访问我们的服务，但是在没有配置 DNS 记录的情况下，可以通过 -H 配置请求头的方式访问。命令如下：

```bash
curl -I -HHost:httpbin.example.com http://$INGRESS_HOST:$INGRESS_PORT/status/200
```

**1.6）使用浏览器访问 httpbin 服务**

下面我们将 httpbin-gateway 的 host 绑定开放化，然后将到路径 /headers 的请求路由到集群中名为 httpbin 的服务的 8000 端口。

```yaml
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: httpbin-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
  - "*"
  gateways:
  - httpbin-gateway
  http:
  - match:
    - uri:
        prefix: /headers
    route:
    - destination:
        port:
          number: 8000
        host: httpbin
EOF
```

接下来只需要通过 [http://$INGRESS\_HOST:$INGRESS\_PORT/headers](http://$INGRESS_HOST:$INGRESS_PORT/headers) 就可以访问了。

**1.7）查看 Kiali Dashboard**

![](../.gitbook/assets/image%20%2815%29.png)

#### （2）testrpc 服务的 gRPC 负载均衡

**2.1）部署 testrpc & testrpcc 服务**

保存以下配置到文件 deploy\_test\_rpc.yaml，然后执行 kubectl apply -f deploy\_test\_rpc.yaml 部署服务。

该配置会产生一个有两个 Pod 名为 testrpc 的 RPC 服务端和有一个 Pod 的名为 testrpcc 的 RPC 客户端。testrpcc 会不停地请求 testrpc。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: testrpc
  labels:
    app: testrpc
spec:
  ports:
  - name: grpc
    port: 10000
    targetPort: 10000
  selector:
    app: testrpc
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: testrpc
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: testrpc
        version: v1
    spec:
      containers:
      - image: docker.io/lshare/testrpc
        imagePullPolicy: IfNotPresent
        name: testrpc
        ports:
        - containerPort: 10000
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: testrpcc
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: testrpcc
        version: v1
    spec:
      containers:
      - image: docker.io/lshare/testrpcc
        imagePullPolicy: IfNotPresent
        name: testrpcc
```

**2.2）配置 Gateway**

保存以下配置到文件 deploy\_rpc\_gateway.yaml，然后执行 kubectl apply -f deploy\_rpc\_gateway.yaml 部署 Gateway。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: testrpc-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 10000
      name: grpc
      protocol: GRPC #or GRPC, which gives the same result
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: testrpc
spec:
  hosts:
  - testrpc
  gateways:
  - testrpc-gateway
  http:
  - route:
    - destination:
        host: testrpc
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: testrpc
spec:
  host: testrpc
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN
```

**2.4）查看 Kiali Dashboard**

![](../.gitbook/assets/image%20%2836%29.png)

## 灰度发布

### 方案概述

**采用 Istio Gateway 对外暴露服务，然后修改 VirtualService 资源，控制 http rout 中各个 destination 的 weight 即可。**

### 相关原理

Istio Gateway 使用 VirtualService 定义路由规则，关于 VirtualService 的更多知识参见 [Istio 官方文档](https://istio.io/docs/reference/config/networking/v1alpha3/virtual-service/)。

### 实现过程

以下配置定义 50% 权重的流量流到v1版本，50% 权重的流量流入v2版本。

```yaml
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 50
    - destination:
        host: reviews
        subset: v3
      weight: 50
```

各个 destination 的 weight 的总和**必须是** 100。参看 [https://istio.io/docs/reference/config/networking/v1alpha3/virtual-service/](https://istio.io/docs/reference/config/networking/v1alpha3/virtual-service/)

## 服务熔断

### 方案概述

**采用 Istio + Envoy 的服务网格方案，在原先的 Pod 中注入 Envoy 代理（即所谓的边车注入），从而达到管理流量的目的，包括服务熔断。**

### 相关原理

#### Istio 中的熔断配置

Istio 中的熔断配置有两种：

1. 连接池配置；
2. 主机隔离配置。

**连接池配置**

连接池配置可以配置本服务的 HTTP 和 TCP 连接。当限制达到时，多余的请求将会被熔断，或者说阻止。

| 字段 | 类型 | 描述 |
| :--- | :--- | :--- |
| maxConnections | int32 | HTTP1/TCP 连接的阈值，默认为 1024 |
| connectTimeout | [google.protobuf.Duration](https://developers.google.com/protocol-buffers/docs/reference/google.protobuf#duration) | TCP 连接超时 |
| tcpKeepalive | [ConnectionPoolSettings.TCPSettings.TcpKeepalive](https://istio.io/docs/reference/config/networking/v1alpha3/destination-rule/#ConnectionPoolSettings-TCPSettings-TcpKeepalive) | TCP keep alive |

表：TCP 连接池配置

| 字段 | 类型 | 描述 |
| :--- | :--- | :--- |
| `http1MaxPendingRequests` | `int32` | HTTP/1.1 等待请求的阈值，默认为 1024 |
| `http2MaxRequests` | `int32` | 给定时间内后端的 HTTP/2 请求阈值，默认为 1024 |
| `maxRequestsPerConnection` | `int32` | 每个 HTTP/2 连接的请求阈值。设置为 1 时会禁用 keep alive。 |
| `maxRetries` | `int32` | 给定时间内进行重试的次数阈值，默认为 1024 |
| `idleTimeout` | `google.protobuf.Duration` | 空闲超时。当连接没有活跃的请求时生效。默认没有设置。通过对 HTTP/1.1 和 HTTP/2 生效。 |

表：HTTP 连接池配置

**主机隔离配置**

主机隔离配置可以配置对服务的上游主机进行定时检查，并通过配置的规则判定主机的健康情况、驱逐不健康的主机以及自动恢复驱逐的主机。

| 字段 | 类型 | 描述 |
| :--- | :--- | :--- |
| `consecutiveErrors` | `int32` | 单个主机能容忍的连续错误的个数，超出将会被驱逐。默认值是 5。对于 HTTP 连接，返回码为 502、503 或 504 视为错误；对于 TCP 连接，连接超时和连接失败事件视为错误。 |
| `interval` | `google.protobuf.Duration` | 两次扫描分析主机的时间间隔。格式为 1h/1m/1s/1ms，必须 &gt;=1ms。默认值时 10s。 |
| `baseEjectionTime` | `google.protobuf.Duration` | 最小驱逐时间。一个主机的驱逐期 = 最小驱逐时间 x 驱逐次数。该技术允许系统自动增加不健康上游主机的驱逐周期。格式：1h/1m/1s/1ms。必须 &gt;= 1ms。默认值为30秒。 |
| `maxEjectionPercent` | `int32` | 负载平衡池中可以驱逐的上游服务的最大主机百分比。默认为10％。 |
| `minHealthPercent` | `int32` | 只要关联的负载平衡池在健康模式下具有至少最小_健康_百分比主机，就会启用异常值检测。当负载平衡池中健康主机的百分比降至此阈值以下时，将禁用异常值检测，并且代理将在池中的所有主机（健康和不健康）之间进行负载平衡。默认值为50％。 |

### 实现过程

#### （1）边车注入

有两种方式进行边车注入：

1. 自动注入
2. 手动注入

自动注入是基于命名空间的标签的，只要给命名空间加上 "istio-injection=enabled" 的标签，随后加入该命名空间的 Pod 都会自动进行边车注入。添加启动边车标签的命令如下：

```text
kubectl label namespace <命名空间> istio-injection=enabled
```

手动注入需要使用 istioctl，具体命令如下：

```text
kubectl apply -f <(istioctl kube-inject -f samples/httpbin/httpbin.yaml)
```

#### （2）限流

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews-cb-policy
spec:
  host: reviews.prod.svc.cluster.local
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http2MaxRequests: 1000
        maxRequestsPerConnection: 10
```

以上规则设置连接池大小为100个连接和1000个并发HTTP2请求，与 “reviews” 服务的请求/连接不超过10个。

#### （3）断路

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews-cb-policy
spec:
  host: reviews.prod.svc.cluster.local
  trafficPolicy:
    outlierDetection:
      consecutiveErrors: 7
      interval: 5m
      baseEjectionTime: 15m
```

以上规则设置每5分钟扫描一次的上游主机，这样任何连续7次出现5XX错误代码的主机都将被驱逐15分钟。

## 日志收集

### 方案概述

**采用 EFK 方案，在集群的每个节点部署一个 Fluent Bit，负责收集日志转发到 Fluentd，Fluentd 处理完数据后路由到多个不同的输出节点（比如 ElasticSearch、S3）**。数据流图如下：

![](../.gitbook/assets/image%20%2830%29.png)

图片来源：[Fluentd vs. Fluent Bit: Side by Side Comparison - logz.io](https://logz.io/blog/fluentd-vs-fluent-bit/)

### 相关原理

#### Fluentd、Fluent Bit 和 Logstash 对比

|  | [Fluentd](https://github.com/fluent/fluentd) | [Fluent Bit](https://github.com/fluent/fluent-bit/) | [Logstash](https://github.com/elastic/logstash) |
| :--- | :--- | :--- | :--- |
| 使用范围 | 容器/服务器 | 容器/服务器 | 容器/服务器 |
| 编写语言 | CRuby | C | JRuby |
| 内存占用 | ~40MB | ~450KB | ~120MBper |
| 性能 | 高性能 | 高性能 | 高性能 |
| 依赖 | 依赖 Ruby Gem | 零依赖，除非插件需要 | 依赖 JVM |
| 插件生态 | &gt; 350 个可用插件 | ~35 个可用插件 | 257 个可用插件 |
| 开源证书 | Apache 证书 2.0 | Apache 证书 2.0 | Apache 证书 2.0 |

该表格信息整理自以下文章：

1. [Fluentd vs. Fluent Bit: Side by Side Comparison - logz.io](https://logz.io/blog/fluentd-vs-fluent-bit/)
2. [Fluentd vs. Logstash: A Comparison of Log Collectors - logz.io](https://logz.io/blog/fluentd-logstash/)
3. [Logstash Plugins - github.com](https://github.com/logstash-plugins)

### 实现过程

#### （1）新建 logging 命名空间

```text
kubectl create namespace logging
```

#### （2）部署 Fluent Bit 配置

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: logging
  labels:
    k8s-app: fluent-bit
data:
  # Configuration files: server, input, filters and output
  # ======================================================
  fluent-bit.conf: |
    [SERVICE]
        Flush         1
        Log_Level     info
        Daemon        off
        Parsers_File  parsers.conf
        HTTP_Server   On
        HTTP_Listen   0.0.0.0
        HTTP_Port     2020

    @INCLUDE input-kubernetes.conf
    @INCLUDE filter-kubernetes.conf
    @INCLUDE output-elasticsearch.conf

  input-kubernetes.conf: |
    [INPUT]
        Name              tail
        Tag               kube.*
        Path              /var/log/containers/*.log
        Parser            docker
        DB                /var/log/flb_kube.db
        Mem_Buf_Limit     5MB
        Skip_Long_Lines   On
        Refresh_Interval  10

  filter-kubernetes.conf: |
    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Kube_Tag_Prefix     kube.var.log.containers.
        Merge_Log           On
        Merge_Log_Key       log_processed
        K8S-Logging.Parser  On
        K8S-Logging.Exclude Off

  output-fluentd.conf: |
    [OUTPUT]
        Name          forward
        Match         *
        Host          ${FLUENTD_HOST}
        Port          ${FLUENTD_PORT}
  parsers.conf: |
    [PARSER]
        Name   apache
        Format regex
        Regex  ^(?<host>[^ ]*) [^ ]* (?<user>[^ ]*) \[(?<time>[^\]]*)\] "(?<method>\S+)(?: +(?<path>[^\"]*?)(?: +\S*)?)?" (?<code>[^ ]*) (?<size>[^ ]*)(?: "(?<referer>[^\"]*)" "(?<agent>[^\"]*)")?$
        Time_Key time
        Time_Format %d/%b/%Y:%H:%M:%S %z

    [PARSER]
        Name   apache2
        Format regex
        Regex  ^(?<host>[^ ]*) [^ ]* (?<user>[^ ]*) \[(?<time>[^\]]*)\] "(?<method>\S+)(?: +(?<path>[^ ]*) +\S*)?" (?<code>[^ ]*) (?<size>[^ ]*)(?: "(?<referer>[^\"]*)" "(?<agent>[^\"]*)")?$
        Time_Key time
        Time_Format %d/%b/%Y:%H:%M:%S %z

    [PARSER]
        Name   apache_error
        Format regex
        Regex  ^\[[^ ]* (?<time>[^\]]*)\] \[(?<level>[^\]]*)\](?: \[pid (?<pid>[^\]]*)\])?( \[client (?<client>[^\]]*)\])? (?<message>.*)$

    [PARSER]
        Name   nginx
        Format regex
        Regex ^(?<remote>[^ ]*) (?<host>[^ ]*) (?<user>[^ ]*) \[(?<time>[^\]]*)\] "(?<method>\S+)(?: +(?<path>[^\"]*?)(?: +\S*)?)?" (?<code>[^ ]*) (?<size>[^ ]*)(?: "(?<referer>[^\"]*)" "(?<agent>[^\"]*)")?$
        Time_Key time
        Time_Format %d/%b/%Y:%H:%M:%S %z

    [PARSER]
        Name   json
        Format json
        Time_Key time
        Time_Format %d/%b/%Y:%H:%M:%S %z

    [PARSER]
        Name        docker
        Format      json
        Time_Key    time
        Time_Format %Y-%m-%dT%H:%M:%S.%L
        Time_Keep   On

    [PARSER]
        Name        syslog
        Format      regex
        Regex       ^\<(?<pri>[0-9]+)\>(?<time>[^ ]* {1,2}[^ ]* [^ ]*) (?<host>[^ ]*) (?<ident>[a-zA-Z0-9_\/\.\-]*)(?:\[(?<pid>[0-9]+)\])?(?:[^\:]*\:)? *(?<message>.*)$
        Time_Key    time
        Time_Format %b %d %H:%M:%S
```

#### （3）部署 Fluent Bit

```yaml
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: logging
  labels:
    k8s-app: fluent-bit-logging
    version: v1
    kubernetes.io/cluster-service: "true"
spec:
  template:
    metadata:
      labels:
        k8s-app: fluent-bit-logging
        version: v1
        kubernetes.io/cluster-service: "true"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "2020"
        prometheus.io/path: /api/v1/metrics/prometheus
    spec:
      containers:
      - name: fluent-bit
        image: fluent/fluent-bit:1.2.1
        imagePullPolicy: Always
        ports:
          - containerPort: 2020
        env:
        - name: FLUENTD_HOST
          value: "fluentd"
        - name: FLUENTD_PORT
          value: "24224"
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: fluent-bit-config
          mountPath: /fluent-bit/etc/
      terminationGracePeriodSeconds: 10
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: fluent-bit-config
        configMap:
          name: fluent-bit-config
      serviceAccountName: fluent-bit
      tolerations:
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      - operator: "Exists"
        effect: "NoExecute"
      - operator: "Exists"
        effect: "NoSchedule"
```

#### （3）部署 Fluntd 配置

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: fluentd-es-config
  namespace: logging
data:
  containers.input.conf: |-
    <source>
      id fluentd-containers.log
      type tail
      path /var/log/containers/*.log
      pos_file /var/log/es-containers.log.pos
      time_format %Y-%m-%dT%H:%M:%S.%NZ
      localtime
      tag raw.kubernetes.*
      format json
      read_from_head true
    </source>
  forward.input.conf: |-
    # 采用 TCP 发送的消息
    <source>
      type forward
    </source>
  output.conf: |-
    <match **>
       type elasticsearch
       log_level info
       include_tag_key true
       host elasticsearch
       port 9200
       logstash_format true
       # 设置 chunk limits.
       buffer_chunk_limit 2M
       buffer_queue_limit 8
       flush_interval 5s
       # 重试间隔绝对不要超过 5 分钟。
       max_retry_wait 30
       # 禁用重试次数限制（永远重试）。
       disable_retry_limit
       # 使用多线程。
       num_threads 2
    </match>
```

#### （4）部署 Fluentd

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fluentd-es
  namespace: logging
  labels:
    app: fluentd-es
    sidecar.istio.io/inject: "false"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: fluentd-es
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: fluentd-es
    spec:
      containers:
      - name: fluentd-es
        image: gcr.io/google-containers/fluentd-elasticsearch:v2.0.1
        env:
        - name: FLUENTD_ARGS
          value: --no-supervisor -q
        resources:
          limits:
            memory: 500Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: config-volume
          mountPath: /etc/fluent/config.d
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: config-volume
        configMap:
          name: fluentd-es-config
```

#### （5）部署 ElasticSearch

```yaml
# Elasticsearch Service
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch
  namespace: logging
  labels:
    app: elasticsearch
spec:
  ports:
  - port: 9200
    protocol: TCP
    targetPort: db
  selector:
    app: elasticsearch
---
# Elasticsearch Deployment
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: elasticsearch
  namespace: logging
  labels:
    app: elasticsearch
spec:
  template:
    metadata:
      labels:
        app: elasticsearch
      annotations:
        sidecar.istio.io/inject: "false"
    spec:
      containers:
      - image: docker.elastic.co/elasticsearch/elasticsearch-oss:6.1.1
        name: elasticsearch
        resources:
          # need more cpu upon initialization, therefore burstable class
          limits:
            cpu: 1000m
          requests:
            cpu: 100m
        env:
          - name: discovery.type
            value: single-node
        ports:
        - containerPort: 9200
          name: db
          protocol: TCP
        - containerPort: 9300
          name: transport
          protocol: TCP
        volumeMounts:
        - name: elasticsearch
          mountPath: /data
      volumes:
      - name: elasticsearch
        emptyDir: {}
```

#### （6）部署 Kibana

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kibana
  namespace: logging
  labels:
    app: kibana
spec:
  ports:
  - port: 5601
  type: NodePort
  selector:
    app: kibana

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: logging
  labels:
    app: kibana
spec:
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
    spec:
      containers:
      - name: kibana
        image: docker.elastic.co/kibana/kibana-oss:6.4.3
        resources:
          limits:
            cpu: 1000m
          requests:
            cpu: 100m
        env:
          - name: ELASTICSEARCH_URL
            value: http://elasticsearch:9200
        ports:
        - containerPort: 5601
```

#### （7）查看日志

使用端口转发方式可以查看 Kibana Dashboard，具体操作如下：

（1）在操作集群的 EC2 上执行以下命令，将 EC2 上端口 5601 的流量转发到集群端口为 5601 的服务，即是 kibana 服务。

```text
kubectl -n logging port-forward $(kubectl -n logging get pod -l app=kibana -o jsonpath='{.items[0].metadata.name}') 5601:5601 &
```

（2）访问 [http://JUMPER\_URL:5601/](http://jumper_url:5601/)。

#### （8）测试

使用 counter 测试本地日志采集，操作如下：

（1）运行 pod/counter，这个 pod 会每隔 1 秒打印一条日志。

```text
kubectl apply -f https://k8s.io/examples/debug/counter-pod.yaml
kubectl logs counter count
```

（2）通过 Kibana 查看日志。

![](../.gitbook/assets/image%20%2829%29.png)

## 服务调用追踪

### 方案概述

**采用 Istio 插件 jaeger tracing 进行服务调用追踪和展示。**

### 相关原理

#### Envoy 支持 Tracing

Envoy支持与系统范围跟踪相关的三个功能：

* 请求ID生成：Envoy将在需要时生成UUID并填充 [x-request-id](https://www.envoyproxy.io/docs/envoy/v1.11.1/configuration/http_conn_man/headers#config-http-conn-man-headers-x-request-id) HTTP标头。应用程序可以转发x-request-id标头以进行统一日志记录和跟踪。
* 客户端跟踪ID加入：[x-client-trace-id](https://www.envoyproxy.io/docs/envoy/v1.11.1/configuration/http_conn_man/headers#config-http-conn-man-headers-x-client-trace-id)标头可用于将不受信任的请求ID加入可信内部 [x-request-id](https://www.envoyproxy.io/docs/envoy/v1.11.1/configuration/http_conn_man/headers#config-http-conn-man-headers-x-request-id)。
* 外部跟踪服务集成

  ：Envoy支持可插拔外部跟踪可视化提供程序，它们分为两个子组：

  * 外部跟踪器是Envoy代码库的一部分，如[LightStep](https://lightstep.com/)， [Zipkin](https://zipkin.io/) 或任何兼容Zipkin的后端（例如[Jaeger](https://github.com/jaegertracing/)）和 [Datadog](https://datadoghq.com/)。
  * 作为第三方插件的外部跟踪器，如[Instana](https://www.instana.com/blog/monitoring-envoy-proxy-microservices/)。

#### B3-propagation 追踪上下文

B3-propagation 定义了一系列以 "x-B3" 为前缀的请求头，用来追踪请求的上下文，关于 B3 的更多知识参见 [https://github.com/openzipkin/b3-propagation](https://github.com/openzipkin/b3-propagation)

"B3" 的含义：Zipkin originated at Twitter. Many services were named after birds. Zipkin started there under the initial project name **Big Brother Bird**. – [Issue\#36 What does "B3" stand for/mean? ](https://github.com/openzipkin/b3-propagation/issues/36)

#### Istio Tracing

Istio Tracing 基于 Envoy 和 B3-propagation。所以在调用服务时需要确保以下请求头被透传下去。

* `x-request-id`
* `x-b3-traceid`
* `x-b3-spanid`
* `x-b3-parentspanid`
* `x-b3-sampled`
* `x-b3-flags`
* `x-ot-span-context`

### 实现过程

#### （1）启用 Jaeger Tracing

按照[安装指南](https://istio.io/zh/docs/setup/)中的说明安装 Istio。使用 Helm chart 或 Helm template 进行安装时，设置 `--set tracing.enabled=true` 选项以启用追踪。

#### （2）设置追踪采样比

**2.1）编辑 istio-pilot deployment**

```text
kubectl -n istio-system edit deploy istio-pilot
```

**2.2）找到 PILOT\_TRACE\_SAMPLING 环境变量，并修改 value: 为期望的百分比。取值范围为 0.0 到 100.0，精度为 0.01。**

#### （3）查看仪表盘

通过请求转发，可以访问 Jaeger UI，可视化地查看服务调用链。

**3.1）在操作集群的 EC2 上执行以下命令，将 EC2 上端口 15302 的流量转发到集群端口为 16686 的服务，即是 jaeger-query 服务。**

```text
kubectl -n istio-system port-forward --address 0.0.0.0  $(kubectl -n istio-system get pod -l app=jaeger -o jsonpath='{.items[0].metadata.name}') 15032:16686 &
```

**3.2）访问 http://JUMPER\_URL:15032。**

**3.4）查找追踪信息**

![](../.gitbook/assets/image%20%2822%29.png)

**3.5）查看追踪的详情**

![](../.gitbook/assets/image%20%2814%29.png)

## 服务监控

### 方案概述

**采用** [**Prometheus**](https://github.com/prometheus) **进行服务监控，随后通过企业微信群机器人进行告警通知。**其架构如下图所示：

![](../.gitbook/assets/image.png)

效果图如下：

![](../.gitbook/assets/image%20%2824%29.png)

### 相关原理

#### Prometheus

Prometheus 是一个开源监控系统，在 2016 年加入 CNCF，其特性如下：

1. 多维度数据模型
2. 灵活的查询语言
3. 不依赖分布式存储，单个服务器节点是自主的
4. 以HTTP方式，通过pull模型拉去时间序列数据
5. 也通过中间网关支持push模型
6. 通过服务发现或者静态配置，来发现目标服务对象
7. 支持多种多样的图表和界面展示，grafana也支持它

### 实现过程

#### （1）部署 Node-Exporter

```yaml
cat <<EOF > node-exporter.yaml
---
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: kube-system
  labels:
    k8s-app: node-exporter
spec:
  template:
    metadata:
      labels:
        k8s-app: node-exporter
    spec:
      containers:
      - image: prom/node-exporter
        name: node-exporter
        ports:
        - containerPort: 9100
          protocol: TCP
          name: http
---
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: node-exporter
  name: node-exporter
  namespace: kube-system
spec:
  ports:
  - name: http
    port: 9100
    nodePort: 31672
    protocol: TCP
  type: NodePort
  selector:
    k8s-app: node-exporter
EOF

kubectl create -f node-exporter.yaml
```

#### （2）创建promethues的集群角色，并为用户绑定角色

```yaml
cat <<EOF > rbac-setup.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
  - nodes
  - nodes/proxy
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups:
  - extensions
  resources:
  - ingresses
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: kube-system
EOF

kubectl create -f rbac-setup.yaml
```

#### （3）定义了要监控的集群资源

```yaml
cat <<EOF > configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: kube-system
data:
  prometheus.yml: |
    global:
      scrape_interval:     15s
      evaluation_interval: 15s
    scrape_configs:

    - job_name: 'kubernetes-apiservers'
      kubernetes_sd_configs:
      - role: endpoints
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: default;kubernetes;https

    - job_name: 'kubernetes-nodes'
      kubernetes_sd_configs:
      - role: node
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics

    - job_name: 'kubernetes-cadvisor'
      kubernetes_sd_configs:
      - role: node
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor

    - job_name: 'kubernetes-service-endpoints'
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
        action: replace
        target_label: __scheme__
        regex: (https?)
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_service_name]
        action: replace
        target_label: kubernetes_name

    - job_name: 'kubernetes-services'
      kubernetes_sd_configs:
      - role: service
      metrics_path: /probe
      params:
        module: [http_2xx]
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_probe]
        action: keep
        regex: true
      - source_labels: [__address__]
        target_label: __param_target
      - target_label: __address__
        replacement: blackbox-exporter.example.com:9115
      - source_labels: [__param_target]
        target_label: instance
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_service_name]
        target_label: kubernetes_name

    - job_name: 'kubernetes-ingresses'
      kubernetes_sd_configs:
      - role: ingress
      relabel_configs:
      - source_labels: [__meta_kubernetes_ingress_annotation_prometheus_io_probe]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_ingress_scheme,__address__,__meta_kubernetes_ingress_path]
        regex: (.+);(.+);(.+)
        replacement: ${1}://${2}${3}
        target_label: __param_target
      - target_label: __address__
        replacement: blackbox-exporter.example.com:9115
      - source_labels: [__param_target]
        target_label: instance
      - action: labelmap
        regex: __meta_kubernetes_ingress_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_ingress_name]
        target_label: kubernetes_name

    - job_name: 'kubernetes-pods'
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: kubernetes_pod_name
EOF

kubectl create -f configmap.yaml
```

#### （4）部署 Promethues

```yaml
cat <<EOF > prometheus.deploy.yaml
---
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  labels:
    name: prometheus-deployment
  name: prometheus
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
      - image: prom/prometheus:v2.0.0
        name: prometheus
        command:
        - "/bin/prometheus"
        args:
        - "--config.file=/etc/prometheus/prometheus.yml"
        - "--storage.tsdb.path=/prometheus"
        - "--storage.tsdb.retention=24h"
        ports:
        - containerPort: 9090
          protocol: TCP
        volumeMounts:
        - mountPath: "/prometheus"
          name: data
        - mountPath: "/etc/prometheus"
          name: config-volume
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
          limits:
            cpu: 500m
            memory: 2500Mi
      serviceAccountName: prometheus  
      volumes:
      - name: data
        emptyDir: {}
      - name: config-volume
        configMap:
          name: prometheus-config 
EOF

kubectl create -f prometheus.deploy.yaml
```

#### （5）暴露 Prometheus 服务供 Grafana 等图形化界面使用

```yaml
cat <<EOF > prometheus.svc.yaml
---
kind: Service
apiVersion: v1
metadata:
  labels:
    app: prometheus
  name: prometheus
  namespace: kube-system
spec:
  type: NodePort
  ports:
  - port: 9090
    targetPort: 9090
    nodePort: 30003
  selector:
    app: prometheus
EOF

kubectl create -f prometheus.svc.yaml
```

Grafana 可以通过 StatefulSet 的方式使其保留卷状态，达到安装的插件在 Pod 重启后依旧存在且可用的目的，具体配置如下：

```yaml
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  labels:
    app: grafana
    chart: grafana
    heritage: Tiller
    release: istio
  name: grafana
  namespace: istio-system
spec:
  volumeClaimTemplates:
  - metadata:
      name: storage
      annotations:
        volume.alpha.kubernetes.io/storage-class: anything
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
  serviceName: "grafana"
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      annotations:
        prometheus.io/scrape: "true"
        sidecar.istio.io/inject: "false"
      labels:
        app: grafana
        chart: grafana
        heritage: Tiller
        release: istio
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - preference:
              matchExpressions:
              - key: beta.kubernetes.io/arch
                operator: In
                values:
                - amd64
            weight: 2
          - preference:
              matchExpressions:
              - key: beta.kubernetes.io/arch
                operator: In
                values:
                - ppc64le
            weight: 2
          - preference:
              matchExpressions:
              - key: beta.kubernetes.io/arch
                operator: In
                values:
                - s390x
            weight: 2
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: beta.kubernetes.io/arch
                operator: In
                values:
                - amd64
                - ppc64le
                - s390x
      containers:
      - env:
        - name: GRAFANA_PORT
          value: "3000"
        - name: GF_AUTH_BASIC_ENABLED
          value: "false"
        - name: GF_AUTH_ANONYMOUS_ENABLED
          value: "true"
        - name: GF_AUTH_ANONYMOUS_ORG_ROLE
          value: Admin
        - name: GF_PATHS_DATA
          value: /data/grafana
        image: grafana/grafana:6.1.6
        imagePullPolicy: IfNotPresent
        name: grafana
        ports:
        - containerPort: 3000
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /login
            port: 3000
            scheme: HTTP
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
        resources:
          requests:
            cpu: 10m
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /var/lib/grafana
          subPath: grafana
          name: storage
        - mountPath: /data/grafana
          name: data
        - mountPath: /var/lib/grafana/dashboards/istio/galley-dashboard.json
          name: dashboards-istio-galley-dashboard
          readOnly: true
          subPath: galley-dashboard.json
        - mountPath: /var/lib/grafana/dashboards/istio/istio-mesh-dashboard.json
          name: dashboards-istio-istio-mesh-dashboard
          readOnly: true
          subPath: istio-mesh-dashboard.json
        - mountPath: /var/lib/grafana/dashboards/istio/istio-performance-dashboard.json
          name: dashboards-istio-istio-performance-dashboard
          readOnly: true
          subPath: istio-performance-dashboard.json
        - mountPath: /var/lib/grafana/dashboards/istio/istio-service-dashboard.json
          name: dashboards-istio-istio-service-dashboard
          readOnly: true
          subPath: istio-service-dashboard.json
        - mountPath: /var/lib/grafana/dashboards/istio/istio-workload-dashboard.json
          name: dashboards-istio-istio-workload-dashboard
          readOnly: true
          subPath: istio-workload-dashboard.json
        - mountPath: /var/lib/grafana/dashboards/istio/mixer-dashboard.json
          name: dashboards-istio-mixer-dashboard
          readOnly: true
          subPath: mixer-dashboard.json
        - mountPath: /var/lib/grafana/dashboards/istio/pilot-dashboard.json
          name: dashboards-istio-pilot-dashboard
          readOnly: true
          subPath: pilot-dashboard.json
        - mountPath: /etc/grafana/provisioning/datasources/datasources.yaml
          name: config
          subPath: datasources.yaml
        - mountPath: /etc/grafana/provisioning/dashboards/dashboardproviders.yaml
          name: config
          subPath: dashboardproviders.yaml
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext:
        fsGroup: 472
        runAsUser: 472
      terminationGracePeriodSeconds: 30
      volumes:
      - configMap:
          defaultMode: 420
          name: istio-grafana
        name: config
      - emptyDir: {}
        name: data
      - configMap:
          defaultMode: 420
          name: istio-grafana-configuration-dashboards-galley-dashboard
        name: dashboards-istio-galley-dashboard
      - configMap:
          defaultMode: 420
          name: istio-grafana-configuration-dashboards-istio-mesh-dashboard
        name: dashboards-istio-istio-mesh-dashboard
      - configMap:
          defaultMode: 420
          name: istio-grafana-configuration-dashboards-istio-performance-dashboard
        name: dashboards-istio-istio-performance-dashboard
      - configMap:
          defaultMode: 420
          name: istio-grafana-configuration-dashboards-istio-service-dashboard
        name: dashboards-istio-istio-service-dashboard
      - configMap:
          defaultMode: 420
          name: istio-grafana-configuration-dashboards-istio-workload-dashboard
        name: dashboards-istio-istio-workload-dashboard
      - configMap:
          defaultMode: 420
          name: istio-grafana-configuration-dashboards-mixer-dashboard
        name: dashboards-istio-mixer-dashboard
      - configMap:
          defaultMode: 420
          name: istio-grafana-configuration-dashboards-pilot-dashboard
        name: dashboards-istio-pilot-dashboard
```

