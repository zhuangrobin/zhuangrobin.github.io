---
title: k8s-start
date: 2020-04-03 21:39:55
tags: k8s 
---
k8s 作为一个编排系统容器编排系统，其被广泛的运用于生产部署服务。极大地简化了开发人员将其服务分发到各个生产服务器的步骤。
其依赖的容器技术使得其所分发的每个单元之间拥有较好的隔离性。
# 故事的开始  
> `kubeadm` 是一个快捷构建 k8s 集群的工具。

> `kubectl` 是 k8s 用于与集群进行交互的命令行工具

搭建一个 k8s 集群只需要以下几行命令，然后便可向集群提交配置文件运行服务
```shell
# node 1
kubeadm init --pod-network-cidr=192.168.0.0/16
# node 2
kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
# client
kubectl apply -f https://docs.projectcalico.org/v3.11/manifests/calico.yaml
```
# 搭建 kubernete 集群所需要的基本组件
k8s 集群中每台机器称作一个`node`，`node`有分为`crontrol-plane`和`worker`两种，`crontrol-plane`负责将用户的请求分析并分发到集群中，`worker`则负责运行实际任务。

部署`crontrol-plane`节点需要以下几个核心组件:

* `apiserver` 负责提供集群的 api 服务，是用户与 k8s 集群交互的入口，同时也是集群中的服务观察集群状态的入口。k8s 中的 api 都被设计为声明式的，即用户只需要关心集群的最终状态是怎样的，具体集群是怎样到达这种状态的由集群负责。而用户描述集群状态则需要通过一系列的资源配置表述出来。

* `controller-manager`则是管理集群中的`controller`实例，controller 被用于管理集群各种资源的状态，不同的资源会由不同的 controller 管理。因此`controller-manager`可以理解为管理整个集群状态的组件

* `scheduler` 则用于调度集群中的资源

* `etcd` 则用于存储集群中的各类信息

`worker`节点所需的核心组件则有以下几个：

* `kubelet`作为一个 agent 负责管理当前`node`与集群相关的一切事物

* `kube-proxy`管理当前节点关于集群网络的规则

# kubeadm 为你简化的流程

## kubeadm init  
`kubeadm init`命令将当前机器初始化为`crontrol-plane`节点，而初始化`control-plane`节点的步骤大致有以下几步：

* 检查宿主机环境，如 swap 是否开启
* 生成自签名 TLS 证书
* 生成 `kubelet` 相关的 kubeconfig，`static pod`配置文件

> k8s 集群中可调度的最小单元被称作`pod`，而`static pod`则是不被集群系统管理只被当前节点管理的单元。

* 启动`kubelet`，由`kubelet`启动相关的`static pod`
* 生成`worker`节点加入集群`tls bootstrapping`所需的token、`configmap`、`csr`的 `auto-approval`

> `configmap`是 k8s 集群中用于存储配置的对象

> `tls bootstrapping`是新加入的节点与 control-plane 节点相互建立信任及通信的过程。

> `csr`(certificate signing request)是 k8s 集群中用于 tls 认证的资源，当集群中的 pod 需要访问 apiserver 时就需要发送一个 csr ，然后等待管理员用`kubectl certificate approve`批准该请求， pod 才可以正常访问集群的 apiserver。 kubeadm 中将该批准流程自动化为所有的请求均批准。

* 启动`kube-proxy` `daemonset` 以及 `kube-dns` `deployment`

> `daemonset`是 k8s 中的一种资源，其用于保证集群中的每个 node 都有对应 pod 的副本，一般用于提供每个节点需要的基础服务或者守护进程。而 `deployment` 是另外一种资源，其用于保证集群中有指定数量的 pod 的副本正在运行。

> `kube-dns`是 k8s 提供用于集群内部域名解析的服务。

## kubectl join

`kubeadm join` 命令将机器初始化为一个`worker`节点，初始化`worker`节点分为以下几个步骤：

* 检查宿主机环境
* 从 control-plane 下载集群相关信息
* 启动 kubelet，进行`tls bootstrapping`过程
* 当前节点加入到集群中之后，kube-proxy daemonset 便会在当前节点启动一个实例

> `tls bootstrapping`具体步骤为：先使用命令行传入的 token 访问 apiserver，然后提交 csr，csr 被 control-plane auto approve，后续当前节点便可自由连接 apiserver。

## kubectl apply -f https://docs.projectcalico.org/v3.11/manifests/calico.yaml  

在执行完上述两个步骤之后，worker node 虽然加入了集群，但会处于 not ready 的状态。因为 k8s 集群中，每个 pod 都会有唯一的 ip 地址，该地址与物理网络是完全隔离的。因此还需要一个网络插件`cni`来将集群中的网络与物理网络关联起来，而`calico`就是这样一个插件。

上面的 yaml 文件会创建一系列 calico 相关的资源，然后启动 daemonset 在每个节点上的 node 部署一个 agent。当新建一个 pod 时，calico 就会将对应 ip 的路由规则写到每个 node 上，以此实现 pod 间的网络互通。
