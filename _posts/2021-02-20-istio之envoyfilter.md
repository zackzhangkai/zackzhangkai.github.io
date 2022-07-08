---
layout: post
published: true
title:  istio之envoyfilter
categories: [document]
tags: [istio]
---
* content
{:toc}

## envoyfilter

支持直接修改envoy的配置文件。

官网：  
<https://istio.io/latest/docs/reference/config/networking/envoy-filter/>

istio 1.6.10 自身带了四个：

```bash
# kubectl get envoyfilter -n istio-system
NAME                               AGE
metadata-exchange-1.6-1-6-10       8d
stats-filter-1.6-1-6-10            8d
tcp-metadata-exchange-1.6-1-6-10   8d
tcp-stats-filter-1.6-1-6-10        8d
```

envoyfilter不同的版本，差别很大

listener 分v2 v3版本

如在istio 1.6中给一个header增加header：

```bash
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: lua-1.6
spec:
  configPatches:
    - applyTo: HTTP_FILTER
      match:
        context: ANY
        listener:
          filterChain:
            filter:
              name: envoy.http_connection_manager
        proxy:
          proxyVersion: ^1\.6.*
      patch:
        operation: INSERT_BEFORE
        value:
          name: envoy.lua
          config:
            inlineCode: |
              function envoy_on_request(handle)
                request_handle:headers():add("foo", "bar")
              end
```

在istio 1.7中：

```bash
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: lua-1.7
spec:
  configPatches:
    - applyTo: HTTP_FILTER
      match:
        context: ANY
        listener:
          filterChain:
            filter:
              name: envoy.http_connection_manager
        proxy:
          proxyVersion: ^1\.7.*
      patch:
        operation: INSERT_BEFORE
        value:
          name: envoy.filters.http.lua
          typed_config:
            '@type': type.googleapis.com/envoy.extensions.filters.http.lua.v3.Lua
            inlineCode: |
              function envoy_on_request(handle)
                request_handle:headers():add("foo", "bar")
              end
```

看其中的一个的完整内容：

```yaml
spec:
  configPatches:
  - applyTo: HTTP_FILTER
    match:
      context: SIDECAR_OUTBOUND
      listener:
        filterChain:
          filter:
            name: envoy.http_connection_manager
            subFilter:
              name: envoy.router
      proxy:
        proxyVersion: ^1\.6.*
    patch:
      operation: INSERT_BEFORE
      value:
        name: istio.stats
        typed_config:
          '@type': type.googleapis.com/udpa.type.v1.TypedStruct
          type_url: type.googleapis.com/envoy.extensions.filters.http.wasm.v3.Wasm
          value:
            config:
              configuration: |
                {
                  "debug": "false",
                  "stat_prefix": "istio"
                }
              root_id: stats_outbound
              vm_config:
                code:
                  local:
                    inline_string: envoy.wasm.stats
                runtime: envoy.wasm.runtime.null
                vm_id: stats_outbound
  - applyTo: HTTP_FILTER
    match:
      context: SIDECAR_INBOUND
      listener:
        filterChain:
          filter:
            name: envoy.http_connection_manager
            subFilter:
              name: envoy.router
      proxy:
        proxyVersion: ^1\.6.*
    patch:
      operation: INSERT_BEFORE
      value:
        name: istio.stats
        typed_config:
          '@type': type.googleapis.com/udpa.type.v1.TypedStruct
          type_url: type.googleapis.com/envoy.extensions.filters.http.wasm.v3.Wasm
          value:
            config:
              configuration: |
                {
                  "debug": "false",,
                  "stat_prefix": "istio"
                }
              root_id: stats_inbound
              vm_config:
                code:
                  local:
                    inline_string: envoy.wasm.stats
                runtime: envoy.wasm.runtime.null
                vm_id: stats_inbound
  - applyTo: HTTP_FILTER
    match:
      context: GATEWAY
      listener:
        filterChain:
          filter:
            name: envoy.http_connection_manager
            subFilter:
              name: envoy.router
      proxy:
        proxyVersion: ^1\.6.*
    patch:
      operation: INSERT_BEFORE
      value:
        name: istio.stats
        typed_config:
          '@type': type.googleapis.com/udpa.type.v1.TypedStruct
          type_url: type.googleapis.com/envoy.extensions.filters.http.wasm.v3.Wasm
          value:
            config:
              configuration: |
                {
                  "debug": "false",
                  "stat_prefix": "istio",
                  "disable_host_header_fallback": true
                }
              root_id: stats_outbound
              vm_config:
                code:
                  local:
                    inline_string: envoy.wasm.stats
                runtime: envoy.wasm.runtime.null
                vm_id: stats_outbound
```

支持项如下：

```bash
EnvoyFilter
EnvoyFilter.ProxyMatch
EnvoyFilter.ClusterMatch
EnvoyFilter.RouteConfigurationMatch
EnvoyFilter.ListenerMatch
EnvoyFilter.Patch
EnvoyFilter.EnvoyConfigObjectMatch
EnvoyFilter.EnvoyConfigObjectPatch
EnvoyFilter.RouteConfigurationMatch.RouteMatch
EnvoyFilter.RouteConfigurationMatch.VirtualHostMatch
EnvoyFilter.ListenerMatch.FilterChainMatch
EnvoyFilter.ListenerMatch.FilterMatch
EnvoyFilter.ListenerMatch.SubFilterMatch
EnvoyFilter.RouteConfigurationMatch.RouteMatch.Action
EnvoyFilter.Patch.Operation
EnvoyFilter.ApplyTo
EnvoyFilter.PatchContext
```

## 几个示例

示例一：

```bash
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: kubesphere-router-user-ip
  namespace: kubesphere-controls-system
spec:
  workloadLabels:
    component: ks-router
  filters:
    - listenerMatch:
        portNumber: 80
        listenerType: ANY
      filterName: envoy.lua
      filterType: HTTP
      filterConfig:
        inlineCode: |
          function envoy_on_request(request_handle)
            local xff_header = request_handle:headers():get("X-Forwarded-For")
            local first_ip = string.gmatch(xff_header, "(%d+.%d+.%d+.%d+)")();
            request_handle:headers():add("X-Custom-User-IP", first_ip);
          end
```

示例二：

```bash
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: proxy-protocol
  namespace: kubesphere-controls-system
spec:
  configPatches:
  - applyTo: LISTENER
    patch:
      operation: MERGE
      value:
        listener_filters:
        - name: envoy.listener.proxy_protocol
        - name: envoy.listener.tls_inspector
  workloadSelector:
    labels:
      component: ks-router
```

示例三，给所有的经过sidecar的容器加一个http header

```bash
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: add-user-agent
  namespace: kubesphere-controls-system
spec:
  workloadLabels:
    component: ks-router
  configPatches:
  - applyTo: NETWORK_FILTER
    match:
      context: SIDECAR_OUTBOUND
      listener:
        filterChain:
          filter:
            name: envoy.filters.network.http_connection_manager
    patch:
      operation: MERGE
      value:
        typedConfig:
          '@type': type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          use_remote_address: true
          skip_xff_append: false
```

此时，所有被 sidecar 代理的 HTTP 出流量都会带上这个头部：x-envoy-downstream-service-cluster （参考envoy文档)[https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_conn_man/headers#x-envoy-downstream-service-cluster]。

<https://istio.io/latest/docs/reference/config/networking/envoy-filter/>
<https://www.zhihu.com/people/tang-yang-31-92>

示例四：

```bash
✗  k get envoyfilter -n kubesphere-controls-system request-id-settings -oyaml

apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  annotations:
  name: request-id-settings
  namespace: kubesphere-controls-system
  resourceVersion: "14729859"
spec:
  configPatches:
  - applyTo: NETWORK_FILTER
    match:
      context: ANY
      listener:
        filterChain:
          filter:
            name: envoy.http_connection_manager
    patch:
      operation: MERGE
      value:
        typed_config:
          '@type': type.googleapis.com/envoy.config.filter.network.http_connection_manager.v2.HttpConnectionManager
          preserve_external_request_id: true
  workloadSelector:
    labels:
      app: kubesphere
      component: ks-router
      project: test
      tier: backend
```

示例，istio 1.6.10，添加外部访问IP

```bash
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  annotations:
  name: use-remote-address
  namespace: istio-system
spec:
  configPatches:
  - applyTo: NETWORK_FILTER
    match:
      context: SIDECAR_OUTBOUND
      listener:
        filterChain:
          filter:
            name: envoy.http_connection_manager
    patch:
      operation: MERGE
      value:
        typedConfig:
          '@type': type.googleapis.com/envoy.config.filter.network.http_connection_manager.v2.HttpConnectionManager
          skip_xff_append: false
          use_remote_address: true
          xff_num_trusted_hops: 10
```

<https://zhuanlan.zhihu.com/p/351690052>  
<https://istio.io/latest/docs/reference/config/networking/envoy-filter/>  
<https://istio.io/latest/news/releases/1.7.x/announcing-1.7/upgrade-notes/>  
