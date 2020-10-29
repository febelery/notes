## 定义

1. kubernetes的本质是应用的生命周期管理，具体说是部署和管理（扩缩容、自动恢复、发布）
2. kubernetes为微服务提供了可扩展、高弹性的部署和管理平台
3. Service Mesh的基础是透明代理，通过sidecar proxy拦截到微服务间流量后再通过控制平面配置管理微服务的行为
4. Service Mesh将流量管理从kubernetes中解耦，Service Mesh内部的流量无需`kube-proxy`组件的支持，通过为更接近微服务应用层的抽象，管理服务间的流量、安全性和可观察性
5. Service Mesh是对kubernetes中的service更上层的抽象，它的下一步是serverless
6. `kube-proxy`拦截进出kubernetes节点（集群内部）的流量，`sidecar proxy`拦截进出pod的流量
7. 如果说 Kubernetes 管理的对象是 Pod，那么 Service Mesh 中管理的对象就是一个个 Service



## 概念

- service：将一组 Pod 暴露给外界访问的一种机制

- 容器就是云计算系统中的进程，容器镜像就是这个系统里的安装包，kubernetes就是操作系统

- Pod: Kubernetes 项目中的最小编排单位。一组共享了network namespace 和 volumne等资源的的容器。凡是调度、网络、存储，以及安全相关的属性，基本上是 Pod 级别的

  - NodeSelector: 将Pod与Node进行绑定

    ```yaml
    apiVersion: v1
    kind: Pod
    ...
    spec: 
    	nodeSelector: 
    		disktype: ssd
    ```

  - NodeName： 一旦 Pod 的这个字段被赋值，Kubernetes 项目就会被认为这个 Pod 已经经过了调度，调度的结果就是赋值的节点名字

  - HostAliases： 定义了 Pod 的 hosts 文件（比如 /etc/hosts）里的内容

  - ImagePullPolicy: 定义了镜像拉取的策略

- sidecar： 辅助容器，完成一些独立于主进程（主容器）之外的工作

- Projected Volume: 投射数据卷, Secret/ConfigMap/Downward API/ServiceAccountToken

- StatefulSet: 有状态应用

  - 拓扑状态
  - 储存状态