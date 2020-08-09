---
title: 从xDS开始读懂istio的实现
date: 2020-08-08 13:13:00
tags: 
- Istio
- Pilot
- Envoy
- XDS
categories: 
- ServiceMesh
comments: true
---

Istio在架构上分为控制面和数据面两个部分，控制面指的是pilot、citadel、galley等用于发送配置、对网格进行管理的组件（1.6版本之后这些组件合并成istiod组件），数据面指的是以sidecar形式部署在工作负载边上用于劫持并处理业务流量的代理（默认是envoy）。

通俗地可以理解为Istio由一个指挥者（istiod）和多个干活的人（envoy）组成，由指挥者向干活的人下发的命令我们称之为xDS（发现协议），或者我们将xDS视做envoy用于如何处理流量的规则。

在istiod侧，存在一些配置概念（作为k8s的CRD）如vitualService/destinationRule，istiod通过将它们翻译成相应的xDS，再向envoy下发；通过定义这些CRD以及k8s中存在的service，我们可以很方便地定义服务治理的路由、流量权重、熔断等等规则，但这些配置究竟如何在envoy侧生效的呢，对于我们来说往往是个黑盒。

这篇文章的目的在于研究istiod是如何根据控制面侧的配置生成数据面侧所需的xDS协议数据。


## pilot中概念与xDS对应关系

为了直观地感受控制面配置变化对envoy的影响，本节直接通过创建不同资源的前后对比来显示不同资源转化成的xDS配置。

准备工作：
1. 安装istio 1.5.0+
2. kubectl label ns default istio-injection=enabled
3. kubectl apply -f demo.yaml # 创建一个pod，用于之后观察其sidecar envoy中的规则
    <details>
    <summary>demo.yaml</summary>

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
    name: demo
    spec:
    containers:
    - args:
        - "36000000000"
        command:
        - sleep
        image: centos:7.2.1511
        imagePullPolicy: IfNotPresent
        name: centos
    ```
    </details>
4. 观察目前envoy中的规则
    <details>
    <summary>listeners</summary>

    ```
    ☁  bin  ./istioctl proxy-config listener demo
    ADDRESS            PORT      TYPE
    10.244.0.186       15020     TCP
    10.97.23.35        15443     TCP
    10.102.119.234     443       TCP
    10.102.119.234     15443     TCP
    10.96.0.10         53        TCP
    10.101.83.24       443       TCP
    10.96.0.1          443       TCP
    10.110.220.255     15011     TCP
    10.110.220.255     15012     TCP
    10.101.83.24       15012     TCP
    10.97.23.35        31400     TCP
    10.110.220.255     443       TCP
    10.97.23.35        443       TCP
    0.0.0.0            20001     TCP
    10.96.0.10         9153      TCP
    10.97.23.35        15031     TCP
    10.97.23.35        15030     TCP
    10.100.78.178      16686     TCP
    0.0.0.0            8080      TCP
    10.97.23.35        15020     TCP
    0.0.0.0            15010     TCP
    10.97.23.35        15029     TCP
    0.0.0.0            14250     TCP
    10.108.46.165      14267     TCP
    0.0.0.0            9411      TCP
    10.97.23.35        15032     TCP
    0.0.0.0            3000      TCP
    10.108.46.165      14268     TCP
    0.0.0.0            15014     TCP
    10.108.46.165      14250     TCP
    0.0.0.0            80        TCP
    0.0.0.0            9090      TCP
    0.0.0.0            15001     TCP
    0.0.0.0            15006     TCP
    0.0.0.0            15090     HTTP
    ```
    </details>

    <details>
    <summary>clusters</summary>

    ```
    ☁  bin  ./istioctl proxy-config cluster demo
    SERVICE FQDN                                                 PORT      SUBSET         DIRECTION     TYPE
    BlackHoleCluster                                             -         -              -             STATIC
    InboundPassthroughClusterIpv4                                -         -              -             ORIGINAL_DST
    PassthroughCluster                                           -         -              -             ORIGINAL_DST
    grafana.istio-system.svc.cluster.local                       3000      -              outbound      EDS
    httpbin.org                                                  80        -              outbound      STRICT_DNS
    istio-egressgateway.istio-system.svc.cluster.local           80        -              outbound      EDS
    istio-egressgateway.istio-system.svc.cluster.local           443       -              outbound      EDS
    istio-egressgateway.istio-system.svc.cluster.local           15443     -              outbound      EDS
    istio-ingressgateway.istio-system.svc.cluster.local          80        -              outbound      EDS
    istio-ingressgateway.istio-system.svc.cluster.local          443       -              outbound      EDS
    istio-ingressgateway.istio-system.svc.cluster.local          15020     -              outbound      EDS
    istio-ingressgateway.istio-system.svc.cluster.local          15029     -              outbound      EDS
    istio-ingressgateway.istio-system.svc.cluster.local          15030     -              outbound      EDS
    istio-ingressgateway.istio-system.svc.cluster.local          15031     -              outbound      EDS
    istio-ingressgateway.istio-system.svc.cluster.local          15032     -              outbound      EDS
    istio-ingressgateway.istio-system.svc.cluster.local          15443     -              outbound      EDS
    istio-ingressgateway.istio-system.svc.cluster.local          31400     -              outbound      EDS
    istio-pilot.istio-system.svc.cluster.local                   443       -              outbound      EDS
    istio-pilot.istio-system.svc.cluster.local                   8080      -              outbound      EDS
    istio-pilot.istio-system.svc.cluster.local                   15010     -              outbound      EDS
    istio-pilot.istio-system.svc.cluster.local                   15011     -              outbound      EDS
    istio-pilot.istio-system.svc.cluster.local                   15012     -              outbound      EDS
    istio-pilot.istio-system.svc.cluster.local                   15014     -              outbound      EDS
    istiod.istio-system.svc.cluster.local                        443       -              outbound      EDS
    istiod.istio-system.svc.cluster.local                        15012     -              outbound      EDS
    jaeger-collector-headless.istio-system.svc.cluster.local     14250     -              outbound      ORIGINAL_DST
    jaeger-collector.istio-system.svc.cluster.local              14250     -              outbound      EDS
    jaeger-collector.istio-system.svc.cluster.local              14267     -              outbound      EDS
    jaeger-collector.istio-system.svc.cluster.local              14268     -              outbound      EDS
    jaeger-query.istio-system.svc.cluster.local                  16686     -              outbound      EDS
    kiali.istio-system.svc.cluster.local                         20001     -              outbound      EDS
    kube-dns.kube-system.svc.cluster.local                       53        -              outbound      EDS
    kube-dns.kube-system.svc.cluster.local                       9153      -              outbound      EDS
    kubernetes.default.svc.cluster.local                         443       -              outbound      EDS
    mgmtCluster                                                  15020     mgmt-15020     inbound       STATIC
    prometheus.istio-system.svc.cluster.local                    9090      -              outbound      EDS
    prometheus_stats                                             -         -              -             STATIC
    sds-grpc                                                     -         -              -             STATIC
    tracing.istio-system.svc.cluster.local                       80        -              outbound      EDS
    xds-grpc                                                     -         -              -             STRICT_DNS
    zipkin                                                       -         -              -             STRICT_DNS
    zipkin.istio-system.svc.cluster.local                        9411      -              outbound      EDS
    ```
    </details>

    <details>
    <summary>routes</summary>

    ```
    ☁  bin  ./istioctl proxy-config route demo  
    NOTE: This output only contains routes loaded via RDS.
    NAME                                                          VIRTUAL HOSTS
    istio-ingressgateway.istio-system.svc.cluster.local:15020     1
    jaeger-collector.istio-system.svc.cluster.local:14250         1
    jaeger-collector.istio-system.svc.cluster.local:14268         1
    kube-dns.kube-system.svc.cluster.local:9153                   1
    istio-ingressgateway.istio-system.svc.cluster.local:15029     1
    istio-ingressgateway.istio-system.svc.cluster.local:15031     1
    jaeger-collector.istio-system.svc.cluster.local:14267         1
    jaeger-query.istio-system.svc.cluster.local:16686             1
    istio-ingressgateway.istio-system.svc.cluster.local:15030     1
    istio-ingressgateway.istio-system.svc.cluster.local:15032     1
    80                                                            5
    3000                                                          2
    8080                                                          2
    9090                                                          2
    9411                                                          2
    14250                                                         3
    15010                                                         2
    15014                                                         2
    20001                                                         2
                                                                1
    ```
    </details>

    <details>
    <summary>endpoints</summary>

    ```
    ☁  bin  ./istioctl proxy-config endpoint demo
    ENDPOINT                        STATUS      OUTLIER CHECK     CLUSTER
    10.101.83.24:15012              HEALTHY     OK                xds-grpc
    10.244.0.173:8080               HEALTHY     OK                outbound|8080||istio-pilot.istio-system.svc.cluster.local
    10.244.0.173:15010              HEALTHY     OK                outbound|15010||istio-pilot.istio-system.svc.cluster.local
    10.244.0.173:15011              HEALTHY     OK                outbound|15011||istio-pilot.istio-system.svc.cluster.local
    10.244.0.173:15012              HEALTHY     OK                outbound|15012||istio-pilot.istio-system.svc.cluster.local
    10.244.0.173:15012              HEALTHY     OK                outbound|15012||istiod.istio-system.svc.cluster.local
    10.244.0.173:15014              HEALTHY     OK                outbound|15014||istio-pilot.istio-system.svc.cluster.local
    10.244.0.173:15017              HEALTHY     OK                outbound|443||istio-pilot.istio-system.svc.cluster.local
    10.244.0.173:15017              HEALTHY     OK                outbound|443||istiod.istio-system.svc.cluster.local
    10.244.0.175:3000               HEALTHY     OK                outbound|3000||grafana.istio-system.svc.cluster.local
    10.244.0.176:53                 HEALTHY     OK                outbound|53||kube-dns.kube-system.svc.cluster.local
    10.244.0.176:9153               HEALTHY     OK                outbound|9153||kube-dns.kube-system.svc.cluster.local
    10.244.0.177:53                 HEALTHY     OK                outbound|53||kube-dns.kube-system.svc.cluster.local
    10.244.0.177:9153               HEALTHY     OK                outbound|9153||kube-dns.kube-system.svc.cluster.local
    10.244.0.179:9090               HEALTHY     OK                outbound|9090||prometheus.istio-system.svc.cluster.local
    10.244.0.181:20001              HEALTHY     OK                outbound|20001||kiali.istio-system.svc.cluster.local
    10.244.0.182:9411               HEALTHY     OK                outbound|9411||zipkin.istio-system.svc.cluster.local
    10.244.0.182:14250              HEALTHY     OK                outbound|14250||jaeger-collector.istio-system.svc.cluster.local
    10.244.0.182:14267              HEALTHY     OK                outbound|14267||jaeger-collector.istio-system.svc.cluster.local
    10.244.0.182:14268              HEALTHY     OK                outbound|14268||jaeger-collector.istio-system.svc.cluster.local
    10.244.0.182:16686              HEALTHY     OK                outbound|16686||jaeger-query.istio-system.svc.cluster.local
    10.244.0.182:16686              HEALTHY     OK                outbound|80||tracing.istio-system.svc.cluster.local
    10.244.0.184:80                 HEALTHY     OK                outbound|80||istio-egressgateway.istio-system.svc.cluster.local
    10.244.0.184:443                HEALTHY     OK                outbound|443||istio-egressgateway.istio-system.svc.cluster.local
    10.244.0.184:15443              HEALTHY     OK                outbound|15443||istio-egressgateway.istio-system.svc.cluster.local
    10.97.98.215:9411               HEALTHY     OK                zipkin
    127.0.0.1:15000                 HEALTHY     OK                prometheus_stats
    127.0.0.1:15020                 HEALTHY     OK                inbound|15020|mgmt-15020|mgmtCluster
    192.168.31.176:6443             HEALTHY     OK                outbound|443||kubernetes.default.svc.cluster.local
    3.220.112.94:80                 HEALTHY     OK                outbound|80||httpbin.org
    54.236.246.173:80               HEALTHY     OK                outbound|80||httpbin.org
    unix:///etc/istio/proxy/SDS     HEALTHY     OK                sds-grpc
    ```
    </details>

### service

创建的service如下：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app
spec:
  selector:
    app: myapp
  ports:
  - port: 8888
    targetPort: 6666
    name: http-app
```

该service中端口协议为http（如果是tcp的话下面的规则有所不同）。

前后envoy规则对比如下：

#### listeners

新增一条规则。这条规则的特征是接收所有端口为8888的连接，而对地址无要求；这是由于创建的service中已经指明对于该服务的代理是L7的，因此会根据host再做路由。
```
ADDRESS            PORT      TYPE
0.0.0.0            8888      TCP
```

如果端口的协议是L4的话，那么此处ADDRESS会是service的clusterIP。

#### clusters

新增一条规则。cluster的命名直接用了service在k8s中的fqdn。
```
SERVICE FQDN                                                 PORT      SUBSET         DIRECTION     TYPE
app.default.svc.cluster.local                                8888      -              outbound      EDS
```

#### routes

新增一条规则。virtualHosts个数为2是因为其中一条是通配所有domain的。
```
NAME                                                          VIRTUAL HOSTS
8888                                                          2
```

<details>
<summary>route</summary>

```
☁  bin  ./istioctl proxy-config route demo --name app.default.svc.cluster.local:8888 -o json
[
    {
        "name": "8888",
        "virtualHosts": [
            {
                "name": "allow_any",
                "domains": [
                    "*"
                ],
                "routes": [
                    {
                        "match": {
                            "prefix": "/"
                        },
                        "route": {
                            "cluster": "PassthroughCluster",
                            "timeout": "0s"
                        }
                    }
                ]
            },
            {
                "name": "app.default.svc.cluster.local:8888",
                "domains": [
                    "app.default.svc.cluster.local",
                    "app.default.svc.cluster.local:8888",
                    "app",
                    "app:8888",
                    "app.default.svc.cluster",
                    "app.default.svc.cluster:8888",
                    "app.default.svc",
                    "app.default.svc:8888",
                    "app.default",
                    "app.default:8888",
                    "10.105.148.44",
                    "10.105.148.44:8888"
                ],
                "routes": [
                    {
                        "name": "default",
                        "match": {
                            "prefix": "/"
                        },
                        "route": {
                            "cluster": "outbound|8888||app.default.svc.cluster.local",
                            "timeout": "0s",
                            "retryPolicy": {
                                "retryOn": "connect-failure,refused-stream,unavailable,cancelled,resource-exhausted,retriable-status-codes",
                                "numRetries": 2,
                                "retryHostPredicate": [
                                    {
                                        "name": "envoy.retry_host_predicates.previous_hosts"
                                    }
                                ],
                                "hostSelectionRetryMaxAttempts": "5",
                                "retriableStatusCodes": [
                                    503
                                ]
                            },
                            "maxGrpcTimeout": "0s"
                        },
                        "decorator": {
                            "operation": "app.default.svc.cluster.local:8888/*"
                        }
                    }
                ]
            }
        ],
        "validateClusters": false
    }
]

```
</details>

### pod

创建的pod如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
  labels:
    app: myapp
    version: v1
spec:
  containers:
  - args:
    - "36000000000"
    command:
    - sleep
    image: centos:7.2.1511
    imagePullPolicy: IfNotPresent
    name: centos
```

envoy前后规则变化如下：

#### endpoints

新增一条规则。istio通过service的selector选到了pod，将其作为服务的实例。

```
ENDPOINT                        STATUS      OUTLIER CHECK     CLUSTER
10.244.0.187:6666               HEALTHY     OK                outbound|8888||app.default.svc.cluster.local
```

### destinationRule

```
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: app
spec:
  host: app
  subsets:
  - name: v1
    labels:
      version: v1
```

#### clusters

新增一条规则。通过destinationRule将cluster拆分成多个的cluster。
```
SERVICE FQDN                                                 PORT      SUBSET         DIRECTION     TYPE
app.default.svc.cluster.local                                8888      v1             outbound      EDS
```

### virtualService

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: app
spec:
  hosts:
  - app
  http:
  - route:
    - destination:
        host: app
        subset: v1
```

#### routes

路由规则的变化在其route的cluster变为了virtualService中指定的subset=v1的cluster。

<details>
<summary>route</summary>

```
☁  bin  ./istioctl proxy-config route demo  --name 8888 -o json
[
    {
        "name": "8888",
        "virtualHosts": [
            {
                "name": "allow_any",
                "domains": [
                    "*"
                ],
                "routes": [
                    {
                        "match": {
                            "prefix": "/"
                        },
                        "route": {
                            "cluster": "PassthroughCluster",
                            "timeout": "0s"
                        }
                    }
                ]
            },
            {
                "name": "app.default.svc.cluster.local:8888",
                "domains": [
                    "app.default.svc.cluster.local",
                    "app.default.svc.cluster.local:8888",
                    "app",
                    "app:8888",
                    "app.default.svc.cluster",
                    "app.default.svc.cluster:8888",
                    "app.default.svc",
                    "app.default.svc:8888",
                    "app.default",
                    "app.default:8888",
                    "10.105.148.44",
                    "10.105.148.44:8888"
                ],
                "routes": [
                    {
                        "match": {
                            "prefix": "/"
                        },
                        "route": {
                            "cluster": "outbound|8888|v1|app.default.svc.cluster.local",
                            "timeout": "0s",
                            "retryPolicy": {
                                "retryOn": "connect-failure,refused-stream,unavailable,cancelled,resource-exhausted,retriable-status-codes",
                                "numRetries": 2,
                                "retryHostPredicate": [
                                    {
                                        "name": "envoy.retry_host_predicates.previous_hosts"
                                    }
                                ],
                                "hostSelectionRetryMaxAttempts": "5",
                                "retriableStatusCodes": [
                                    503
                                ]
                            },
                            "maxGrpcTimeout": "0s"
                        },
                        "metadata": {
                            "filterMetadata": {
                                "istio": {
                                    "config": "/apis/networking.istio.io/v1alpha3/namespaces/default/virtual-service/app"
                                }
                            }
                        },
                        "decorator": {
                            "operation": "app.default.svc.cluster.local:8888/*"
                        }
                    }
                ]
            }
        ],
        "validateClusters": false
    }
]

```
</details>

## pilot源码实现

istiod的主体是pilot。pilot分为pilot-discovery和pilot-agent两部分，前者集成在istiod中，后者部署在sidecar中，管理envoy的生命周期。
pilot-discovery会将来自kubernetes/mcp/配置文件中的配置信息转化xDS协议数据推送给envoy。

主要模块有3个：
1. ServiceController：服务发现控制器，从kubernetes、serviceentry（也是k8s中的crd）中获取服务和服务实例地址(对consul的支持在1.6.x中移除)
2. ConfigController：配置管理控制器，从kubernetes、mcp、file中获取流量治理策略的配置
3. DiscoveryService：将ServiceController和ConfigController中的数据转换成xDS协议下发给envoy
    1. DiscoveryService的实现中，ConfigGenerator负责取ConfigController配置
    2. EndpointShardsByService负责存放服务发现数据，由ServiceController维护
    3. Generators负责生成xDS格式的数据 

![pilot-discovery](pilot-discovery.png)

pilot-discovery与envoy通信建立在grpc连接上，当控制面监听到配置的变化时向envoy推送数据如下图所示：

![channel](channel.png)