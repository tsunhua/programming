# Amazon EKS

## 概述

Amazon Elastic Kubernetes Service \(Amazon EKS\) 是一项托管服务，可让您在 AWS 上轻松运行 Kubernetes，而无需支持或维护您自己的 Kubernetes 控制层面。

Amazon EKS 跨多个可用区运行 Kubernetes 控制层面实例以确保高可用性。Amazon EKS 可以自动检测和替换运行状况不佳的控制层面实例，并为它们提供自动版本升级和修补。

Amazon EKS 还与许多 AWS 服务集成以便为您的应用程序提供可扩展性和安全性，包括：

* 用于容器镜像的 Amazon ECR
* 用于负载分配的 Elastic Load Balancing
* 用于身份验证的 IAM
* 用于隔离的 Amazon VPC

## 版本

| K8S 版本 | K8S 发布时间 | EKS 平台版本 | EKS 发布日志 |
| :--- | :--- | :--- | :--- |
| 1.13.7 | 2019.6.7 | eks.1 | Initial release of Kubernetes 1.13 for Amazon EKS. For more information, see [Kubernetes 1.13](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html#kubernetes-1.13). |
| 1.12.6 | 2019.2.27 | eks.2 | New platform version to support custom DNS names in the Kubelet certificate and improve `etcd` performance. This fixes a bug that caused worker node Kubelet daemons to request a new certificate every few seconds. |
| 1.12.6 | 2019.2.27 | eks.1 | Initial release of Kubernetes 1.12 for Amazon EKS. |
| 1.11.8 | 2019.3.1 | eks.3 | New platform version to support custom DNS names in the Kubelet certificate and improve `etcd` performance. |
| 1.11.8 | 2019.3.1 | eks.2 | New platform version updating Amazon EKS Kubernetes 1.11 clusters to patch level 1.11.8 to address [CVE-2019-1002100](https://discuss.kubernetes.io/t/kubernetes-security-announcement-v1-11-8-1-12-6-1-13-4-released-to-address-medium-severity-cve-2019-1002100/5147). |

## 预备

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

```bash
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

参考：

1. [Amazon EKS 基于身份的策略示例](https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/security_iam_id-based-policy-examples.html)
2. [https://github.com/weaveworks/eksctl/issues/204\#issuecomment-450280786（这位小哥说他亲自试了](https://github.com/weaveworks/eksctl/issues/204#issuecomment-450280786（这位小哥说他亲自试了) 30 多次才补全的，而我试了将近 40 次）
3. [https://docs.aws.amazon.com/autoscaling/ec2/userguide/control-access-using-iam.html](https://docs.aws.amazon.com/autoscaling/ec2/userguide/control-access-using-iam.html) 

注意：要有适量网关、VPC 和 IP 数量空余，否则会报达到最大限制错误。

## 创建集群

使用以下命令开始创建集群，其原理是：通过 aws cli 调用 CloudFormation 的相关 API，启动一个创建 EKS Cluster 的 Stack 和一个创建 EKS nodes 的 Stack 去创建集群所需的各种资源（包括网关、IP、VPC、EC2 等等）。

```text
eksctl create cluster \
--name prod \
--version 1.13 \
--nodegroup-name standard-workers \
--node-type t3.medium \
--nodes 3 \
--nodes-min 1 \
--nodes-max 4 \
--node-ami auto
```

> 注意：如果选择 P2 或 P3 实例类型和 Amazon EKS 优化的 AMI（具有 GPU 支持），则必须使用以下命令在集群上将[适用于 Kubernetes 的 NVIDIA 设备插件](https://github.com/NVIDIA/k8s-device-plugin)用作守护程序集。
>
> ```text
> kubectl apply -f https:``//raw.githubusercontent.com/NVIDIA/k8s-device-plugin/1.0.0-beta/nvidia-device-plugin.yml
> ```

还可以通过配置文件创建集群，如有以下名为 cluster.yaml 的文件，

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

执行以下命令即可完成安装：

```text
eksctl create cluster -f cluster.yaml
```

## 配置 IAM 授权

安装 aws-iam-authenticator 即可完成 IAM 授权，参见：[https://docs.aws.amazon.com/zh\_cn/eks/latest/userguide/install-aws-iam-authenticator.html](https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/install-aws-iam-authenticator.html)

```text
curl -o aws-iam-authenticator https://amazon-eks.s3-us-west-2.amazonaws.com/1.13.7/2019-06-11/bin/linux/amd64/aws-iam-authenticator
chmod +x ./aws-iam-authenticator
mkdir -p $HOME/bin && mv ./aws-iam-authenticator $HOME/bin/aws-iam-authenticator && export PATH=$HOME/bin:$PATH
echo 'export PATH=$HOME/bin:$PATH' >> ~/.bashrc
// 获取 token？
aws-iam-authenticator token -i <cluster name>
// 查看调用者?
aws sts get-caller-identity
```

## 创建 kubeconfig

参见：[https://docs.aws.amazon.com/zh\_cn/eks/latest/userguide/create-kubeconfig.html](https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/create-kubeconfig.html)

使用以下命令自动生成 kubeconfig

```text
// 生成 kubeconfig
aws eks --region <your region> update-kubeconfig --name <cluster name>
// 查看 kubeconfig
cat ~/.kube/config
```

## 查看集群状态

```text
// 查看节点状态
kubectl get nodes
// 查看服务状态
kubectl get svc
// 查看事件
kubectl get events --all-namespaces
```

## 部署 Dashboard

参见：

1. [https://aws.amazon.com/cn/premiumsupport/knowledge-center/eks-cluster-kubernetes-dashboard/](https://aws.amazon.com/cn/premiumsupport/knowledge-center/eks-cluster-kubernetes-dashboard/)
2. [https://docs.aws.amazon.com/zh\_cn/eks/latest/userguide/dashboard-tutorial.html](https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/dashboard-tutorial.html)
3. [https://youtu.be/JcZJqSa65Yc](https://youtu.be/JcZJqSa65Yc)

```bash
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
// 检索 eks-admin 服务账户的身份验证令牌。从输出中复制 <authentication_token> 值。您可以使用此令牌连接到控制面板
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep eks-admin | awk '{print $1}')


// 将所有请求从您的 Amazon EC2 实例本地主机端口转发到 Kubernetes 控制面板端口
kubectl port-forward svc/kubernetes-dashboard -n kube-system --address 0.0.0.0 6443:443
https://127.0.0.1:6443 输入 Token 即可访问 Dashboard。
```

## 节点扩充

```text
eksctl scale nodegroup --cluster=ecp-cafe --nodes=3 --name=ecp-cafe-workers
```

## 删除集群

```text
eksctl delete cluster --region=<your region> --name=<cluster name>
```

## 更多操作

* [10.3 Kubernetes 实战：管理 Hello World 集群](http://wiki.tec-do.com/pages/viewpage.action?pageId=58568894)
* [https://kubernetes.io/docs/tutorials/](https://kubernetes.io/docs/tutorials/)

