# istio listener构建
istio listener需要构建以下集中：
```go
// buildSidecarListeners produces a list of listeners for sidecar proxies
func (configgen *ConfigGeneratorImpl) buildSidecarListeners(builder *ListenerBuilder) *ListenerBuilder {
	if builder.push.Mesh.ProxyListenPort > 0 {
		// Any build order change need a careful code review
		builder.appendSidecarInboundListeners().
			appendSidecarOutboundListeners().
			buildHTTPProxyListener().
			buildVirtualOutboundListener()
	}
	return builder
}
```
我们先来看一下实际例子：
![[Pasted image 20230426150133.png]]
我们知道pod中的enovy sidecar是通过iptable将入口流量重定向到了15006端口，由envoy代理，最后再转发到服务中。在[-Istio 中的 Sidecar 注入、透明流量劫持及流量路由过程详解](https://jimmysong.io/blog/sidecar-injection-iptables-and-traffic-routing/)这篇文章中详细说明了inbound和outbound流量的代理流程，简单来说，外部流量->iptables劫持匹配入站规则->envoy inbound 15006->匹配route，再找到cluster->iptable劫持匹配出站规则，假设源地址为127.0.0.6->透传到pod内应用服务。这里不再赘述，本篇也只关注appendSidecarInboundListeners构建inbound listener流程，outbound listener后续再写。
查看inbound listener的配置：
```json
// ./istioctl pc listeners ratings-v1-85cc46b6d4-4kngk --port 15006 -ojson
[
    {
        "name":"virtualInbound",
        "address":{
            "socketAddress":{
                "address":"0.0.0.0",
                "portValue":15006
            }
        },
        "filterChains":[
            Object{...},
            Object{...},
            Object{...},
            Object{...},
            Object{...},
            Object{...},
            {
                "filterChainMatch":{
                    "destinationPort":9080,
                    "transportProtocol":"tls",
                    "applicationProtocols":[
                        "istio",
                        "istio-peer-exchange",
                        "istio-http/1.0",
                        "istio-http/1.1",
                        "istio-h2"
                    ]
                },
                "filters":[
                    Object{...},
                    Object{...},
                    {
                        "name":"envoy.filters.network.http_connection_manager",
                        "typedConfig":{
                            "@type":"type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager",
                            "statPrefix":"inbound_0.0.0.0_9080",
                            "routeConfig":{
                                "name":"inbound|9080||",
                                "virtualHosts":[
                                    {
                                        "name":"inbound|http|9080",
                                        "domains":[
                                            "*"
                                        ],
                                        "routes":[
                                            {
                                                "name":"default",
                                                "match":{
                                                    "prefix":"/"
                                                },
                                                "route":{
                                                    "cluster":"inbound|9080||",
                                                    "timeout":"0s",
                                                    "maxStreamDuration":{
                                                        "maxStreamDuration":"0s",
                                                        "grpcTimeoutHeaderMax":"0s"
                                                    }
                                                },
                                                "decorator":{
                                                    "operation":"ratings.default.svc.cluster.local:9080/*"
                                                }
                                            }
                                        ]
                                    }
                                ],
                                "validateClusters":false
                            },
                            "httpFilters":[
                                Object{...},
                                Object{...},
                                Object{...},
                                Object{...},
                                {
                                    "name":"envoy.filters.http.router",
                                    "typedConfig":{
                                        "@type":"type.googleapis.com/envoy.extensions.filters.http.router.v3.Router"
                                    }
                                }
                            ],
                            "tracing":Object{...},
                            "serverName":"istio-envoy",
                            "streamIdleTimeout":"0s",
                            "accessLog":Array[1],
                            "useRemoteAddress":false,
                            "forwardClientCertDetails":"APPEND_FORWARD",
                            "setCurrentClientCertDetails":{
                                "subject":true,
                                "dns":true,
                                "uri":true
                            },
                            "upgradeConfigs":[
                                {
                                    "upgradeType":"websocket"
                                }
                            ],
                            "normalizePath":true,
                            "pathWithEscapedSlashesAction":"KEEP_UNCHANGED",
                            "requestIdExtension":{
                                "typedConfig":{
                                    "@type":"type.googleapis.com/envoy.extensions.request_id.uuid.v3.UuidRequestIdConfig",
                                    "useRequestIdForTraceSampling":true
                                }
                            }
                        }
                    }
                ],
                "transportSocket":Object{...},
                "name":"0.0.0.0_9080"
            },
            Object{...}
        ],
        "listenerFilters":Array[3],
        "listenerFiltersTimeout":"0s",
        "continueOnListenerFiltersTimeout":true,
        "trafficDirection":"INBOUND",
        "accessLog":Array[1]
    }
]
```
可以看到此listener名称为virtualInbound，监听本地的15006端口，其中filter_chains有多项，这里只展示了上面截图中DESTINATION为Cluster: inbound|9080||的过滤链，查看此cluster如下，会将源地址设置为127.0.0.6，type为ORIGINAL_DST，即目的地址保持downstream请求的目的地址，envoy不进行干预改写，这是cluster服务发现的一种，我们通常用的还有eds，即需要设置endpoint地址，还有dns，具体可以查看envoy文档。当inbound流量经过envoy再出来时还会进入iptables，iptables规则中识别到源地址为127.0.0.6，则会透传给业务容器。
```shell
[root@ocean bin]# ./istioctl pc clusters ratings-v1-85cc46b6d4-4kngk --fqdn "inbound|9080||" -ojson
[
    {
        "name": "inbound|9080||",
        "type": "ORIGINAL_DST",
        "connectTimeout": "10s",
        "lbPolicy": "CLUSTER_PROVIDED",
        "circuitBreakers": {
            "thresholds": [
                {
                    "maxConnections": 4294967295,
                    "maxPendingRequests": 4294967295,
                    "maxRequests": 4294967295,
                    "maxRetries": 4294967295,
                    "trackRemaining": true
                }
            ]
        },
        "upstreamBindConfig": {
            "sourceAddress": {
                "address": "127.0.0.6",
                "portValue": 0
            }
        },
        "commonLbConfig": {},
        "metadata": {
            "filterMetadata": {
                "istio": {
                    "services": [
                        {
                            "host": "ratings.default.svc.cluster.local",
                            "name": "ratings",
                            "namespace": "default"
                        }
                    ]
                }
            }
        }
    }
]
```
这里我们思考一个问题，我们是否可以改写规则，让流量不进入业务容器？可以的，这正是我们之前说的sidecar crd中ingress listener做的事情，改写默认的inbound listener。
通过此例，我们对inbound listener有了一定的认识，然后来探究下inbound listener规则是如何生成的。
# inbound listener生成
构建inbound listeners(需要[sidecarScope](https://github.com/weiwei852346/istio-deep-dive/tree/master/sidecar-crd)z中的service, envoy filter crd, sidecar crd资源)，主要逻辑在*buildInboundListeners, pilot/pkg/networking/core/v1alpha3/listener_inbound.go* 中
1. 如果sidecarScope没有找到sidecar ingress listener应用到此node proxy，如果存在，则inbound的listener则会通过sidecar ingress中的配置生成，可以参考sidecarScope中ingress listener部分理解。
2. 如果不存在sidecar，则需要遍历node proxy中的serviceInstances，根据instances目的端口协议构建[]inboundChainConfig
   这也是和存在sidecar时生成inboundChainConfig的主要区别，存在sidecar时，则需要根据sidcar的ingress中配置的服务去生成。
   serviceInstances为node proxy中跑的业务服务的抽象，即默认的inbound listener会将目的设置为本地的业务服务中，但是如果用sidecar，就可以随便配，将inbound流量重定向到任意地方。
   这里的service instances也是istio中抽象的概念，定义如下
```go
   // There are two reasons why this returns multiple ServiceInstances instead of one:  
// - A ServiceInstance has a single IstioEndpoint which has a single Port.  But a Service  
//   may have many ports.  So a workload implementing such a Service would need//   multiple ServiceInstances, one for each port.  
// - A single workload may implement multiple logical Services.  
//  
// In the second case, multiple services may be implemented by the same physical port number,  
// though with a different ServicePort and IstioEndpoint for each.  If any of these overlapping// services are not HTTP or H2-based, behavior is undefined, since the listener may not be able to  
// determine the intended destination of a connection without a Host header on the request.
 type ServiceInstance struct {  
   Service     *Service       `json:"service,omitempty"`  
   ServicePort *Port          `json:"servicePort,omitempty"`  
   Endpoint    *IstioEndpoint `json:"endpoint,omitempty"`  
 }
```
   同sidecarscope一样，也是从pushContext中找到node proxy相关的serviceInstances，node proxy中的serviceInstance是是个数组，或者是一个服务有多个端口，那么服务的每个端口都用一个serviceInstance去描述，也可能存在一个负载对应多个逻辑服务，这多个逻辑服务共用一个物理端口。随后serviceInstance将会转化成inboundChainConfig，然后被用于去生成inbound listener，如上面例子中inbound|9080||。
3. inboundChainConfig中主要属性
   port 9080
   clusterName  如inbound|9080||
   bindToPort  bindToPort会true时，表示inbound listener需要绑定到端口，最后会生成一个9080的listener，且每个inboundChainConfig都生成一个新的listener并将其转化为envoy的filter_chain，而如果为false，则将inboundChainConfig转化为filter_chain都添加到名为virtualInbound的listener中。
4. inboundChainConfig到envoy filter_chain的转化需要构建FilterChainMatch，filters，TransportSocket几部分
5. 首先filters的设置，位于*lb.buildInboundNetworkFiltersForHTTP(cc)*。先判断协议，如果为http，需要构建一些网络过滤器，其他网络过滤器先略过，最重要的为http_connection_manager且必须位于最后一项，为tcp流程较为简单，略过不谈。
6. http_connection_manager流程较为简单清晰，其中的路由配置，在inbound listener中是静态配置的，参见上面的inbound listener例子中的virtualHost，其name即为inbound|http|端口，Domain为*，匹配所有，routes则为默认生成的路由，默认路由中最为关键的cluster配置也是通过端口自动构建出来的，cds cluster创建时，也遵循这个规则。
   需要注意的是，路由的配置并非内置死，是可以通过envoy filter对路由进行配置的，构建完inbound默认路由后，添加了envoy filter监测点(*buildSidecarInboundHTTPRouteConfig*中，`r = envoyfilter.ApplyRouteConfigurationPatches(networking.EnvoyFilter_SIDECAR_INBOUND, lb.node, efw, r)`)，对路由进行patch。envoy filter暂且不说，后续专门说下envoy filter的类型和编写，贴一下envoy filter的类型
```go
   const (
	EnvoyFilter_INVALID EnvoyFilter_ApplyTo = 0
	// Applies the patch to the listener.
	EnvoyFilter_LISTENER EnvoyFilter_ApplyTo = 1
	// Applies the patch to the filter chain.
	EnvoyFilter_FILTER_CHAIN EnvoyFilter_ApplyTo = 2
	// Applies the patch to the network filter chain, to modify an
	// existing filter or add a new filter.
	EnvoyFilter_NETWORK_FILTER EnvoyFilter_ApplyTo = 3
	// Applies the patch to the HTTP filter chain in the http
	// connection manager, to modify an existing filter or add a new
	// filter.
	EnvoyFilter_HTTP_FILTER EnvoyFilter_ApplyTo = 4
	// Applies the patch to the Route configuration (rds output)
	// inside a HTTP connection manager. This does not apply to the
	// virtual host. Currently, only `MERGE` operation is allowed on the
	// route configuration objects.
	EnvoyFilter_ROUTE_CONFIGURATION EnvoyFilter_ApplyTo = 5
	// Applies the patch to a virtual host inside a route configuration.
	EnvoyFilter_VIRTUAL_HOST EnvoyFilter_ApplyTo = 6
	// Applies the patch to a route object inside the matched virtual
	// host in a route configuration.
	EnvoyFilter_HTTP_ROUTE EnvoyFilter_ApplyTo = 7
	// Applies the patch to a cluster in a CDS output. Also used to add new clusters.
	EnvoyFilter_CLUSTER EnvoyFilter_ApplyTo = 8
	// Applies the patch to or adds an extension config in ECDS output. Note that ECDS
	// is only supported by HTTP filters.
	EnvoyFilter_EXTENSION_CONFIG EnvoyFilter_ApplyTo = 9
	// Applies the patch to bootstrap configuration.
	EnvoyFilter_BOOTSTRAP EnvoyFilter_ApplyTo = 10
	// Applies the patch to the listener filter.
	EnvoyFilter_LISTENER_FILTER EnvoyFilter_ApplyTo = 11
)
```
7. 除了rds配置，http_connection_mananger还需要添加http filters，包括
     - istio.metadata_exchange
     - envoy.filters.http.fault
     - envoy.filters.http.cors
     - istio.stats
     - envoy.filters.http.router
    envoy.filters.http.router必须为最后一项，配置较为简单。
8. 上面只是说了DESTINATION为Cluster: inbound|9080||的filter_chain生成，在virtualInbound filter中，还包括InboundPassthroughClusterIpv4这个filter_chain(*buildInboundPassthroughChains*)，服务发现为ORIGINAL_DST，透传请求原始目的地址，至于什么时候会用到？有一种情况，即k8s没有创建服务service，此时istio不会探测到service，也不会下发上面的 inbound|9080|| listerner规则，这时候我们如果直接在其他容器中用pod ip访问，则会命中这条规则，然后透传到业务服务中。
```shell
   [root@ocean bin]# ./istioctl pc clusters ratings-v1-85cc46b6d4-4kngk --fqdn InboundPassthroughClusterIpv4 -ojson
[
    {
        "name": "InboundPassthroughClusterIpv4",
        "type": "ORIGINAL_DST",
        "connectTimeout": "10s",
        "lbPolicy": "CLUSTER_PROVIDED",
        "circuitBreakers": {
            "thresholds": [
                {
                    "maxConnections": 4294967295,
                    "maxPendingRequests": 4294967295,
                    "maxRequests": 4294967295,
                    "maxRetries": 4294967295,
                    "trackRemaining": true
                }
            ]
        },
        "typedExtensionProtocolOptions": {
            "envoy.extensions.upstreams.http.v3.HttpProtocolOptions": {
                "@type": "type.googleapis.com/envoy.extensions.upstreams.http.v3.HttpProtocolOptions",
                "commonHttpProtocolOptions": {
                    "idleTimeout": "300s"
                },
                "useDownstreamProtocolConfig": {
                    "httpProtocolOptions": {},
                    "http2ProtocolOptions": {}
                }
            }
        },
        "upstreamBindConfig": {
            "sourceAddress": {
                "address": "127.0.0.6",
                "portValue": 0
            }
        }
    }
]
```




