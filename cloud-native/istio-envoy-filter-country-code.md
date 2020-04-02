# Istio 中使用 EnvoyFilter 注入国家二字码的实现

## 实现

1. 将 `istio-ingressgateway` 的部署类型从 `Deployment` 改为 `DaemonSet`
2. `istio-ingressgateway Service 中 .spec.externalTrafficPolicy 值改为 Local`
3. `istio-ingressgateway Service 中 增加 service.beta.kubernetes.io/aws-load-balancer-type: nlb 注解`
4. 应用以下的 EnvoyFilter \(以下配置将对带有 country-code-injection: enabled 标签的部署注入 CloudFront-Viewer-Country 请求头\)

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: country-code-injector
  namespace: ecp-cafe
spec:
  workloadSelector:
    labels:
      country-code-injection: enabled
  configPatches:
    # The first patch adds the lua filter to the listener/http connection manager
  - applyTo: HTTP_FILTER
    match:
      context: SIDECAR_INBOUND
      listener:
        # portNumber: 8080
        filterChain:
          filter:
            name: "envoy.http_connection_manager"
            subFilter:
              name: "envoy.router"
    patch:
      operation: INSERT_BEFORE
      value: # lua filter specification
       name: envoy.lua
       config:
         inlineCode: |
           function envoy_on_request(request_handle)
              -- get ip from headers
              local xff_header = request_handle:headers():get("X-Forwarded-For");
              local client_ip
              for ip in  string.gmatch (xff_header, "(%d+.%d+.%d+.%d+)") do
                  client_ip = ip
              end
              -- query country by client_ip
              local headers, body = request_handle:httpCall(
              "lua_cluster",
              {
                [":method"] = "GET",
                [":authority"] = "lua_cluster",
                [":path"] = "/v1?only=countryCode&ip="..client_ip
              },
              "",
              100)
              request_handle:headers():add("CloudFront-Viewer-Country",body)
           end
  - applyTo: CLUSTER
    match:
      context: SIDECAR_OUTBOUND
    patch:
      operation: ADD
      value: # cluster specification
        name: "lua_cluster"
        type: STRICT_DNS
        connect_timeout: 0.5s
        lb_policy: ROUND_ROBIN
        hosts:
        - socket_address:
            protocol: TCP
            address: "10.2.31.92"
            port_value: 8888
```

## 原理

### AWS ELB 中网络负载均衡器支持保留源地址

各种负载均衡器的支持功能见：[https://aws.amazon.com/cn/elasticloadbalancing/features/?nc=sn&loc=2](https://aws.amazon.com/cn/elasticloadbalancing/features/?nc=sn&loc=2)

### Istio 支持 istio-ingressgateway 设置网络负载均衡器

参见：[https://kubernetes.io/docs/concepts/services-networking/service/\#aws-nlb-support](https://kubernetes.io/docs/concepts/services-networking/service/#aws-nlb-support)

### Envoy 支持使用 lua 脚本修改请求和响应体

参见：

1. [https://istio.io/docs/reference/config/networking/v1alpha3/envoy-filter/](https://istio.io/docs/reference/config/networking/v1alpha3/envoy-filter/)
2. [https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http\_filters/lua\_filter](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/lua_filter)

