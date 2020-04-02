# Istio

## 概述

Istio 是服务网格的经典实现，用来管理集群中的流量。见 [https://istio.io/zh](https://istio.io/zh)。

## 安装

（1）下载 Istio

```bash
curl -L https://git.io/getLatestIstio | sh -
cd istio-1.*
```

（2）配置 Helm

Helm 是 Kubernetes 的包管理器。安装方法见：[https://helm.sh/docs/using\_helm/\#installing-helm](https://helm.sh/docs/using_helm/#installing-helm)

安装完成之后需要按如下方法配置下 Helm 的服务端组件 Tiller：

```bash
kubectl create -f install/kubernetes/helm/helm-service-account.yaml
helm init --service-account tiller
```

（3）执行安装

```bash
# 安装 Istio CRD（自定义的资源定义）
helm install \
--wait \
--name istio-init \
--namespace istio-system \
install/kubernetes/helm/istio-init


# 安装 Istio 核心组件
# --values 指定了启动的 demo 服务（可选）
helm install \
--wait \
--name istio \
--namespace istio-system \
install/kubernetes/helm/istio \
--values install/kubernetes/helm/istio/values-istio-demo.yaml
```

## 熔断配置

Istio 中的熔断配置有两种：

1. 连接池配置；
2. 主机隔离配置。

### 连接池配置

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

### 主机隔离配置

主机隔离配置可以配置对服务的上游主机进行定时检查，并通过配置的规则判定主机的健康情况、驱逐不健康的主机以及自动恢复驱逐的主机。

| 字段 | 类型 | 描述 |
| :--- | :--- | :--- |
| `consecutiveErrors` | `int32` | 单个主机能容忍的连续错误的个数，超出将会被驱逐。默认值是 5。对于 HTTP 连接，返回码为 502、503 或 504 视为错误；对于 TCP 连接，连接超时和连接失败事件视为错误。 |
| `interval` | `google.protobuf.Duration` | 两次扫描分析主机的时间间隔。格式为 1h/1m/1s/1ms，必须 &gt;=1ms。默认值时 10s。 |
| `baseEjectionTime` | `google.protobuf.Duration` | 最小驱逐时间。一个主机的驱逐期 = 最小驱逐时间 x 驱逐次数。该技术允许系统自动增加不健康上游主机的驱逐周期。格式：1h/1m/1s/1ms。必须 &gt;= 1ms。默认值为30秒。 |
| `maxEjectionPercent` | `int32` | 负载平衡池中可以驱逐的上游服务的最大主机百分比。默认为10％。 |
| `minHealthPercent` | `int32` | 只要关联的负载平衡池在健康模式下具有至少最小_健康_百分比主机，就会启用异常值检测。当负载平衡池中健康主机的百分比降至此阈值以下时，将禁用异常值检测，并且代理将在池中的所有主机（健康和不健康）之间进行负载平衡。默认值为50％。 |

### 示例

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
    outlierDetection:
      consecutiveErrors: 7
      interval: 5m
      baseEjectionTime: 15m
```

以上规则设置连接池大小为100个连接和1000个并发HTTP2请求，与 “reviews” 服务的请求/连接不超过10个。此外，它配置每5分钟扫描一次的上游主机，这样任何连续7次出现5XX错误代码的主机都将被驱逐15分钟。

## 负载均衡

> 负载均衡（Load Balancing）是指能将客户端的请求均衡地负载到服务的各个节点（计算、网络或存储资源），可分为客户端的负载均衡和服务端的负载均衡，还可分为按连接的负载均衡和按请求的负载均衡。
>
> 客户端的负载均衡依赖于对客户端的信任，一旦客户端暴走，服务端很可能出现节点过载。按连接的负载均衡一旦连接建立后，后续的请求都会交由特定的节点进行处理，请求暴增时，服务端也很可能出现节点过载。因此我们希望做的是**服务端的按请求的负载均衡**。

Istio 中使用 Gateway 进行负载均衡。要使用 Istio 的负载均衡功能，需要把我们的流量从 Gateway 中流入到设定的 VirtualService，从而导向我们的微服务。理清下 Istio Gateway 相关的几个 CRD 的含义：

1. VirtualService：设定路由规则；
2. DestinationRule：设定流量策略，包括熔断、限流和负载均衡策略；
3. ServiceEntry：注册网格外服务；
4. Gateway：网关，负载均衡器，指定要开放的端口、协议和 SNI 配置（SSL/TLS 证书绑定相关）。

在整个服务网格的视点，我们发现整个服务网格的入口由 Gateway 定义，出口由 ServiceEntry 定义，内部的网格路由由 VirtualService 定义，内部的网格流量由 DestinationRule 定义。

Istio 支持 3 种负载均衡算法，分别是：

1. 轮询（ROUND\_ROBIN）
2. 最小连接（LEAST\_CONN）
3. 随机（RANDOM）

## 实例：httpbin 服务的 HTTP 请求负载均衡

参考：[https://istio.io/zh/docs/tasks/traffic-management/ingress/](https://istio.io/zh/docs/tasks/traffic-management/ingress/)

### （1）部署 httpbin 应用

```text
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.2/samples/httpbin/httpbin.yaml
```

### （2）确定入口 IP 和端口

```text
kubectl get svc istio-ingressgateway -n istio-system
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].port}')
```

### （3）配置 Gateway

以下配置定义了一个名为 httpbin-gateway 的 Gateway 资源，采用 Istio 的 gateway 默认实现，暴露 host 为 "httpbin.example.com" 端口为 80 的 VirtualService。

```text
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

### （4）配置 VirtualService

以下配置定义了一个名为 httpbin 的 VirtualService，其绑定到名为 httpbin-gateway 的 Gateway，绑定名为 "httpbin.example.com" 的 host，将 host 的 /status 和 /delay 路径路由到集群中名为 httpbin，端口为 8000 的服务。

```text
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

### （5）使用 curl 访问 httpbin 服务

在配置了 DNS 解析的情况下我们通过 [http://httpbin.example.com/status/200](http://httpbin.example.com/status/200) 即可访问我们的服务，但是在没有配置 DNS 记录的情况下，可以通过 -H 配置请求头的方式访问。命令如下：

```text
curl -I -HHost:httpbin.example.com http://$INGRESS_HOST:$INGRESS_PORT/status/200
```

### （6）使用浏览器访问 httpbin 服务

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

### （7）验证负载均衡

扩充 httpbin 服务的底层 pod 数量，然后指定其中一个的 version 标签为 v2。

```text
kubectl scale deployment httpbin --replicas=2
kubectl get pods | grep httpbin
kubectl label pods my-pod version=v2 --overwrite
```

接下来，不断访问我们的 httpbin 服务，然后观察 Kiali Dashboard 就可以看到如下的图像：

![](../.gitbook/assets/image%20%2813%29.png)

## 实例：testrpc 服务的 gRPC 负载均衡

> gRPC 是 Google 推出的一款基于 HTTP/2 协议支持异构系统 RPC 通信的依赖库，其使用 [protocol buffers](https://developers.google.com/protocol-buffers/docs/overview) 作为 RPC 消息和方法的定义语言。
>
> protocol buffers 目前有两个版本，v2 和 v3，在没有历史包袱的情况下，我们选用 v3 。
>
> 关于 proto3（即 protocol buffers v3） 的语法，见 [https://developers.google.com/protocol-buffers/docs/reference/proto3-spec。](https://developers.google.com/protocol-buffers/docs/reference/proto3-spec。)
>
> 关于如何在快速在 Go 中使用 gRPC，见 [https://grpc.io/docs/tutorials/basic/go/。](https://grpc.io/docs/tutorials/basic/go/。)

### （1）部署 testrpc & testrpcc 服务

```text
保存以下配置到文件 deploy_test_rpc.yaml，然后执行 kubectl apply -f deploy_test_rpc.yaml 部署服务。
```

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

### （2）配置 Gateway

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

### （2） 验证负载均衡

扩充 testrpc 服务的底层 pod 数量，然后指定其中一个的 version 标签为 v2。

```text
kubectl scale deployment testrpc --replicas=2
kubectl get pods | grep testrpc
kubectl label pods my-pod version=v2 --overwrite
```

接下来，不断访问我们的 testrpc 服务，然后观察 Kiali Dashboard 就可以看到如下的图像。流量有时经过 v2 的 Pod，有时经过 v1 的 Pod，而 testrpcc 只有一个 Pod 且只建立了一次连接，随后的请求均衡地负载到 testrpc 服务中的两个 Pod 中。

![](../.gitbook/assets/image%20%2825%29.png)

![](../.gitbook/assets/image%20%2824%29.png)

## 参考

1. [使用 Helm 进行安装 – istio.io/zh](https://istio.io/zh/docs/setup/kubernetes/install/helm/)
2. [Amazon EKS上的 Istio 入门指南 – AWS 官方博客](https://aws.amazon.com/cn/blogs/china/aws-getting-started-istio-eks/)
3. [Destination Rule - istio.io](https://istio.io/docs/reference/config/networking/v1alpha3/destination-rule/)
4. [熔断 - istio.io/zh](https://istio.io/zh/docs/tasks/traffic-management/circuit-breaking/)
5. [熔断与异常检测在 Istio 中的应用 - yangcs.net](https://www.yangcs.net/posts/circuit_breaking-and-outlier-detection-in-istio/)

