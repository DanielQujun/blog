---
layout:     post
title:      "阿里ACK网络插件hybridnet"
subtitle:   "hybridnet介绍"
date:       2022-03-05
author:     "QuJun"
URL: "/2022/03/05/hybridnet-introduce/"
image:      "https://qiniu.bobozhu.cn/xihuLake.jpeg"
tags:
    - cni
    - ack
    - hybridnet
    - kubernetes

categories: [ Tech ]
---
hybridnet 目前是阿里容器云平台ACK的网络方案。它宣传点是可以在物理机和虚拟机的异构环境(Hyebrid)之上，构建一层underlay + overlay的统一网络平面。

本文于2022-04-03基于d8b0db494 commitID分析
## 组件架构

![https://qiniu.bobozhu.cn/blog/hybrid-architect.png](https://qiniu.bobozhu.cn/blog/hybrid-architect.png)

1. Hybridnet-webhook: 用于对其自身的CR和pod做validate和mutation
2. Hybridnet-manager: 用于实现Ipam,监听Pod,network,node,ipinstance等对象
3. Hybridnet: CNI-plugin二进制插件，kubelet设置pod网络时调用，目前的功能只是调用hybridnet-daemon生成IP信息；
4. hybridnet-daemon: daemonSet应用，运行在每个节点上，用于维护bgp、vxlan信息，并监听一个本地unix socket，用于给cni插件调用，为pod配置网络”`configureNic`"

## 网络资源对象(CR)

网络CR其实比较简单，从创建时序来看：

           Network→Subnet→ipinstance

### Network资源

network资源为集群安装后网络初始化的第一步，主要用于定义网络的类型，即underlay、overlay，其中underlay支持vlan和BGP模式，overlay目前支持vxlan模式。通过nodeSelector关联节点，节点必须要关联一个网络，否则会打上taints，不能调度。overlay的不设置是默认是关联所有节点，而underlay则必须要填，因为underlay需要初始化一些设备？

```go
// NetworkSpec defines the desired state of Network
type NetworkSpec struct {
   // +kubebuilder:validation:Optional
   // 节点关联网络
   NodeSelector map[string]string `json:"nodeSelector,omitempty"`
   // +kubebuilder:validation:Optional
   NetID *int32 `json:"netID"`
   // +kubebuilder:validation:Optional
   // +kubebuilder:validation:Type=string
   // Deprecated, will be removed in v0.5.0
   SwitchID string `json:"switchID"`
   // +kubebuilder:validation:Optional
   // +kubebuilder:validation:Type=string
   Type NetworkType `json:"type,omitempty"`
   // +kubebuilder:validation:Optional
   // +kubebuilder:validation:Type=string
   Mode NetworkMode `json:"mode,omitempty"`
   // +kubebuilder:validation:Optional
   Config *NetworkConfig `json:"config,omitempty"`
}

type NetworkConfig struct {
   // +kubebuilder:validation:Optional
   BGPPeers []BGPPeer `json:"bgpPeers,omitempty"`
}

type BGPPeer struct {
   // +kubebuilder:validation:Required
   ASN int32 `json:"asn"`
   // +kubebuilder:validation:Required
   Address string `json:"address"`
   // +kubebuilder:validation:Optional
   GracefulRestartSeconds int32 `json:"gracefulRestartSeconds,omitempty"`
   // +kubebuilder:validation:Optional
   Password string `json:"password,omitempty"`
}
```

### Subnet资源

即某个Network网络下的子网，通过network字段关联上面的network，以及还有NetID(主要用来避免同名network？)

```go
// SubnetSpec defines the desired state of Subnet
type SubnetSpec struct {
   // +kubebuilder:validation:Required
   Range AddressRange `json:"range"`
   // +kubebuilder:validation:Optional
   NetID *int32 `json:"netID"`
   // +kubebuilder:validation:Required
   Network string `json:"network"`
   // +kubebuilder:validation:Optional
   Config *SubnetConfig `json:"config"`
}

type AddressRange struct {
   // +kubebuilder:validation:Required
   Version IPVersion `json:"version"`
   // +kubebuilder:validation:Optional
   Start string `json:"start,omitempty"`
   // +kubebuilder:validation:Optional
   End string `json:"end,omitempty"`
   // +kubebuilder:validation:Required
   CIDR string `json:"cidr"`
   // +kubebuilder:validation:Optional
   Gateway string `json:"gateway"`
   // +kubebuilder:validation:Optional
   ReservedIPs []string `json:"reservedIPs,omitempty"`
   // +kubebuilder:validation:Optional
   ExcludeIPs []string `json:"excludeIPs,omitempty"`
}

type SubnetConfig struct {
   // +kubebuilder:validation:Optional
   GatewayType string `json:"gatewayType"`
   // +kubebuilder:validation:Optional
   GatewayNode string `json:"gatewayNode"`
   // +kubebuilder:validation:Optional
   AutoNatOutgoing *bool `json:"autoNatOutgoing"`
   // +kubebuilder:validation:Optional
   Private *bool `json:"private"`
   // +kubebuilder:validation:Optional
   AllowSubnets []string `json:"allowSubnets"`
}
```

> network和subnet分离的设计策略，猜想主要是因为混合网络，每个子网应该都是对等的，里面的IP主要用来ipam分配，而不同的网络则底层有不同的网络通信方案，需要创建对应的设备和网络策略，这样具体的底层网络和上层ipam分配就解耦了。

### IPInstance资源

每一个Pod创建后分配的IP都会创建相应IPInstance资源。官方的文档主要说是为了观测性，但是从ipam分配IP的角度，利用k8s资源创建时的唯一性，也可以考虑用来避免IP分配冲突。

```go
type IPInstanceSpec struct {
   // +kubebuilder:validation:Required
   Network string `json:"network"`
   // +kubebuilder:validation:Required
   Subnet string `json:"subnet"`
   // +kubebuilder:validation:Required
   Address Address `json:"address"`
}
type Address struct {
   // +kubebuilder:validation:Required
   Version IPVersion `json:"version"`
   // +kubebuilder:validation:Required
   IP string `json:"ip"`
   // +kubebuilder:validation:Optional
   Gateway string `json:"gateway,omitempty"`
   // +kubebuilder:validation:Required
   NetID *int32 `json:"netID"`
   // +kubebuilder:validation:Required
   MAC string `json:"mac"`
}
// IPInstanceStatus defines the observed state of IPInstance
type IPInstanceStatus struct {
   // +kubebuilder:validation:Optional
   NodeName string `json:"nodeName"`
   // +kubebuilder:validation:Optional
   Phase IPPhase `json:"phase"`
   // +kubebuilder:validation:Optional
   PodName string `json:"podName"`
   // +kubebuilder:validation:Optional
   PodNamespace string `json:"podNamespace"`
   // +kubebuilder:validation:Optional
   SandboxID string `json:"sandboxID"`
}

```

## 底层通信方案
### vxlan网络
![https://qiniu.bobozhu.cn/blog/%E9%98%BF%E9%87%8Chybridnet-vxlan.png](https://qiniu.bobozhu.cn/blog/%E9%98%BF%E9%87%8Chybridnet-vxlan.png)
简单来说的，就是通过arp_proxy和arp表加fdb来控制路由和转发目的地。本次主要测试的vxlan，但是vlan网络应该也差不多。
1. 容器中设置默认路由“10.64.0.1”，这个IP实际并不存在，利用的arp_proxy将网络包转发到了外面的veth设备上，这一点和calico是一样的。
2. 当数据包到了外面的宿主机的veth设备之后，即可通过宿主机的网络栈来进行IP转发。首先查找三层网络路由转发，可以看到本机IP直接转发到端口，其他的转发至vxlan设备上。
```
[root@ceph-3 ~]# ip rule show all 
0:      from all lookup local
1:      from all lookup 39999
2:      from all lookup 40000
3:      from all fwmark 0x20/0x20 lookup 40001
4:      from 100.64.0.0/16 lookup 10000
32766:  from all lookup main
32767:  from all lookup default
[root@ceph-3 ~]# ip route list table 40000
100.64.0.0/16 dev eth0.vxlan4 
throw 100.64.0.1 
[root@ceph-3 ~]# ip route list table 40001
default dev eth0.vxlan4 
[root@ceph-3 ~]# ip route list table 39999
100.64.0.6 dev h_aa7712f10226 
100.64.0.11 dev h_fcf14656d306 
100.64.0.18 dev h_5dbddc40e98e 
100.64.0.21 dev h_97eedc14ff71 
100.64.0.27 dev h_0ef07cb9e3ac 
100.64.0.31 dev h_6504fffdb68a 
100.64.0.32 dev h_aae715a553bc 
100.64.0.36 dev h_d11ad7b61bf9 
100.64.0.37 dev h_0c78cdf017c4 
100.64.0.38 dev h_15297b342990 
100.64.0.39 dev h_e90c92b218c2 
100.64.0.43 dev h_5f3024019a93 
[root@ceph-3 ~]# ip route show table 40001
default dev eth0.vxlan4 
```
3. 查找arp和fdb表里来做二层数据转发，如果目的IP是本机的pod，则直接转发至网络，如果在其他节点，则转发至eth0.vxlan4的vxlan网卡上，在看上会比flannel的vxlan方案简单一些，但是代价是条目会多很多？
```
[root@ceph-3 ~]# bridge fdb show 
01:00:5e:00:00:01 dev eth0 self permanent
33:33:00:00:00:01 dev eth0 self permanent
33:33:ff:9f:fc:e1 dev eth0 self permanent
01:80:c2:00:00:21 dev eth0 self permanent
33:33:ff:a0:2b:53 dev eth0 self permanent
33:33:00:00:00:02 dev eth0 self permanent
33:33:ff:00:00:00 dev eth0 self permanent
00:00:00:00:00:00 dev eth0.vxlan4 dst 192.168.0.49 self permanent
00:00:00:00:00:00 dev eth0.vxlan4 dst 192.168.0.210 self permanent
00:00:00:00:00:00 dev eth0.vxlan4 dst 192.168.0.41 self permanent
00:50:56:a0:2b:53 dev eth0.vxlan4 dst 192.168.0.49 self permanent
00:50:56:a0:cc:68 dev eth0.vxlan4 dst 192.168.0.41 self permanent
00:50:56:a0:4f:76 dev eth0.vxlan4 dst 192.168.0.210 self permanent
```
3. 数据包到了对端主机后，流程也是一样的。
> 这个方案和flannel的vxlan比看上去更加简单，也不像flannel一样需要给每个节点分配固定的cidr，似乎比flannel更好一些。

### vlan网络
底层设备用的ipvlan，文档中说共享了宿主的网卡的mac地址。和macvlan相比的话，在有mac地址限制的网络环境里兼容性会更好一些。
### BGP网络
简单看了眼代码，和calico的也有差异，创建时需要手动指定一个且只能一个的peer节点，估计是为了对其机房里的现有网络打通？后续再深入看。
### 多集群组网
目前代码中有支持识别多个集群的逻辑，估计可以通过vxlan或者bgp来实现。
## IPAM
### Manager核心代码
Manager组件中启动多个资源的controller，用来管理IP分配:
```
	ipamStore := networking.NewIPAMStore(mgr.GetClient())

	if err = (&networking.IPAMReconciler{
		Client:                mgr.GetClient(),
		Refresh:               ipamManager,
		ControllerConcurrency: concurrency.ControllerConcurrency(controllerConcurrency[networking.ControllerIPAM]),
	}).SetupWithManager(mgr); err != nil {
		entryLog.Error(err, "unable to inject controller", "controller", networking.ControllerIPAM)
		os.Exit(1)
	}

	if err = (&networking.IPInstanceReconciler{
		Client:                mgr.GetClient(),
		IPAMManager:           ipamManager,
		IPAMStore:             ipamStore,
		ControllerConcurrency: concurrency.ControllerConcurrency(controllerConcurrency[networking.ControllerIPInstance]),
	}).SetupWithManager(mgr); err != nil {
		entryLog.Error(err, "unable to inject controller", "controller", networking.ControllerIPInstance)
		os.Exit(1)
	}

	if err = (&networking.NodeReconciler{
		Client:                mgr.GetClient(),
		ControllerConcurrency: concurrency.ControllerConcurrency(controllerConcurrency[networking.ControllerNode]),
	}).SetupWithManager(mgr); err != nil {
		entryLog.Error(err, "unable to inject controller", "controller", networking.ControllerNode)
		os.Exit(1)
	}

	if err = (&networking.PodReconciler{
		APIReader:             mgr.GetAPIReader(),
		Client:                mgr.GetClient(),
		Recorder:              mgr.GetEventRecorderFor(networking.ControllerPod + "Controller"),
		IPAMStore:             ipamStore,
		IPAMManager:           ipamManager,
		ControllerConcurrency: concurrency.ControllerConcurrency(controllerConcurrency[networking.ControllerPod]),
	}).SetupWithManager(mgr); err != nil {
		entryLog.Error(err, "unable to inject controller", "controller", networking.ControllerPod)
		os.Exit(1)
	}

	if err = (&networking.NetworkStatusReconciler{
		Client:                mgr.GetClient(),
		IPAMManager:           ipamManager,
		Recorder:              mgr.GetEventRecorderFor(networking.ControllerNetworkStatus + "Controller"),
		ControllerConcurrency: concurrency.ControllerConcurrency(controllerConcurrency[networking.ControllerNetworkStatus]),
	}).SetupWithManager(mgr); err != nil {
		entryLog.Error(err, "unable to inject controller", "controller", networking.ControllerNetworkStatus)
		os.Exit(1)
	}

	if err = (&networking.SubnetStatusReconciler{
		Client:                mgr.GetClient(),
		IPAMManager:           ipamManager,
		Recorder:              mgr.GetEventRecorderFor(networking.ControllerSubnetStatus + "Controller"),
		ControllerConcurrency: concurrency.ControllerConcurrency(controllerConcurrency[networking.ControllerSubnetStatus]),
	}).SetupWithManager(mgr); err != nil {
		entryLog.Error(err, "unable to inject controller", "controller", networking.ControllerSubnetStatus)
		os.Exit(1)
	}

	if err = (&networking.QuotaReconciler{
		Client:                mgr.GetClient(),
		ControllerConcurrency: concurrency.ControllerConcurrency(controllerConcurrency[networking.ControllerQuota]),
	}).SetupWithManager(mgr); err != nil {
		entryLog.Error(err, "unable to inject controller", "controller", networking.ControllerQuota)
		os.Exit(1)
	}

	if feature.MultiClusterEnabled() {
		clusterCheckEvent := make(chan multicluster.ClusterCheckEvent, 5)

		uuidMutex, err := multicluster.NewUUIDMutexFromClient(mgr.GetClient())
		if err != nil {
			entryLog.Error(err, "unable to create cluster UUID mutex")
			os.Exit(1)
		}

		daemonHub := managerruntime.NewDaemonHub(signalContext)

		clusterStatusChecker, err := initClusterStatusChecker(mgr)
		if err != nil {
			entryLog.Error(err, "unable to init cluster status checker")
			os.Exit(1)
		}

		if err = (&multicluster.RemoteClusterUUIDReconciler{
			Client:                mgr.GetClient(),
			Recorder:              mgr.GetEventRecorderFor(multicluster.ControllerRemoteClusterUUID + "Controller"),
			UUIDMutex:             uuidMutex,
			ControllerConcurrency: concurrency.ControllerConcurrency(controllerConcurrency[multicluster.ControllerRemoteClusterUUID]),
		}).SetupWithManager(mgr); err != nil {
			entryLog.Error(err, "unable to inject controller", "controller", multicluster.ControllerRemoteClusterUUID)
			os.Exit(1)
		}

		if err = (&multicluster.RemoteClusterReconciler{
			Client:                mgr.GetClient(),
			Recorder:              mgr.GetEventRecorderFor(multicluster.ControllerRemoteCluster + "Controller"),
			UUIDMutex:             uuidMutex,
			DaemonHub:             daemonHub,
			LocalManager:          mgr,
			Event:                 clusterCheckEvent,
			ControllerConcurrency: concurrency.ControllerConcurrency(controllerConcurrency[multicluster.ControllerRemoteCluster]),
		}).SetupWithManager(mgr); err != nil {
			entryLog.Error(err, "unable to inject controller", "controller", multicluster.ControllerRemoteCluster)
			os.Exit(1)
		}

		if err = mgr.Add(&multicluster.RemoteClusterStatusChecker{
			Client:      mgr.GetClient(),
			Logger:      mgr.GetLogger().WithName("checker").WithName(multicluster.CheckerRemoteClusterStatus),
			CheckPeriod: 30 * time.Second,
			DaemonHub:   daemonHub,
			Checker:     clusterStatusChecker,
			Event:       clusterCheckEvent,
			Recorder:    mgr.GetEventRecorderFor(multicluster.CheckerRemoteClusterStatus + "Checker"),
		}); err != nil {
			entryLog.Error(err, "unable to inject checker", "checker", multicluster.CheckerRemoteClusterStatus)
			os.Exit(1)
		}
	}

	<-signalContext.Done()
```
1. IPAMReconciler,监听Subnet资源，刷新network。
2. 监听IPinstance事件，在删除时，将IP从ipamstore中释放，并移除ipinstance的finalizer，也就是说IP能否从IP池中释放变为可用分配状态，是通过ipinstance对象来控制的。而ipinstance对象的owner是被分配的pod，当pod删除时，k8s的GC策略将连带删除ipinstance。
3. 监听node，给节点打上underlay-network-attachment、overlay-network-attachment的标签，是否关联通过查询network资源列表里是否存在此节点。`labels.SelectorFromSet(network.Spec.NodeSelector).Matches(labels.Set(nodeLabels))`,overlay网络会直接跳过nodeSelector会关联所有节点。
4. 监听Pod事件，IP分配核心逻辑如下：
```
    1. 判断DeletionTimestamp是否为0，如果为0，判断是否为需要保留IP的statefulset pod.
    2. utils.PodIsEvicted(pod) || utils.PodIsCompleted(pod)的pod，直接decouple这个pod,即执行删除其拥有的ipinstance对象（这个地方是显示的执行删除，为了避免gc不及时？），然后将pod的Ip注解移除；
    3. 判断注解中是否已有信息，如果已经有了，则直接返回
    4. 开始分配IP,
    4.1 检查pod的注解或者网络中有没有指定网络名称，有则返回网络名，没有则根据网络类型随机从符合当前节点中的网络选一个
    4.2 statefulset的pod考虑重新分配或者复用IP的逻辑
    4.3 普通pod，读取注解或标签中的subnet，然后调用`r.IPAMManager.Allocate(networkName, subnetName, pod.Name, pod.Namespace)`分配IP,这里涉及到subnet中可用IP的管理，待后续分析AllocateNext函数；
    5.Couple，即创建pod相关联的IPinstance对象，并给pod添加Ip注解；

至此，pod的Reconcile完成，即IP的分配与释放逻辑也完成
```

### hybridnet-daemon核心代码
hybridnet-daemon涉及到将分配的IP配置至容器中，以及如何实现底层网络互联互通。
#### HTTP接口部分
在添加IP时，daemon的httphander，在接收到add请求后，由handleAdd函数进行处理
```
1. 获取Pod资源，检查其注解是否已经分配IP，最多重试10次。
2. 注解中有分配信息后，根据分配的IP，Get instance对象，并检查ipinstance对象的status中是否与当前的pod信息吻合，符合之后开始创建IP信息结构体utils.IPInfo
3. 开始配置网卡configureNic，创建veth设备对（LinkSetNsFd这个函数比较有意思），
3.1 在host端设置proxy_arp、route_localnet、proxy_delay、forwarding内核参数，在route table 3999中配置该pod的路由。
3.2 ConfigureContainerNic函数，继续在宿主机上配置设置forward，rp_filter、gc_thresh1，检查pod网络是否准备完成checkPodNetConfigReady，主要检查ip neigh、ip rule、ip routes是否设置完成,bgp网络还需检查peer节点是否连接正常。然后进入容器的namespace设置网卡名、IP、mac、MTU
4. 配置成功，更新IPInstance资源status字段
```
#### daemon的控制器部分
1. route控制器，监听当前集群和remote集群的subnet资源，生成对应的路由信息
```
	routeV4Manager, err := route.CreateRouteManager(config.LocalDirectTableNum,
		config.ToOverlaySubnetTableNum,
		config.OverlayMarkTableNum,
		netlink.FAMILY_V4,
	)
```
2. iptables控制器
3. bgp控制器
4. subnetReconciler: 给节点创建对应的网络设备，如vxlan、vlan等
5. ipInstanceReconciler：创建响应的arp信息
6. nodeReconciler: 主要同步vxlan网络设备，以及远端的vtep信息；

至此，hybridnet网络组件分析基本完成，其多集群的mesh组网没有还没细看，这一块也是目前的一个重点方向，后续再深入分析。

## 参考

1. **[与容器服务 ACK 发行版的深度对话第二弹：如何借助 hybridnet 构建混合云统一网络平面](https://mp.weixin.qq.com/s?__biz=MzUzNzYxNjAzMg==&mid=2247522467&idx=2&sn=91eeafd77ba3dea17a3da614c85a558a&chksm=fae6936ccd911a7aae3dc6ae22dd059791f6c7be94243be844b5f301dbb68b0414b292422bb7&mpshare=1&scene=1&srcid=0223nVlX0ElqYFKBBqI8EfU7&sharer_sharetime=1645630227908&sharer_shareid=b7c3f69daaaf7e03aeac9c7acdba908d&exportkey=AyQ8vAbaM2%2BpM1zNg2vvxPs%3D&acctmode=0&pass_ticket=EYtVsGbRZMzB9tOCu2RBTCOj0NCGOT73LuBLKiQRD3Ue%2BT%2BtmcR9cOuRCumAuPe0&wx_header=0#rd)**
2. **[hybridWIKI](https://github.com/alibaba/hybridnet/wiki/)**