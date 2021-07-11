---
title: 使用istio的tracing功能的一个简单例子
date: 2021-7-11 14:37:00
tags: 
- Istio
- Tracing
- OpenTelemetry
- Jaeger
categories: 
- ServiceMesh
comments: true
---

Istio内置了链路追踪的能力（调用链），依赖的是envoy能够在劫持的流量中自动注入用于构成链路的Trace信息（例如OpenZipkin的b3 propagation，在Http中的header是X-B3-xxx），并且将该信息上报给外部的Trace服务（例如LightStep/Zipkin/Jaeger/Datadog等）。

而为了将这种能力进化成`分布式`的链路追踪，需要程序能将入站请求中的Trace信息传递给出站请求(参考 https://istio.io/docs/tasks/telemetry/distributed-tracing/overview/#trace-context-propagation ),用户的业务代码因此需要进行修改。
Istio宣称的所谓的对业务代码"零侵入"是不可靠的，业务代码不进行任何更改而想享受分布式追踪的能力，除非是以下场景 
1. 程序使用了特殊的框架，能够自动从上下文中获取入站请求的Trace信息并注入到出站请求中。参考后文给的代码示例，我们将通过指导或提供我们的sdk仅可能地简化用户代码中对`Trace信息传递`这一逻辑。
2. 分布式追踪退化到仅能追溯一跳请求。假设用户并不对Trace信息进行传递，那么如果有服务调用关系为`A->B->C`，最终通过调用链我们仅能查看到`A->B,B->C`。

当我们构建起完整的链路追踪，我们在jaeger上得到的trace信息将如下图所示：（从中我们能获取服务间的调用关系/调用的耗时/服务内部处理的耗时/错误的源头）

![jaeger-example](jaeger-example.png)

## Istio中Trace实现原理

作为Sidecar的envoy会自动生成TraceId和SpanId并向Jaeger/Zipkin等上报，可以通过下方示意图理解。

![trace_spans_flow](trace_spans_flow.png)

1. TraceId和第一个SpanId既可以由input的proxy生成也可以由ouput的proxy生成，也就是说调用链的第一个请求来自于网格外或网格内都可以生成Trace信息
2. Sampled=1时，envoy才会将Span上报。我们的程序在传递Trace信息时一般将Sampled信息原样传递，因此第一个请求中的Sampled尤为重要；我们通过给第一个请求主动配置相应header或通过ingressgateway访问服务时Sampled才会等于1
3. 通过Span的时间戳我们可以计算网络的耗时和程序自身的耗时，以上图为例：SpanId=2的span的总耗时减SpanId=3的span的总耗时约等于服务B的耗时（我们将本地通信的耗时忽略不计）;SpanId=1的span起始时间减去SpanId=2的span的起始时间约等于服务A请求到服务B请求的网络耗时（envoy处理请求的耗时忽略不计）

## 分布式追踪示例

在[tracing-example](https://github.com/pangsq/tutorials/tree/main/servicemesh/tracing下)中提供了一个简单例子，其中包含一个go的服务(go-service)和一个java的服务(java-service)，他们的逻辑相同，都是:
1. 将请求转发给预先设置的后置服务（go-service -> java-service, java-service -> go-service），保留path信息并传递trace信息 
2. queryParam参数中包含一个ttl值，每次转发ttl减1，当减到0时，将请求转发给httpbin

![forward](forward.png)

`go-service`中用于在http request间传递Trace信息的方式是通过context.Context，生成和使用该context见[propagators](https://github.com/pangsq/tutorials/blob/main/servicemesh/tracing/go-service/tracing/propagators.go)

`java-service`中的方式是通过自动生成spring的scoped=request的bean,详见[TraceContext](https://github.com/pangsq/tutorials/blob/main/servicemesh/tracing/java-service/src/main/java/com/example/restservice/TracingContext.java)，将Span的信息存储在该bean中，这种做法能保证无论outgoing的请求生成在哪个controller或service中都能拿到相关上下文input请求中的Span信息

两个服务的编译方式和部署方式见各项目中的`README.md`，httpbin的部署见[httpbin.yaml](https://github.com/pangsq/tutorials/blob/main/servicemesh/tracing/httpbin.yaml)

在将服务部署好后通过网关访问

![curl_go_service_example](curl_go_service_example.png)

在jaeger上查看trace (通过调用链我们可以看到请求在go-service和java-service间反复横跳

![curl_go_service_result](curl_go_service_result.png)

## Ref:

- [envoy-tracing](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/observability/tracing)
- [envoy-headers](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_conn_man/headers#x-b3-traceid)
- [jaeger-architecture](https://www.jaegertracing.io/docs/1.23/architecture/)
- [洞若观火：使用OpenTracing增强Istio的调用链跟踪](https://zhaohuabing.com/post/2019-06-22-using-opentracing-with-istio/)
- [Istio 调用链埋点原理剖析—是否真的“零修改”？](https://www.infoq.cn/article/pqy*pfphox9oqq9icrtt)