## service mesh

### 演进

```mermaid
graph LR
	A[代码耦合]-->B[公共库]-->C[代理]-->D[边车模式 sidercar]-->E[service mesh]
```



### 定义

> service mesh是负责服务间通信的基础设施
>
> 以sidecar的部署方式透明的为服务提供**流量控制**的能力

- 本质：基础设施层
- 功能：请求的分发
- 产品形态：网络代理（sidecar）
- 特点：应用透明
- 管理和控制服务间的通信
  - 服务注册、发现
  - 路由、流量转移
  - 弹性能力（熔断、超时、重试）
  - 安全
  - 可观察性



### 功能

- 流量控制
  - 路由
    - 蓝绿部署
    - 灰度发布
    - A/B测试
  - 流量转移
  - 超时重试
  - 熔断
  - 故障注入
  - 流量镜像
- 策略
  - 流量限制
  - 黑白名单
- 网络安全
  - 授权及身份认证
- 可观察性
  - 指标收集和展示
  - 日志收集
  - 分布式追踪



### vs `kubernetes`

- kubernetes
  - 解决容器编排与调度问题
  - 本质上是管理应用生命周期（调度器）
  - 给予service mesh支持和帮助
- service mesh
  - 解决服务间网络通信问题
  - 本质上是管理服务通信（代理）
  - 是对kubernetes网络功能方面的扩展和延伸



### vs `API 网关`

功能有重叠，但角色不同。service mesh在应用内，API网关在应用之上（边界）



## Istio

Istio，Kubernetes的好帮手

从场景来看，Kubernetes已经提供了非常强大的应用负载的部署、升级、扩容等运行管理能力。Kubernetes中的Service机制也已经可以做服务注册、服务发现和负载均衡，支持通过服务名访问到服务实例。

从微服务的工具集观点来看，Kubernetes本身是支持微服务的架构，在Pod中部署微服务很合适，也已经解决了微服务的互访互通问题，但对服务间访问的管理如服务的熔断、限流、动态路由、调用链追踪等都不在Kubernetes的能力范围内。那么，如何提供一套从底层的负载部署运行到上层的服务访问治理端到端的解决方案？

目前，最完美的答案就是在Kubernetes上叠加Istio这个好帮手。

![istio](https://s1.ax1x.com/2020/10/27/BQ3QLF.md.png)

Kubernetes的Service基于每个节点的Kube-proxy从Kube-apiserver上获取Service和Endpoint的信息，并将对Service的请求经过负载均衡转发到对应的 Endpoint 上。但Kubernetes只提供了4层负载均衡能力，无法基于应用层的信息进行负载均衡，更不会提供应用层的流量管理，在服务运行管理上也只提供了基本的探针机制，并不提供服务访问指标和调用链追踪这种应用的服务运行诊断能力。

Istio复用了Kubernetes Service的定义，在实现上进行了更细粒度的控制。Istio的服务发现就是从Kube-apiserver中获取Service和Endpoint，然后将其转换成Istio服务模型的Service和ServiceInstance，但是其数据面组件不再是Kube-proxy，而是在每个Pod里部署的Sidecar，也可以将其看作每个服务实例的Proxy。这样，Proxy的粒度就更细了，和服务实例的联系也更紧密了，可以做更多更细粒度的服务治理通过拦截Pod的Inbound流量和Outbound流量，并在Sidecar上解析各种应用层协议，Istio可以提供真正的应用层治理、监控和安全等能力。



### 流量控制

#### 主要功能

- 路由、流量转移
  - 蓝绿、灰度、AB测试
- 弹性能力
  - 超时重试、熔断
- 测试能力
  - 故障注入
  - 流量镜像

![流量控制](https://s1.ax1x.com/2020/10/13/0fJIld.png)



#### 动态路由

动态路由的设置一般是通过虚拟服务和目标规则这两个动态路由

![动态路由](https://s1.ax1x.com/2020/10/13/0hAEIx.png)

### 虚拟服务（Virtual Service）

> 路由规则的集合，描述满足条件的请求去哪里

- 将流量路由到给定目标地址
- 请求地址与真实的工作负载解耦
- 包含一组路由规则 
- 通常和目标规则（Destination Rule）成对出现
- 丰富的路由匹配规则
- 将 Kubernetes 服务连接到 Istio Gateway。它还可以执行更多操作，例如定义一组流量路由规则，以便在主机被寻址时应用

### 目标规则（Destination Rule）

  - 定义虚拟服务路由目标地址的真实地址，即子集（subset）
  - 描述到达目标的请求怎么处理
  - 设置负载均衡的方式
    - 随机
    
    - 权重
    
    - 最少请求数
    
### 网关（Gateway）

  > Service mesh最重要的一个核心功能就是流量控制，除了网格内部，也就是服务间的流量控制以外，它希望对进入网格以及出网格的流量也进行一个整体的统一控制，因此它需要给自己的网格边界上也设置这样的网关产品

![gateway](https://s1.ax1x.com/2020/10/13/0hZmWV.md.png)

  - 把内部服务暴露给外部访问
  - 管理进出网格的流量
  - 处在网格边界的负载均衡器
  - 接受外部请求，转发给网格内的服务
  - 配置对外端口、协议与内部服务的映射关系
  - 应用
    - 暴露网格内的服务给外界访问
    - 访问安全（HTTPS、mTLS等）
    - 统一应用入口，API聚合

### 服务入口（Service Entry）

  > 需要管理外部服务，比如需要对访问外部服务的请求做一些流量控制，扩展mesh

![服务入口](https://s1.ax1x.com/2020/10/13/0hZ8oR.md.png)

  - 把外部服务注册到网格中
  - 功能
    - 为外部目标转发请求
    - 添加超时重试等策略
    - 扩展网格

### 边车（Sidecar）

  - 调整Envoy代理接管的端口和协议
  - 限制Envoy代理可访问的服务



### 外部服务

- 配置global.outboundTrafficPolicy.mode = ALLOW_ANY
- 使用服务入口（Service Entry）
- 配置sidecar让流量绕过代理
- 配置Egress网关
  - 定义了网格的出口点，允许将监控、路由等功能应用于离开网格的流量



## 可观察性

梳理服务的交互关系，了解应用的行为与状态。Istio从指标日志追踪三个方面提供了与现有流行工具的集成，可以全面监控网格和应用

![总结](https://s1.ax1x.com/2020/10/13/0hCFld.md.png)

#### kiali（望远镜）

- Istio的可观察性控制台
- 通过服务拓扑帮助理解服务网格的借口
- 提供网格的健康状态师徒
- 具有服务网格配置功能

#### Prometheus

记录和查看Istio网格的指标。把瞬时的系统信息组合起来，最终形成一个较为连续的时间序列

- 采集多维指标来进行不同维度数据分析
- 预警警报

![架构图](https://prometheus.io/assets/architecture.png)



#### Grafana

指标分析平台

#### Jeager（猎人）

端对端分布式追踪系统，针对复杂的分布式系统，对业务链路进行监控和问题排查

![jeager架构](https://s1.ax1x.com/2020/10/13/0h9HW4.md.png)



## 安全

### 机制

- 透明的安全层
- CA：秘钥和证书管理
- API Server：认证、授权策略分发
- Envoy：服务间安全通信（认证、加密）

### 功能

- 认证（authentication）：you are who you say you are（你是你说的那个人吗）
  - 身份认证
  - 对等认证
- 授权（authorization）：you can do what you want to do（你能做你想做的事儿吗）
  - 网格、命名空间、服务级别
  - form、to、when
- 验证（validation）：input is correct（输入是正确的吗）



## 云原生

### 理念

- 应用只关注业务（**透明**）
- 中间件（类库、框架）**下沉**，成为云的一部分
- 云提供基础设施和各种能力（非业务需求）
- 解耦业务和非业务功能（**分离**）

