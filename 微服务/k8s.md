

















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