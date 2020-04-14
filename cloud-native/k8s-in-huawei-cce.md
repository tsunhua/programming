# 基于华为云 CCE 的集群部署方案

## 概述

本方案采用华为云 CCE（底层是 Kubernetes）进行应用部署。Kubernetes 主节点由 CCE 托管并保证高可用，Node 节点采用华为云弹性云服务器\(4 核 8 G\)两台，分布在不同区域从而避免单点故障。集群控制器选用一台跟集群同一 VPC 的弹性云服务器\(2 核 4 G\)，搭建 Jenkins、Docker 、Kubtctl 等环境用于通过 Jenkins 打包部署服务。另外还会使用到华为云的容器镜像服务存储服务镜像、云磁盘服务挂载磁盘到服务器等。如代码使用 Gitlab 私有仓库，还需要确保集群控制器有拉取代码的网络路径和权限。

## 价格估算

以华东-上海二区域为例，选择按需使用的计费模式，估算月费单如下：

| 服务 | 规格 | 数量 | 时价（美元） | 日费（美元） | 月费（美元） |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Kubernetes 主节点 | 集群管理规模 1-50 节点 | N/R | 0.45 | 10.8 | 324 |
| 弹性云服务器\(ECS\) -Node 节点 | 虚拟机节点通用计算增强型 c3.xlarge.24 核 8 G | 2 | 0.3 | 7.2 | 216 |
| 弹性云服务器\(ECS\) -集群控制器 | 通用计算型 s3.large.22 核 4 G | 1 | 0.0532 | 1.2768 | 38.304 |
| 容器镜像服务\(SWR\) | 10G 以下 | 1 | 0.0000 | 0.0000 | 0.0000 |
| 弹性负载均衡\(ELB\) | 假定tps为 4，每个响应 1kb，每月有 10G 流量，则月费：0.123  _10 + 0.0030_  24 \* 30 | N/R | 0.0047 | 0.113 | 3.39 |
| 分布式缓存服务\(Memcached\) | 单机版2G | 1 | 0.04 | 0.96 | 28.8 |
| 文档数据库服务\(DDS\) | 亚太-新加坡副本集1 核 4 G | 1 | 0.09 | 2.16 | 64.8 |
| 总计 | N/R | N/R | 0.9379 | 22.5098 | 675.294 |

## 创建 Kubernetes 集群

### 服务选型

![](../.gitbook/assets/image%20%289%29.png)

### 创建节点

![](../.gitbook/assets/image%20%2812%29.png)

### 安装插件

![](../.gitbook/assets/image%20%2831%29.png)

### 配置确认

![](../.gitbook/assets/image%20%283%29.png)

### 完成

提交即进入完成页面，会显示创建进度，预计等待 3 ~ 10 分钟。

## 创建集群控制器

期望通过一台集群控制器使用 kubectl 控制整个集群。

根据[官方帮助文档](https://support-intl.huaweicloud.com/zh-cn/usermanual-cce/cce_01_0107.html)：

> CCE支持VPC网络内访问集群和互联网方式访问集群。
>
> * VPC网络内访问：您需要在“弹性云服务器”控制台购买一台云服务器，并确保和当前集群在同个VPC内。
> * 互联网访问：您需要准备一台能连接公网的云服务器。

操作步骤如下：

### 创建集群控制器

使用与当前集群同样的 VPC 创建一个 2 核 4 G 的云服务器，作为集群控制器 。

### 在集群控制器中配置 kubectl

登录[CCE控制台](https://console-intl.huaweicloud.com/cce2.0/?utm_source=helpcenter)，在左侧导航栏中选择“资源管理 &gt; 集群管理”，单击待连接集群下的“命令行工具 &gt; kubectl”。

![](../.gitbook/assets/image%20%2817%29.png)

随后执行以下操作：

1. 下载 kubectl 配置文件并拷贝到集群控制器中
2. 在集群控制器中执行以下命令安装 kubectl 并配置

   ```bash
   curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
   chmod +x ./kubectl
   sudo mv ./kubectl /usr/local/bin/kubectl
   sudo mkdir -p $HOME/.kube
   mv -f kubeconfig.json $HOME/.kube/config
   kubectl config use-context internal
   ```

### 查看集群信息

```text
kubectl cluster-info
```

### 安装 Docker

参考[官方文档](https://docs.docker.com/v17.09/engine/installation/linux/docker-ce/centos/#install-docker-ce-1)，按以下步骤在 CentOS 中安装 Docker。

**Install Docker CE in CentOS**

```bash
# Uninstall old versions
sudo yum remove docker \
                  docker-common \
                  docker-selinux \
                  docker-engine
# Add Docker Repo
sudo yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2
sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
# Install Docker CE
sudo yum install docker-ce

# Start Docker
sudo systemctl start docker

sudo usermod -a -G docker root

# Verify by hello-world image
sudo docker run hello-world
```

### 安装 Helm

**Install helm in China**

```bash
# Download helm
wget https://get.helm.sh/helm-v2.16.0-linux-amd64.tar.gz
# Unpack it
tar -zxvf helm-v2.16.0-linux-amd64.tar.gz
# Move to bin dir
mv linux-amd64/helm /usr/local/bin/helm

# Config ServiceAccount
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: tiller
  namespace: kube-system
EOF

# Init helm（中国大陆）
## Select proper Image
docker search tiller
helm init --service-account tiller --tiller-image=sapcc/tiller:v2.16.1

# Init helm (海外)
helm init --service-account tiller
```

### 安装 metric

```bash
# 安装
helm install stable/metrics-server \
    --name metrics-server \
    --version 2.11.0 \
    --namespace metrics
# 验证
kubectl get apiservice v1beta1.metrics.k8s.io -o yaml
```

### 安装 Istio

```bash
# 下载 istio 并配置环境变量
export ISTIO_VERSION=1.5.1
https://istio.io/downloadIstio sh -
cd istio-$ISTIO_VERSION
export PATH=$PWD/bin:$PATH

# 安装 istio
istioctl manifest apply --set profile=default --set addonComponents.prometheus.enabled=false

# 修改 $ISTIO_HOME/install/kubernetes/helm/istio/charts/gateways/values.yaml，添加以下注解
kubernetes.io/elb.class: union
kubernetes.io/session-affinity-mode: 114.119.180.209
kubernetes.io/elb.id: 2a92f374-811f-4faf-b28e-4f51f16596ab

# 确认 istio 安装情况
kubectl get svc -n istio-system

# 查看事件
kubectl  get event -n istio-system --sort-by="{.lastTimestamp}"
```

## 负载均衡

参考[官方文档](https://support.huaweicloud.com/api-cce/cce_02_0087.html)，需要在 Service 中的 annotations 处添加以下注解：

| 注解 | 是否必填 | 可选值 |
| :--- | :--- | :--- |
| kubernetes.io/elb.class | 是 | union |
| kubernetes.io/session-affinity-mode | 可选，需开启会话保持时必填 | SOURCE\_IP |
| kubernetes.io/elb.id | 可选，使用已有 ELB 时必填 | -- |
| kubernetes.io/elb.subnet-id | 可选，自动创建 ELS 时必填，Kubernetes v1.11.7-r0以上版本的集群可不填 | -- |
| kubernetes.io/elb.autocreate | 可选，自动创建 ELS 时必填 | 以下结构的 JSON 字符串形式，形如："{\"type\":\"inner\"}" 1. name：String，自动创建的负载均衡的名称。取值范围：1-64个字符，中英文，数字，下划线，中划线。 2. type：String，负载均衡实例网络类型，公网或者私网。public：公网型负载均衡inner：私网型负载均衡 3. bandwidth\_name：String，带宽的名称，默认值为：cce-bandwidth-**\*\***。1-64 字符，中文、英文字符、数字、下划线、中划线或点组成。 4. bandwidth\_chargemode：String，带宽付费模式。bandwidth：按带宽计费。traffic：按流量计费 5. bandwidth\_size：Integer，带宽大小，请根据具体region带宽支持范围设置，详情请参考[申请弹性公网IP](https://support.huaweicloud.com/api-vpc/zh-cn_topic_0020090596.html#ZH-CN_TOPIC_0020090596__table11041789)中**表4 bandwidth字段说明中size字段**。 6. bandwidth\_sharetype：String，带宽共享方式。PER：共享带宽。WHOLE：共享带宽 7. eip\_type：String，弹性公网IP类型，请参考ELB支持的弹性公网IP类型，详情请参考[申请弹性公网IP](https://support.huaweicloud.com/api-vpc/zh-cn_topic_0020090596.html#ZH-CN_TOPIC_0020090596__table11041789)中**表3 publicip字段说明type字段**。 |

## 文件存储卷\(PVC\)

参见：[https://support-intl.huaweicloud.com/zh-cn/usermanual-cce/cce\_01\_0111.html](https://support-intl.huaweicloud.com/zh-cn/usermanual-cce/cce_01_0111.html)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  annotations:
    volume.beta.kubernetes.io/storage-class: nfs-rw
    volume.beta.kubernetes.io/storage-provisioner: flexvolume-huawei.com/fuxinfs
  name: pvc-sfs-example
  namespace: default
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  volumeName: pv-sfs-example
  volumeNamespace: default
```

## 部署示例：nginx

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: nginx
  labels:
    istio-injection: enabled
---
kind: Service
apiVersion: v1
metadata:
  name: nginx
  namespace: nginx
  labels:
    app: nginx
spec:
  ports:
    - name: http
      protocol: TCP
      port: 8080
      targetPort: 80
  selector:
    app: nginx
  type: NodePort
  sessionAffinity: None
  externalTrafficPolicy: Cluster
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: nginx
  namespace: nginx
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.17.2-alpine
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
          imagePullPolicy: Always
          resources:
            requests:
              cpu: 100m
              memory: 200Mi
            limits:
              cpu: 1000m
              memory: 500Mi
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
      dnsPolicy: ClusterFirst
      schedulerName: default-scheduler
      imagePullSecrets:
      - name: default-secret
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 25%
      maxSurge: 25%
---
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: nginx-gateway
  namespace: nginx
spec:
  selector:
    istio: ingressgateway # use Istio nginx gateway implementation
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "nginx-dev.mobisummer-inc.com"
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: nginx
  namespace: nginx
spec:
  host: nginx
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: nginx
  namespace: nginx
spec:
  hosts:
  - "nginx-dev.mobisummer-inc.com"
  gateways:
  - nginx-gateway
  http:
  - match:
    route:
    - destination:
        port:
          number: 8080
        host: nginx
```

## 问题

### 节点无法拉取外网镜像

排查是否设置了 EIP，如果节点没有 EIP，那么无法拉取外网镜像。

