# 设计稳定的微服务系统时不得不考虑的场景

我们的生产环境经常会出现一些不稳定的情况，如：

- 大促时瞬间洪峰流量导致系统超出最大负载，load 飙高，系统崩溃导致用户无法下单
- “黑马”热点商品击穿缓存，DB 被打垮，挤占正常流量
- 调用端被不稳定服务拖垮，线程池被占满，导致整个调用链路卡死

这些不稳定的场景可能会导致严重后果。大家可能想问：如何做到均匀平滑的用户访问？如何预防流量过大或服务不稳定带来的影响？

## 介绍

下面两种方式是在面对流量不稳定因素时常见的两种方案，也是我们在涉及高可用的系统前不得不考虑的两种能力，是服务流量治理中非常关键的一环。

### 流量控制

流量是非常随机性的、不可预测的。前一秒可能还风平浪静，后一秒可能就出现流量洪峰了（例如双十一零点的场景）。每个系统、服务都有其能承载的容量上限，如果突然而来的流量超过了系统的承受能力，就可能会导致请求处理不过来，堆积的请求处理缓慢，CPU/Load 飙高，最后导致系统崩溃。因此，我们需要针对这种突发的流量来进行限制，在尽可能处理请求的同时来保障服务不被打垮，这就是流量控制。

### 熔断降级

一个服务常常会调用别的模块，可能是另外的一个远程服务、数据库，或者第三方 API 等。例如，支付的时候，可能需要远程调用银联提供的 API；查询某个商品的价格，可能需要进行数据库查询。然而，这个被依赖服务的稳定性是不能保证的。如果依赖的服务出现了不稳定的情况，请求的响应时间变长，那么调用服务的方法的响应时间也会变长，线程会产生堆积，最终可能耗尽业务自身的线程池，服务本身也变得不可用。

![dependency](https://user-images.githubusercontent.com/9434884/62410811-cd871680-b61d-11e9-9df7-3ee41c618644.png)

现代微服务架构都是分布式的，由非常多的服务组成。不同服务之间相互调用，组成复杂的调用链路。以上的问题在链路调用中会产生放大的效果。复杂链路上的某一环不稳定，就可能会层层级联，最终导致整个链路都不可用。因此我们需要对不稳定的弱依赖服务进行熔断降级，暂时切断不稳定调用，避免局部不稳定因素导致整体的雪崩。

> **NOTE**：不少同学在问了，那么是不是服务的量级很小就不用进行流量控制限流防护了呢？是不是微服务的架构比较简单就不用引入熔断保护机制了呢？
> A：其实，这与请求的量级、架构的复杂程度无关。很多时候，可能正是一个非常边缘的服务出现故障而导致整体业务受影响，造成巨大损失。我们需要具有面向失败设计的意识，在平时就做好容量规划和强弱依赖的梳理，合理地配置流控降级规则，做好事前防护，而不是在线上出现问题以后再进行补救。

在流量控制、降级与容错场景下，我们有多种方式来描述我们的治理方案。其中 [OpenSergo](https://opensergo.io/zh-cn) 是一套开放、通用的、面向分布式服务架构、覆盖全链路异构化生态的服务治理标准，基于业界服务治理场景与实践形成服务治理通用标准。OpenSergo 的最大特点就是以统一的一套配置/DSL/协议定义服务治理规则，面向多语言异构化架构，做到全链路生态覆盖。

我们来看一下 OpenSergo 是如何定义流控降级与容错的标准，以及支撑这些标准的实现有哪些，能帮助我们解决哪些问题。

## OpenSergo 流控降级与容错 v1alpha1 标准

在 OpenSergo 中，我们结合 Sentinel 等框架的场景实践对流控降级与容错场景的实现抽象出标准的 CRD。OpenSergo 的容错治理规则 ([`FaultToleranceRule`](https://github.com/opensergo/opensergo-specification/blob/main/specification/zh-Hans/fault-tolerance.md)) 由以下三部分组成：

- Target: 针对什么样的请求
- Strategy: 容错或控制策略，如流控、熔断、并发控制、自适应过载保护、离群实例摘除等
- FallbackAction: 触发后的 fallback 行为，如返回某个错误或状态码

![image](https://github.com/opensergo/opensergo-specification/raw/main/specification/zh-Hans/images/fault-tolerance-rule-overview.png)

那么针对常用的流控降级场景，OpenSergo 具体的标准定义是什么样的，它是如何解决我们的问题的？我们来看一下。

首先提到的，只要微服务框架适配了 OpenSergo，即可通过统一 CRD 的方式来进行流控降级等治理。无论是 Java 还是 Go 还是 Mesh 服务，无论是 HTTP 请求还是 RPC 调用，还是数据库 SQL 访问，我们都可以用这统一的容错治理规则 CRD 来给微服务架构中的每一环配置容错治理，来保障我们服务链路的稳定性。让我们来详细看看 OpenSergo 在各个具体场景下的一个配置。

### 流量控制示例

以下示例定义了一个集群流控的策略，集群总体维度每秒不超过 180个请求。示例 CR YAML:

```yaml
apiVersion: fault-tolerance.opensergo.io/v1alpha1
kind: RateLimitStrategy
metadata:
  name: rate-limit-foo
spec:
  metricType: RequestAmount
  limitMode: Global
  threshold: 180
  statDuration: "1s"
```

这样一个简单的 CR 就能给我们的系统配置上一个流量控制的能力，流控能力相当于应用的一个安全气囊，超出系统服务能力以外的请求将被拒绝，具体逻辑可由我们自定义（如返回指定内容或跳转页面）。

![flow-monitor](https://user-images.githubusercontent.com/9434884/177271215-09df1452-30a8-4630-8bc9-599bc0b86393.png)

## 熔断保护示例

以下示例定义了一个慢调用比例熔断策略，示例 CR YAML:

```yaml
apiVersion: fault-tolerance.opensergo.io/v1alpha1
kind: CircuitBreakerStrategy
metadata:
  name: circuit-breaker-slow-foo
spec:
  strategy: SlowRequestRatio
  triggerRatio: '60%'
  statDuration: '30s'
  recoveryTimeout: '5s'
  minRequestAmount: 5
  slowConditions:
    maxAllowedRt: '500ms'
```

这个 CR 的语意就是：在 30s 内请求超过 500ms 的比例达到 60% 时，且请求数达到5个，则会自动触发熔断，熔断恢复时长为 5s。

![cb](https://user-images.githubusercontent.com/9434884/62473394-3bc10a00-b7d3-11e9-8998-68d9cf3e01f2.png)

想象一下，在业务高峰期。当某些下游的服务提供者遇到性能瓶颈，甚至影响业务。我们对部分非关键服务消费者配置一个这样的规则，当一段时间内的慢调用比例或错误比例达到一定条件时自动触发熔断，后续一段时间服务调用直接返回 Mock 的结果，这样既可以保障调用端不被不稳定服务拖垮，又可以给不稳定下游服务一些“喘息”的时间，同时可以保障整个业务链路的正常运转。

## 流控降级与容错标准实现

### Sentinel

[Sentinel](https://github.com/alibaba/Sentinel) 是面向分布式、多语言异构化服务架构的流量治理组件，主要以流量为切入点，从流量控制、流量整形、熔断降级、系统自适应过载保护、热点流量防护等多个维度来帮助开发者保障微服务的稳定性。Sentinel 承接了阿里巴巴近 10 年的双十一大促流量的核心场景，例如秒杀、冷启动、消息削峰填谷、集群流量控制、实时熔断下游不可用服务等，是保障微服务高可用的利器，原生支持 Java/Go/C++/Rust 等多种语言，并且提供 Istio/Envoy/MOSN/Dapr 流控支持来为 Service Mesh 提供高可用防护的能力。

![sentinel-and-opensergo](https://user-images.githubusercontent.com/9434884/177269576-eb915bc7-2939-44fa-91c5-0ec0d209c877.png)

在今年6月份的阿里云中间件开发者峰会中，社区宣布了 Sentinel 2.0 流量治理的全面升级。Sentinel 2.0 将原生支持 OpenSergo 流量治理相关 CRD 配置，作为流量治理的标准实现，结合 Sentinel 提供的各框架的适配模块，让 Dubbo, Spring Cloud Alibaba, gRPC 等20+框架能够无缝接入到 OpenSergo 生态中，用统一的 CRD 来配置流量路由、流控降级、服务容错等治理规则。

### 阿里云微服务解决方案

在阿里云上提供了一款[完全遵循 OpenSergo 微服务标准的企业级产品 MSE](https://www.aliyun.com/product/aliware/mse?sentinel-website.blog.0.0.0)，[MSE 服务治理企业版中的流量治理能力](https://help.aliyun.com/document_detail/429556.html?sentinel-website.blog.0.0.0)我们可以理解为是一个企业版的 Sentinel。下面我们基于 MSE 来演示一下，如何通过流量控制与熔断降级来保护我们的微服务，可以从容地面对不确定性的流量以及一系列不稳定的场景。

#### 配置流控规则

我们可以在监控详情页面查看每个接口实时的监控情况。

![image](https://user-images.githubusercontent.com/9434884/177269802-e52624c5-3c4f-4337-9a5d-9a067c8271ac.png)

我们可以点击接口概览右上角的“新增防护规则”按钮，添加一条流控规则：

![image](https://user-images.githubusercontent.com/9434884/177269852-ac56df55-436a-4703-ac50-6bce02b05057.png)

我们可以配置最简单的 [QPS 模式的流控规则](https://help.aliyun.com/document_detail/421958.html?sentinel-website.blog.0.0.0)，比如上面的例子即限制该接口每秒单机调用量不超过 80 次。

#### 监控查看流控效果

配置规则后，稍等片刻即可在监控页面看到限流效果：

![image](https://user-images.githubusercontent.com/9434884/177269999-08e21922-f4f6-47cc-98e5-680f3dd8c651.png)

被拒绝的流量也会返回错误信息。MSE 自带的框架埋点都有默认的流控处理逻辑，如 Web 接口被限流后返回 429 Too Many Requests，DAO 层被限流后抛出异常等。若用户希望更灵活地定制各层的流控处理逻辑，可以通过 SDK 方式接入并[配置自定义的流控处理逻辑](https://help.aliyun.com/document_detail/421992.html)。

## 总结

流控降级与容错是我们设计稳定的微服务系统时不得不考虑的场景，如果我们设计每一套系统都要花许多心思来设计系统的流控降级与容错能力，这将会成为让我们每一个开发者都头疼的问题。那么我们接触与设计了那么多系统的流控降级，有没什么通用的场景、最佳实践、设计标准与规范乃至参考实现可以沉淀的？本文从场景出发简单介绍了 OpenSergo 的流量控制与熔断保护标准，同时也介绍了 Sentinel 流量防护的背景和手段，最后通过示例来介绍如何利用 MSE 服务治理的流量防护能力来为您的应用保驾护航。

OpenSergo 标准目前还在早期演进阶段，还需要不断地与社区一起完善共建。如果您也对流控降级与容错的场景有诉求，对微服务治理的标准建设有兴趣，欢迎一起参与 OpenSergo 标准建设及 Sentinel 流量治理标准实现共建。我们会通过公开、透明、民主的方式来制定标准、推动实施。在社区也通过 GitHub issue、Gitter、邮件列表、社区双周会等机制，确保通过社区协作的方式来共建标准与实现。

相关文章：[OpenSergo 即将发布 v1alpha1，丰富全链路异构架构的服务治理能力](https://opensergo.io/zh-cn/blog/opensergo-v1alpha1-is-coming/)



