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

### service

### pod

### virtualService

### destinationRule

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

<img src="pilot-discovery.png" width="1000" height="600" align="center" />

pilot-discovery与envoy通信建立在grpc连接上，当控制面监听到配置的变化时向envoy推送数据如下图所示：

<img src="channel.png" width="1000" height="600" align="center" />