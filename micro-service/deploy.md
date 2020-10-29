## virtualbox 安装centos

### 网络设置

#### 网卡一

用`网络地址转换(NAT)`，其他设置不变，也不用端口转发

这个网卡的主要作用就是用来访问外网

#### 网卡二

用`仅主机（Host-Only）网络`，`界面名称`需要提前virtualbox全局菜单栏设置

`管理`-->`主机网络管理器`，这个网卡的主要作用就是用来虚拟机之间连接和和宿主机之间连接

进入虚拟机后，查看网络二的名称`ip addr`，比如是`enp0s8`，执行如下命令
```bash
$ touch /etc/sysconfig/network-scripts/ifcfg-enp0s8

TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=enp0s3
UUID=53c8eb8b-db2d-4ed7-b404-bb0289092be1
DEVICE=enp0s8
ONBOOT=yes
IPADDR=192.168.56.100 # 配置网络静态ip
NETMASK=255.255.255.0
GATEWAY=192.168.56.1
DNS1=8.8.8.8
DNS2=114.114.114.114

$ nmcli c reload
$ nmcli c up enp0s8
```


## 系统参数

### k8s.conf

```bash
$ vi /etc/sysctl.d/k8s.conf

net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
net.ipv4.tcp_tw_recycle=1 # 表示开启TCP连接中TIME-WAIT sockets的快速回收，默认为0，表示关闭。
net.ipv4.tcp_keepalive_time=600 # 超过这个时间没有数据传输，就开始发送存活探测包
net.ipv4.tcp_keepalive_intvl=15 # keepalive探测包的发送间隔
net.ipv4.tcp_keepalive_probes=3 # 如果对方不予应答，探测包的发送次数
vm.swappiness=0 # 禁止使用 swap 空间，只有当系统 OOM 时才允许使用它
vm.overcommit_memory=1 # 不检查物理内存是否够用
vm.panic_on_oom=0 # 开启 OOM
fs.inotify.max_user_instances=8192
fs.inotify.max_user_watches=1048576
fs.file-max=52706963
fs.nr_open=52706963
net.ipv6.conf.all.disable_ipv6=1
net.netfilter.nf_conntrack_max=2310720

$ syscel --system
$ syscel -p /etc/sysctl.d/k8s.conf
```

### ipvs.modules

```bash
$ vi /etc/sysconfig/modules/ipvs.modules

modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
modprobe br_netfilter

$ chmod 755 /etc/sysconfig/modules/ipvs.modules 
$ bash /etc/sysconfig/modules/ipvs.modules
$ lsmod | grep -e ip_vs -e nf_conntrack_ipv4
```


## 安装

>  [高可用集群](https://www.funtl.com/zh/service-mesh-kubernetes/%E9%AB%98%E5%8F%AF%E7%94%A8%E9%9B%86%E7%BE%A4.html)   [k8s高可用集群搭建](https://www.jianshu.com/p/3de558d8b57a)

![拓扑](https://s1.ax1x.com/2020/10/20/Bppnot.png)

- Keepalived 提供 `kube-apiserver` 对外服务的虚拟 IP（VIP）
- HAProxy 监听 Keepalived VIP
- 运行 Keepalived 和 HAProxy 的节点称为 LB（负载均衡） 节点
- Keepalived 是一主多备运行模式，故至少需要两个 LB 节点
- Keepalived 在运行过程中周期检查本机的 HAProxy 进程状态，如果检测到 HAProxy 进程异常，则触发重新选主的过程，VIP 将飘移到新选出来的主节点，从而实现 VIP 的高可用
- 所有组件（如 kubeclt、apiserver、controller-manager、scheduler 等）都通过 VIP +HAProxy 监听的 6444 端口访问 `kube-apiserver` 服务（**注意：`kube-apiserver` 默认端口为 6443，为了避免冲突我们将 HAProxy 端口设置为 6444，其它组件都是通过该端口统一请求 apiserver**）

### 节点配置


| 主机名               | IP             | 角色   |  CPU/内存 | 磁盘 |
| -------------------- | -------------- | ------ |  -------- | ---- |
| master-01 | 192.168.56.101 | Master |  2核2G    | 20G  |
| master-02 | 192.168.56.102 | Master |  2核2G    | 20G  |
| master-03 | 192.168.56.103 | Master |  2核2G    | 20G  |
| node-01   | 192.168.56.111 | Node   |  2核4G    | 20G  |
| node-02   | 192.168.56.112 | Node   |  2核4G    | 20G  |
| node-03   | 192.168.56.113 | Node   |  2核4G    | 20G  |
| VIP (virtual ip) | 192.168.56.50 | -      | -        | -    |

### HAProxy

> 替代方案LVS

```bash
$ dnf install haproxy
$ systemctl enable haproxy.service
$ vim /etc/haproxy/haproxy.cfg

...
backend app
    mode        tcp
    balance     roundrobin
    server  master01 192.168.56.101 check
    server  master02 192.168.56.102 check
    server  master03 192.168.56.103 check
...
```



### Keepalive

```bash
$ dnf install keepalive
$ systemctl enable keepalived.service
$ vim /etc/keepalived/keepalived.conf

...
vrrp_script chk_haproxy {
	# check keepalived
    script "/bin/bash -c 'if [[ $(netstat -nlp | grep 5000) ]]; then exit 0;fi'"
    interval 2
    weight 11
}

vrrp_instance VI_1 {
    state MASTER # 其他节点改为BACKUP
    interface enp0s8
    virtual_router_id 51
    priority 100
    advert_int 1
    nopreempt

    authentication {
        auth_type PASS
        auth_pass password 
    }
    virtual_ipaddress {
        192.168.56.50
    }
}

virtual_server 192.168.56.50 6443 {
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

  real_server 192.168.56.101 6443 {
    weight 1
    TCP_CHECK {
      connect_timeout 10
      nb_get_retry 3
      delay_before_retry 3
      connect_port 6443
    }
  }
  # more
}
...
```



### Dashboard

> [部署 Dashboard 和 metrics-server](https://tomoyadeng.github.io/blog/2019/08/11/k8s-dashboard-openssl/index.html)

```bash
# 查看登录token
$ kubectl -n kube-system describe secret $(kubectl get secret --all-namespaces | grep admin | awk '{print $2}')
```

```yaml
# dashboard.yaml
# 配置 Ingress Nginx 提供访问入口
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: kubernetes-dashboard-ingress
  namespace: kubernetes-dashboard
  annotations:
    kubernetes.io/ingress.class: "nginx"
    # 开启use-regex，启用path的正则匹配
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /
    # 默认为 true，启用 TLS 时，http请求会 308 重定向到https
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    # 默认为 http，开启后端服务使用 proxy_pass https://协议
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  rules:
  - host: dashboard.febelery.com
    http:
      paths:
      - path: /
        backend:
          serviceName: kubernetes-dashboard
          servicePort: 443
  tls:
  - secretName: kubernetes-dashboard-certs
    hosts:
    - dashboard.febelery.com

---

# 新建管理员 
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
```



### nfs

```shell
$ apt install nfs-kernel-server -y
$ dnf install nfs-utils -y
$ systemctl enable nfs-server.serivce
$ vim /etc/exports
	/nfs *(rw,sync,insecure,no_subtree_check,no_root_squash)
$ systemctl start nfs-server.service
$ showmount -e "your ip"
$ mount -t nfs "nfs ip":/nfs /mnt/nfs
```

### ingress

> [ingress在物理机上的nodePort和hostNetwork两种部署方式解析及比较](https://xuxinkun.github.io/2019/06/11/ingress/)
>
> [nginx ingress controller](https://kubernetes.github.io/ingress-nginx/deploy/)

#### 安装ingress-nginx

1. 导出配置

   ```bash
   $ helm inspect values ingress-nginx/ingress-nginx > ingress-nginx.yaml
   ```

2. 配置`ngress-nginx.yaml`

   不需要设置`namespace`，默认对所有namespace有效
   
   ```yaml
   controller:
     image:
     	# 镜像仓库换成aliyun代理 https://cr.console.aliyun.com/cn-hangzhou/instances/images
       repository: registry.cn-hangzhou.aliyuncs.com/k8s-gcr-io-latest/ingress-nginx-controller
       tag: "v0.40.2"
    digest: sha256:0100c173327bbb124c76ea1511dade4cec718234c23f8e7a41f27ad03f361431
   ```

   - nodePort

      nodePort的部署思路就是通过在每个节点上开辟nodePort的端口，将流量引入进来，而后通过iptables首先转发到ingress-controller容器中(图中的nginx容器)，而后由nginx根据ingress的规则进行判断，将其转发到对应的应用web容器中

      ![nodePort](https://s1.ax1x.com/2020/10/29/B8z3LV.jpg)

      ```yaml
   type: NodePort
      nodePorts:
       http: 32080
       https: 32443
       tcp:
         8080: 32808
     ```
   
      
   
   - hostNetwork
   
      hostNetwork模式不再需要创建一个nodePort的svc，而是直接在每个节点都创建一个ingress-controller的容器，而且将该容器的网络模式设为hostNetwork。也就是说每个节点物理机的80和443端口将会被ingress-controller中的nginx容器占用。当流量通过80/443端口进入时，将直接进入到nginx中。而后nginx根据ingress规则再将流量转发到对应的web应用容器中
   
      ![](https://s1.ax1x.com/2020/10/29/BGSElR.jpg)
      
      ```yaml
      dnsPolicy: ClusterFirstWithHostNet
      ## 需要注释nodeSelector，如果不注释，则需要在需要安装ingress的node节点打标签
      ## kubectl label node --all kubernetes.io/os=linux
      # nodeSelector:
      #   kubernetes.io/os: linux
      hostNetwork: true
      kind: DaemonSet
      ## 这个绑定外部IP 
      ## https://kubernetes.io/zh/docs/concepts/services-networking/service/#external-ips
      # externalIPs: [192.168.56.50]
      ```
      
      
   
3. 安装

   ```bash
   $ helm install ingress-nginx ingress-nginx/ingress-nginx -f ingress-nginx.yaml
   ```

4. 使用

   在外部通过域名配置ingress时，可以用hosts指定任意一台`nodes`的IP地址即可


### harbor

> [通过 Helm 搭建 Docker 镜像仓库 Harbor](http://www.mydlq.club/article/66/)
>
> [在 Kubernetes 在快速安装 Harbor](https://www.qikqiak.com/post/harbor-quick-install/)

#### 新建PV、PVC

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: harbor
  labels:
    name: harbor
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: harbor-pv
  labels:
    app: harbor-pv
spec:
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /media/ross/hp/virtualbox/nfs/harbor
    server: 172.16.11.84
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: harbor-pvc
  namespace: harbor
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 50Gi
  selector:
    matchLabels:
      app: harbor-pv
```

#### 安装harbor

1. 导出配置

    ```bash
    $ helm inspect values harbor/harbor > harbor.yaml
    ```

2. 配置`harbor.yaml`

    ```yaml
    expose:
      type: nodePort
      ## 当用ingress时，不需要填写域名后面的端口号，可以用443端口，这个时候的ingress controller的type为hostNetwork
      # type: ingress
      tls:
        enabled: true
        certSource: auto
        auto:
          # The common name used to generate the certificate, it's necessary when the type isn't "ingress"
          # 这个字段填写的是域名，用于docker login tls握手用
          commonName: "docker.febelery.com"
      nodePort:
        name: harbor
        ports:
          http:
            port: 80
            nodePort: 30002
          https:
            # The service port Harbor listens on when serving with HTTPS
            port: 443
            # The node port Harbor listens on when serving with HTTPS
            nodePort: 30003
          # Only needed when notary.enabled is set to true
          notary:
            port: 4443
            nodePort: 30004
    # The external URL for Harbor core service. It is used to
    # 1) populate the docker/helm commands showed on portal
    # 2) populate the token service URL returned to docker/notary client
    #
    # Format: protocol://domain[:port]. Usually:
    # 1) if "expose.type" is "ingress", the "domain" should be
    # the value of "expose.ingress.hosts.core"
    # 2) if "expose.type" is "clusterIP", the "domain" should be
    # the value of "expose.clusterIP.name"
    # 3) if "expose.type" is "nodePort", the "domain" should be
    # the IP address of k8s node
    #
    # If Harbor is deployed behind the proxy, set it as the URL of proxy
    # 这个地址很重要，和commonName对应
    externalURL: https://docker.febelery.com:30003
    ```

3. 使用`helm`安装harbor

    ```bash
    $ helm install harbor harbor/harbor -f harbor-helm.yaml -n harbor
    ```

#### 登录 Harbor 仓库

在客户端（宿主机）中配置`/etc/hosts`

```shell
192.168.56.50 docker.febelery.com
```

打开浏览器输入`https://docker.febelery.com:30003`，登录后进入 `系统管理` --> `配置管理` --> `系统设置` --> `镜像库根证书`  下载ca.crt

```shell
$ mkdir -p /etc/docker/certs.d/docker.febelery.com
$ echo "ca.crt 内容" >> /etc/docker/certs.d/docker.febelery.com/ca.crt
$ vim /etc/docker/daemon.josn
	{
        "registry-mirrors":[
            "https://docker.mirrors.ustc.edu.cn",
            "http://f1361db2.m.daocloud.io"
        ],
        "insecure-registries":[
            "https://docker.febelery.com:30003"
        ]
    }
$ sudo systemctl restart docker.service
$ docker login https://docker.febelery.com:30003 -u admin -p Harbor12345 # 登录harbor仓库
```



### 设置

#### docker

```shell
{
    "registry-mirrors":[
        "https://docker.mirrors.ustc.edu.cn",
        "http://f1361db2.m.daocloud.io"
    ],
    "insecure-registries":[
    ],
    "exec-opts":[
        "native.cgroupdriver=systemd"
    ]
}
  
```
```bash
dnf install iproute-tc && dnf provides tc first
```


#### calico

  ```shell
  $ kubeadm config print init-defaults --kubeconfig ClusterConfiguration > kubeadm.yml
  $ vim kubeadm.yml
      apiServer:
        timeoutForControlPlane: 4m0s
      apiVersion: kubeadm.k8s.io/v1beta1
      certificatesDir: /etc/kubernetes/pki
      clusterName: kubernetes
      controlPlaneEndpoint: "192.168.141.50:6444"
      controllerManager: {}
      dns:
        type: CoreDNS
      etcd:
        local:
          dataDir: /var/lib/etcd
      imageRepository: registry.aliyuncs.com/google_containers
      kind: ClusterConfiguration
      kubernetesVersion: v1.14.2
      networking:
        dnsDomain: cluster.local
        # 主要修改在这里，替换 Calico 网段为我们虚拟机不重叠的网段（这里用的是 Flannel 默认网段）
        podSubnet: "10.244.0.0/16"
        serviceSubnet: 10.96.0.0/12
      scheduler: {}
  
  $ curl https://docs.projectcalico.org/manifests/calico.yaml -O
  $ vim calico.yaml
      containers:
        - env:
        - name: DATASTORE_TYPE
        value: kubernetes
        - name: IP_AUTODETECTION_METHOD  # DaemonSet中添加该环境变量
        value: interface=enp0s8   # 指定内网网卡，经过测试并没有啥用，加入集群要手动指定
        - name: WAIT_FOR_DATASTORE
        value: "true"
  ```


## 常见问题

#### kubeadm init 在主节点不成功

可以用`ip addr`查看当前VIP是否在此节点，如果不在次节点，可停止其他节点的keepalive `systemctl stop keepalived.service`

#### 在第三个master节点无法加入集群

提示`etcd`健康检查不成功

在virtualbox虚拟机中，可能每台虚拟机的`enp0s3`的IP地址一样。

可以用`kubectl describe configmaps kubeadm-config -n kube-system`查看第二台master的advertiseAddress是否和第三个节点的IP地址一样，可以手动指定advertiseAddress

`kubeadm join xxx --apiserver-advertise-address "your local machine ip"`

#### kubeadm默认证书只有一年有效期

> [更新一个10年有效期的 Kubernetes 证书](https://www.qikqiak.com/post/update-k8s-10y-expire-certs/) 

```shell
$ kubeadm alpha certs check-expiration
```

#### `kubeadm join`使用的token过期后，如何加入集群

```bash
$ kubeadm token create
$ kubeadm token list
$ openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | \
	openssl rsa -pubin -outform der 2>/dev/null | \
	openssl dgst -sha256 -hex | sed 's/^.* //'
$ kubeadm join 192.168.56.50:6443 --token xx.xxx \
    --discovery-token-ca-cert-hash sha256:xxx
```

#### 查看`kubeadmin join`加入命令

```bash
$ kubeadm token create --print-join-command
```




