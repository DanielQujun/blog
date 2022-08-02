---
layout:     post
title:      "Kubernetes CNI 网络插件系列"
subtitle:   "记录一些CNI插件"
date:       2022-02-20
author:     "QuJun"
URL: "/2022/02/20/kubernetes-cni-series/"
image:      "https://qiniu.bobozhu.cn/blog/kubernetes-cni.jpg"
tags:
    - CNI
    - kubernetes

categories: [ Tech ]
---

> “聊聊Kubernetes的CNI”


## 关于这个系列
此系列的博客，主要是用来分析Kubernetes环境下，各种开源组件的组网实现方案及其IPAM设计。由于flannel、calico甚至cilium都已经非常流行，网上的文章也比较多，此系列更关注的是国内比较有特点的网络组件，具备一些业务特性，如网络池划分，固定IP等；

此系列既用作于自己的笔记加深对网络的理解，也希望能对其他的云原生同学们有所启发。

既然谈到的云原生的网络组件，首先我们要考虑下面的**约法三章**和**四大目标**【1】
### 约法三章
* 第一条：任意两个 pod 之间其实是可以直接通信的，无需经过显式地使用 NAT 来接收数据和地址的转换；
* 第二条：node 与 pod 之间是可以直接通信的，无需使用明显的地址转换；
* 第三条：pod 看到自己的 IP 跟别人看见它所用的IP是一样的，中间不能经过转换。

### 四大目标

四大目标其实是在设计一个 K8s 的系统为外部世界提供服务的时候，从网络的角度要想清楚，外部世界如何一步一步连接到容器内部的应用？

* 外部世界和 service 之间是怎么通信的？就是有一个互联网或者是公司外部的一个用户，怎么用到 service？service 特指 K8s 里面的服务概念。
* service 如何与它后端的 pod 通讯？
* pod 和 pod 之间调用是怎么做到通信的？
* 最后就是 pod 内部容器与容器之间的通信？

根据上述的要求设计组网通信方案后，我们考虑其业务特性，如IPAM管理

### IPAM
IPAM(IP Address Management)本来在k8s的最初设想中应该是不重要的一部分，根据前面的**约法三章**和**四大目标**，我们只需要保证pod与pod，pod与节点之间能相互连通就行，但是在真实的生产环境下，特别是商业化的云平台，需要支持不同的物理网络，可能存在多个网络并存的情况，并且需要做一些租户的网络划分或者隔离，从运维的角度上来说，网络地址的规划也是有必要的。

通用的网络管理流程可以分为以下几个步骤：

1. 创建网络类型: 如我们的网络插件支持vlan、vxlan、IPIP、BGP等多中类型；
2. 创建网络池： 从类型里划分出子网；
3. 分配IP： 应用启动时分配以及配置IP；

其中分配IP可以分为三种情况：
1. 预分配IP：即在我们创建应用前，就给这个应用提前占用一些IP，此时又分为两种情况：
  * A. 用户选定某个网络里一些IP给应用来分配
  * B. 用户指定某个网络里随机预占用一些IP给应用
> A、B两个操作都是应用在给k8s提交创建应用资源前给IPAM发送的请求。
2. 随机分配IP：指定了某个网络，等到POD创建，kubelet真正请求CNI的时候再分配

第二点是原生k8s的IP管理方案，实现上比较简单，和第一点本来应该没什么交叉的，但是由于可能存在业务逻辑，比如切换网络，那此时会涉及资源上的释放，那么还是需要有一个reconcile的过程。

而对于第一点，可能某些同学会觉得没有必要，因为K8S场景下应用应该是动态扩缩容的，此时一个预分配的逻辑引入，那么反而限制了应用的这个能力。在我看来也是如此，它更多的作用应该还是存在于有状态应用，需要有确定IP来访问，应对的是固定IP场景。

## 分析的内容
1. 组件功能特性分析介绍；
2. 底层网络实现方案，如underlay、overlay、host route,不同的组件即便都用的vxlan，
   在通信实现上可能都有所区别；
2. IPAM设计思路，以及其CNI接口实现思路，设计源码分析；

## 分析的组件
1. 阿里ACK: [HybridNet](https://bobozhu.cn/2022/03/05/hybridnet-introduce/)
2. 灵雀云: kube-ovn，灵雀云自己写了一些源码分析，先跳过。[kube-ovn](https://mp.weixin.qq.com/mp/appmsgalbum?action=getalbum&__biz=MzkxNzI2NjAxMg==&scene=1&album_id=2393354058289184770&count=3#wechat_redirect)
3. 阿里云: Terway
4. vmware: antrea

> 优先分析以上组件是因为其具备较多的业务特性，符合一个云平台网络管理配置的需求，因此暂不考虑flannel，calico等

本系列不会深入分析CNI接口规范，IPIP、BGP、Vxlan等网络技术，因为本人目前主要是在设计网络方案，
所以更关注其设计和实现思路，后续再深入其底层网络实现技术。

## 参考
1. 阿里公开课网络概念: https://edu.aliyun.com/lesson_1651_18361?spm=5176.10731542.0.0.5ff17abdKHWXyy#_18361
2. https://thenewstack.io/hackers-guide-kubernetes-networking/
