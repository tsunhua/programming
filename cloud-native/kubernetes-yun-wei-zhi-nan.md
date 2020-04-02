# Kubernetes 运维指南

## 节点相关操作

### 如何查看我的节点？

```bash
[ec2-user@ip-10-2-19-198 ~]$ kubectl get node
NAME                                        STATUS   ROLES    AGE   VERSION
ip-10-2-71-110.us-west-2.compute.internal   Ready    <none>   3d    v1.13.8-eks-cd3eb0
ip-10-2-95-229.us-west-2.compute.internal   Ready    <none>   3d    v1.13.8-eks-cd3eb0
```

### 如何扩缩我的节点？

在跳板机上使用 eksctl scale nodegroup 即可进行指定节点组内节点数量的扩缩，下面命令指定了名为 ecp-cafe 集群中 workers 节点组共有节点 2 个：

```text
eksctl scale nodegroup --cluster=ecp-cafe --nodes=2 --name=workers
```

### 如何驱逐问题节点？

当发现某节点出问题时可以将其中的 Pod 转移，然后其驱逐，如下：

```bash
# 先扩充多一个节点
eksctl scale nodegroup --cluster=ecp-cafe --nodes=3 --name=workers

# 查看问题节点上运行的 Pod
kubectl get pods --all-namespaces -o wide --field-selector spec.nodeName=ip-10-9-105-231.us-west-2.compute.internal

# 转移问题节点上的 Pod
kubectl drain ip-10-9-105-231.us-west-2.compute.internal --delete-local-data --force --ignore-daemonsets


# 如果误驱逐，可以恢复
kubectl uncordon ip-10-9-105-231.us-west-2.compute.internal

# 删除问题节点
kubectl delete node ip-10-9-105-231.us-west-2.compute.internal
```

### 如何删除我的节点？

在跳板机上可以获取节点名并根据节点名删除问题节点，如下：

```bash
[ec2-user@ip-10-2-19-198 ~]$ kubectl delete node ip-10-2-95-229.us-west-2.compute.internal
```

### 如何更换节点组实例类型？

```bash
# 查看已有节点组
eksctl get nodegroups --cluster ecp-cafe
# 将节点组缩为 0
eksctl scale nodegroup --cluster=ecp-cafe --nodes=0 --name=workers
# 删除节点组
eksctl delete  nodegroup --cluster=ecp-cafe --name=workers
# 新建节点组
eksctl create nodegroup --config-file=cluster.yaml
```

## Kubernetes 升级维护

### 如何查看当前 Kubernetes 版本？

```bash
[ec2-user@ip-10-2-19-198 ~]$ kubectl  version
Client Version: version.Info{Major:"1", Minor:"16", GitVersion:"v1.16.0", GitCommit:"2bd9643cee5b3b3a5ecbd3af49d09018f0773c77", GitTreeState:"clean", BuildDate:"2019-09-18T14:36:53Z", GoVersion:"go1.12.9", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"13+", GitVersion:"v1.13.11-eks-5876d6", GitCommit:"5876d6b7429820450950ade17fe7b4bf5ccada7f", GitTreeState:"clean", BuildDate:"2019-09-24T20:54:25Z", GoVersion:"go1.11.13", Compiler:"gc", Platform:"linux/amd64"}
```

### 如何更新 Kubernetes 版本？

使用 EKS 服务时，参见：[https://docs.aws.amazon.com/zh\_cn/eks/latest/userguide/update-cluster.html](https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/update-cluster.html)

尽管 Amazon EKS 运行高度可用的控制层面，但您可能在更新期间遇到次要服务中断。例如，如果您尝试在 API 服务器终止并由运行新版 Kubernetes 的新 API 服务器替换前后连接到 API 服务器，则可能遇到 API 调用错误或连接问题。如果出现这种情况，请重试您的 API 操作直至成功。

## Istio 升级维护

### 如何升级 istio 版本？

升级 istio 版本时，需要先下载新版本的 istio，并设置 ISTIO\_HOME 环境变量，如下：

```bash
export ISTIO_VERSION=1.3.4
curl -L https://git.io/getLatestIstio | sh -
cd istio-$ISTIO_VERSION
export ISTIO_HOME=`pwd`
export PATH=$ISTIO_HOME/bin:$PATH
```

接着，拉取自定义的 istio 配置，cd 进入进行覆盖操作：

```bash
cp -r ./ $ISTIO_HOME/install/kubernetes/helm/istio
```

最后，开始进行升级操作：

```bash
# 升级 istio-init
helm upgrade --install istio-init $ISTIO_HOME/install/kubernetes/helm/istio-init --namespace istio-system

# 升级 istio
helm upgrade istio $ISTIO_HOME/install/kubernetes/helm/istio --namespace istio-system --reuse-values

# 滚动重启注入了 istio-proxy 的 deployment 等
kubectl rollout restart deployment --namespace ecp-cafe
```

## 服务发布与配置管理

服务的发布与配置管理都在 Jenkins 中进行，通过选定不同的指令进行不同的发布操作，目前支持的操作如下：

1. 发布稳定部署（pub\_stable\_deploy）
2. 发布金丝雀部署（pub\_canary\_deploy）
3. 发布稳定部署而不重新打镜像（pub\_stable\_deploy\_reuse\_image）
4. 发布金丝雀部署而不重新打镜像（pub\_canary\_deploy\_reuse\_image）
5. 删除金丝雀部署（delete\_canary\_deploy）
6. 回滚到上一个稳定部署（undo\_stable\_deploy）
7. 发布稳定配置（pub\_stable\_config）
8. 发布稳定配置并重启服务（pub\_stable\_config\_and\_restart）
9. 发布金丝雀配置（pub\_canary\_config）
10. 取消金丝雀配置（undo\_canary\_config）

### 如何全量发布服务？

当需要重新打镜像时，按如下流程进行：

pub\_stable\_config =&gt; pub\_stable\_deploy

当不需要重新打打镜像时（通常是改动了部署配置而代码没有变更），按如下流程进行：

pub\_stable\_config =&gt; pub\_stable\_deploy\_reuse\_image

### 如何灰度发布服务？

当需要重新打镜像时，按如下流程进行：

pub\_stable\_config =&gt;pub\_canary\_deploy

当不需要重新打打镜像时（通常是改动了部署配置而代码没有变更），按如下流程进行：

pub\_stable\_config =&gt; pub\_canary\_deploy\_reuse\_imag

### 如何更新服务配置？

一般情况下，我们希望配置更新后，服务也能滚动更新，这时只需要执行以下流程即可：

pub\_stable\_config\_and\_restart

当我们需要对配置进行灰度时，可以使用执行以下流程对服务的其中一个 Pod 应用新配置：

1. 先在该服务的配置仓库根目录创建 cm-canary 文件夹，然后放置需要灰度的配置文件
2. pub\_canary\_config

当灰度结束时，请执行以下流程，取消配置灰度：

undo\_canary\_config

## 异常监控、告警及处理

### 如何监控我的服务？

通过 Grafana 面板可以便捷地查看当前集群中所有的节点、服务及 Pod 等资源的运行状况。

### 如何查看设定的告警？

可以在 Prometheus 面板上看到设定的告警规则，并查看当前值。发生告警时，根据生产配置，会告警到测试同学提供的地址，然后测试同学会通知到相关的企业微信群组。

### 如何处理告警？

目前，我们针对集群设定以下的告警项：

1. 节点 CPU 高使用率告警；
2. 节点内存高使用率告警；
3. 节点磁盘高使用率告警；
4. 服务 5xx 响应告警；
5. 服务 4xx 响应告警。

不同的告警有不同的处理流程：

**（1）节点 CPU 高使用率告警处理流程**

* 通过 Grafana 面板定位到高 CPU 占用的 Pod
* 通过 Grafana 面板获知其请求量，并据此判定该 Pod 是否发生异常
* 通过 Kibanna 面板查看该 Pod 的日志进一步排查
* 当确定 Pod 异常时，可通过 kubectl delete pod 方式暂时移除
* 当确定请求压力过大时，可以适当扩充 Pod数量，甚至扩充节点数量

**（2）节点内存高使用率告警处理流程**

* 通过 Grafana 面板定位到高内存占用的 Pod
* 通过 Grafana 面板获知其请求量，并据此判定该 Pod 是否发生异常
* 通过 Kibanna 面板查看该 Pod 的日志进一步排查
* 当确定 Pod 异常时，可通过 kubectl delete pod 方式暂时移除
* 当确定请求压力过大时，可以适当扩充 Pod数量，甚至扩充节点数量

**（3）节点磁盘高使用率告警处理流程**

集群使用的 aws 的文件存储服务（EFS），磁盘容量是随使用率扩大而弹性扩大，不应有告警。如出现，请先将 EFS 挂载到跳板机，然后查看里面的存储内容、存储量进行排查。如怀疑是 aws 问题，请及时向 aws support 提 issue。

**（4）服务 5xx 响应告警处理流程**

* 通过 Prometheus 面板（切换到 Graph 一栏）查看最近一段时间的 5xx 错误及其 response\_flags

  ```text
  ceil(sum by(destination_service_name,response_flags) (increase(istio_requests_total{destination_service_namespace="ecp-cafe",response_code=~"^5.*",response_flags!="DC"}[1m])))
  ```

* 如 response\_flags 为 “\_”，则为代码内部返回的 5xx
* 如 response\_flags 为其他值，请参考 [https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access\_log](https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log) 进行排查

（5）服务 4xx 响应告警处理流程

* 通过 Prometheus 面板（切换到 Graph 一栏）查看最近一段时间的 4xx 错误及其 response\_flags

  ```text
  ceil(sum by(destination_service_name,response_flags) (increase(istio_requests_total{destination_service_namespace="ecp-cafe",response_code=~"^4.*",response_flags!="DC"}[1m])))
  ```

* 如 response\_flags 为 “\_”，则为代码内部返回的 4xx
* 如 response\_flags 为其他值，请参考 [https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access\_log](https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log) 进行排查

## 跳板机维护

### 保护好跳板机很重要

没有了跳板机我们无法对集群进行任何操作，因此我们必须保护好跳板机，请照下列清单逐项检查是否完成：

* 启用跳板机的终止保护
* 创建该跳板机镜像，以便恢复

### 跳板机挂了，我该怎么办？

* 使用 cafe-ctl 镜像创建一台新的 EC2，并配置服务权限跟原先的跳板机一模一样
* 执行 `aws eks --region us-west-2 update-kubeconfig --name ecp-cafe` 将新集群绑定到新跳板机

## 附

关于 kubectl 及 eksctl 的更多操作，请参见：

1. [kubectl 命令表](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
2. [eksctl 起步](https://eksctl.io/introduction/getting-started/)

