---
layout:     post
title:      "Kubernetes 自定义控制器开发流程"
subtitle:   "快速准备operator编写框架的流程"
date:       2022-02-26
author:     "QuJun"
URL: "/2022/02/26/kubernetes-operator-dev-guide/"
image:      "https://qiniu.bobozhu.cn/blog/kubernetes-cni.jpg"
tags:
    - operator-sdk
    - operator
    - code-generator
    - kubebuilder
    - kubernetes

categories: [ Tech ]
---

编写operator有两个比较流行的框架，operator-sdk和kubebuilder，使用这类框架可以快速生成完整的项目文件，我们只需要专注于CR定义和业务逻辑的编写，上手十分方便。但美中不足的是此类框架屏蔽太多底层内容，并且与其他三方交互也不太方便*(没有clientset)*,因此我们考虑结合kubernetes自身的code-generator项目一起来用。

## 使用kubebuilder生成开发框架【1】

operator-sdk可能会合并到Kubebuilder,而且operator-sdk本身也是基于kubebuilder做的包装，提供的helm脚本、OLM管理特性暂时也用不到，因此可以先学kubebuilder。

首先了解Kubernetes里的Groups、Versions、Kinds、Resources。

简单来说： 一个API Group是一组功能的集合，这个Group下面可以有多个Version。 然后一个Group和Version组合之后，下面可能有多个Kinds。每个资源Resource其实都对应了一个Kind。

这样一个*GVK(group-version-kind)*就确定了一个类型。

可能我们还会接触一个scheme的概念，它指一个具体的格式来对应我们定义的GVK，如json格式，这里可能还涉及到一些注册编码的概念，得深入apimachinary库去看，后面再研究。

```go
{
    "kind": "CronJob",
    "apiVersion": "batch.tutorial.kubebuilder.io/v1",
    ...
}
```

1. 安装kubebuilder

```go
curl -L -o kubebuilder https://go.kubebuilder.io/dl/latest/$(go env GOOS)/$(go env GOARCH)
chmod +x kubebuilder && mv kubebuilder /usr/local/bin/
```

2. 初始化项目

```go
mkdir -p ~/projects/guestbook
cd ~/projects/guestbook
# --domain 指定你的group里的域名， 
# --repo 对应你当前的项目，即go.mod里module名称，如果已经有go.mod文件，可以不需要指定此参数
kubebuilder init --domain tiduyun.com --repo git.tiduyun.com/tsi-controller
```

此时会生成main.go，Dockerfile, Makefile ,以及一些部署controller的yaml

```go
tree
.
├── config
│   ├── default
│   │   ├── kustomization.yaml
│   │   ├── manager_auth_proxy_patch.yaml
│   │   └── manager_config_patch.yaml
│   ├── manager
│   │   ├── controller_manager_config.yaml
│   │   ├── kustomization.yaml
│   │   └── manager.yaml
│   ├── prometheus
│   │   ├── kustomization.yaml
│   │   └── monitor.yaml
│   └── rbac
│       ├── auth_proxy_client_clusterrole.yaml
│       ├── auth_proxy_role_binding.yaml
│       ├── auth_proxy_role.yaml
│       ├── auth_proxy_service.yaml
│       ├── kustomization.yaml
│       ├── leader_election_role_binding.yaml
│       ├── leader_election_role.yaml
│       ├── role_binding.yaml
│       └── service_account.yaml
├── Dockerfile
├── go.mod
├── go.sum
├── hack
│   └── boilerplate.go.txt
├── main.go
├── Makefile
└── PROJECT
```

3. 初始化API group

```go
kubebuilder create api --group tsi --version v1alpha1 --kind IPPool
# 如果有多个Kind需要生成，可以指定另外的kind再执行一次。
# 执行命令后会提示：
# 是否创建资源，即api目录下types.go, deepcopy.go
Create Resource [y/n]
y
# 是否创建controller.go, 如果后面我们用code-generator生成clientset和informer的话是可以不用生成的。
Create Controller [y/n]
y
```

此时会生成controller和api目录。并且main.go中会自动注册controller

4. 编辑api/${version}目录的${kind}_types.go，添加CR所需的业务字段，然后使用下面的命令生成其对应的CRD manifest。

```go
make manifests
# 实际执行的是controller-gen命令，controller-gen也是kubebuilder下面的一个项目；
/home/danielqu/data/go/src/git.tiduyun.com/tsi-controller/bin/controller-gen rbac:roleName=manager-role crd webhook paths="./..." output:crd:artifacts:config=config/crd/bases

# 也可以简单点，只生成CRD：
bin/controller-gen crd paths="./..." output:crd:artifacts:config=config/crd/bases
```

5. 如果types中有更改，需要重新生成CRD，执行下列命令：

```go
bin/controller-gen object:headerFile="hack/boilerplate.go.txt" paths="./..."
bin/controller-gen crd paths="./..." output:crd:artifacts:config=config/crd/bases
```

至此，利用kubebuilder生成原始api和crd的流程已经完成，如果我们不打算用clientset和informer来管理对象的话，就可以直接去controller目录中写*[Reconciler](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/reconcile?tab=doc)*逻辑了。
```go
package controllers

import (
	"context"

	"k8s.io/apimachinery/pkg/runtime"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/log"

	tsiv1alpha1 "git.tiduyun.com/tsi-controller/api/v1alpha1"
)

// IPPoolReconciler reconciles a IPPool object
type IPPoolReconciler struct {
	client.Client
	Scheme *runtime.Scheme
}

//+kubebuilder:rbac:groups=tsi.tiduyun.com,resources=ippools,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=tsi.tiduyun.com,resources=ippools/status,verbs=get;update;patch
//+kubebuilder:rbac:groups=tsi.tiduyun.com,resources=ippools/finalizers,verbs=update

// Reconcile is part of the main kubernetes reconciliation loop which aims to
// move the current state of the cluster closer to the desired state.
// TODO(user): Modify the Reconcile function to compare the state specified by
// the IPPool object against the actual cluster state, and then
// perform operations to make the cluster state reflect the state specified by
// the user.
//
// For more details, check Reconcile and its Result here:
// - https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.11.0/pkg/reconcile
func (r *IPPoolReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	_ = log.FromContext(ctx)

	// TODO(user): your logic here

	return ctrl.Result{}, nil
}

```
## k8s的code-generator

kubebuilder使用controller-runtime库，屏蔽了太多细节，并且其client是动态的，只支持json解码，性能没有原生clientset好，并且也不方便第三方引用开发，所以我们还是某些场景下还是用kubernetes的code-generator项目来生成clientset和informer。

此时基于前面kubebuilder生成的代码继续使用code-generator，可以省掉一些初始化types.go和手写CRD的操作

> 主要原因就是想省掉写CRD的这些操作，接下的步骤里会直接用controller-gen加code-generator来生成deepcopy代码和CRD。

    

1. 由于kubebuilder生成的目录结构不一样，首先做一个转换

```go
# kubebuilder生成的api目录结构code-generator无法识别，需要转换目录结构
# APIS_PKG对应kubebuilder默认生成的api目录
APIS_PKG=api
# GROUP 对应kubebuild create api时指定的group
GROUP=tsi
# VERSION 对应kubebuild create api时指定的version
VERSION=v1alpha1
mkdir -p "${APIS_PKG}/${GROUP}" && cp -r "${APIS_PKG}/${VERSION}/" "${APIS_PKG}/${GROUP}"
rm -rf "${APIS_PKG}/${VERSION}/"
```

2. 在api/${GROUP}/${version} 目录下添加doc.go,注意里面的group和package名需要与项目对应。

```go
// +k8s:deepcopy-gen=package

// Package v1alpha1 is the v1alpha1 version of the API.
//+groupName=tsi.tiduyun.com
package v1alpha1
```

3. 在api/${GROUP}/${version} 目录添加`register.go`

```go
package v1alpha1
  
  import (
      "k8s.io/apimachinery/pkg/runtime/schema"
  )
  
  // SchemeGroupVersion is group version used to register these objects.
  var SchemeGroupVersion = GroupVersion
  
  // Resource takes an unqualified resource and returns a Group qualified GroupResource
  func Resource(resource string) schema.GroupResource {
      return SchemeGroupVersion.WithResource(resource).GroupResource()
  }
```

4. 修改 api/${GROUP}/${version}/{crd}_types.go文件，在每个资源上添加genclient和deepcopy的tag。

```go
// +genclient
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
//+kubebuilder:object:root=true
//+kubebuilder:subresource:status
//+kubebuilder:resource:scope=Cluster
// IPPool is the Schema for the ippools API
type IPPool struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   IPPoolSpec   `json:"spec,omitempty"`
	Status IPPoolStatus `json:"status,omitempty"`
}

//+kubebuilder:object:root=true
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// IPPoolList contains a list of IPPool
type IPPoolList struct {
```

5. 在hack目录下准备tools.go ,用于引入code-generator包

```go
// +build tools
  
  package tools
  
  import _ "k8s.io/code-generator"
```

6. 在hack下创建update-codegen.sh脚本，注意根据项目修改相应变量

```go
  `MODULE` 和 `go.mod` 保持一致

  `API_PKG=api`，和 `api` 目录保持一致

  `OUTPUT_PKG=generated/example`，与生成Resource时指定的group保持一致

  `GROUP=example`， 和生成Resource时指定的group 保持一致

  `VERSION=v1`， 和生成Resource时指定的version保持一致
  
  #!/usr/bin/env bash
  #表示有报错即退出 跟set -e含义一样
  set -o errexit
  #执行脚本的时候，如果遇到不存在的变量，Bash 默认忽略它 ,跟 set -u含义一样
  set -o nounset
  # 只要一个子命令失败，整个管道命令就失败，脚本就会终止执行 
  set -o pipefail
  
  #kubebuilder项目的MODULE
  MODULE=my.domain/example
  
  #api包
  # APIS_PKG=api, kubebuilder默认生成的api目录
  APIS_PKG=api
  
  # GROUP=tsi对应我们kubebuild create api时指定的group
  GROUP=tsi
  VERSION=v1
  GROUP_VERSION=$GROUP:$VERSION

  #client代码输出目录
  OUTPUT_PKG=pkg/generated/client
  
  SCRIPT_ROOT=$(dirname "${BASH_SOURCE[0]}")/..
  CODEGEN_PKG=${CODEGEN_PKG:-$(cd "${SCRIPT_ROOT}"; ls -d -1 ./vendor/k8s.io/code-generator 2>/dev/null || echo ../code-generator)}
  
  # generate the code with:
  # --output-base    because this script should also be able to run inside the vendor dir of
  #                  k8s.io/kubernetes. The output-base is needed for the generators to output into the vendor dir
  #                  instead of the $GOPATH directly. For normal projects this can be dropped.
  #client,informer,lister(注意: code-generator 生成的deepcopy不适配 kubebuilder 所生成的api)
  bash "${CODEGEN_PKG}"/generate-groups.sh "deepcopy,client,informer,lister" \
    ${MODULE}/${OUTPUT_PKG} ${MODULE}/${APIS_PKG} \
    ${GROUP_VERSION} \
    --go-header-file "${SCRIPT_ROOT}"/hack/boilerplate.go.txt

  # 更新CRD
  bin/controller-gen crd paths="./..." output:crd:artifacts:config=config/crd/bases

  
```

7. 拉取code-generator至vendor目录，`go mod vendor`
8. 生成clientset， `./hacker/update-codegen.sh`

## controller代码开发示例

1. 定义一个结构体

```go
import (
	"context"
	"fmt"
	"time"

	appsv1 "k8s.io/api/apps/v1"
	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/api/errors"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	utilruntime "k8s.io/apimachinery/pkg/util/runtime"
	"k8s.io/apimachinery/pkg/util/wait"
	appsinformers "k8s.io/client-go/informers/apps/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/kubernetes/scheme"
	typedcorev1 "k8s.io/client-go/kubernetes/typed/core/v1"
	appslisters "k8s.io/client-go/listers/apps/v1"
	"k8s.io/client-go/tools/cache"
	"k8s.io/client-go/tools/record"
	"k8s.io/client-go/util/workqueue"
	"k8s.io/klog/v2"

	samplev1alpha1 "k8s.io/sample-controller/pkg/apis/samplecontroller/v1alpha1"
	clientset "k8s.io/sample-controller/pkg/generated/clientset/versioned"
	samplescheme "k8s.io/sample-controller/pkg/generated/clientset/versioned/scheme"
	informers "k8s.io/sample-controller/pkg/generated/informers/externalversions/samplecontroller/v1alpha1"
	listers "k8s.io/sample-controller/pkg/generated/listers/samplecontroller/v1alpha1"
)

// Controller is the controller implementation for Foo resources
type Controller struct {
	// kubeclientset is a standard kubernetes clientset
	kubeclientset kubernetes.Interface
	// sampleclientset is a clientset for our own API group
	sampleclientset clientset.Interface

	deploymentsLister appslisters.DeploymentLister
	deploymentsSynced cache.InformerSynced
	foosLister        listers.FooLister
	foosSynced        cache.InformerSynced

	// workqueue is a rate limited work queue. This is used to queue work to be
	// processed instead of performing it as soon as a change happens. This
	// means we can ensure we only process a fixed amount of resources at a
	// time, and makes it easy to ensure we are never processing the same item
	// simultaneously in two different workers.
	workqueue workqueue.RateLimitingInterface
	// recorder is an event recorder for recording Event resources to the
	// Kubernetes API.
	recorder record.EventRecorder
}
```

2. 初始化

```go
func NewController(
	kubeclientset kubernetes.Interface,
	sampleclientset clientset.Interface,
	deploymentInformer appsinformers.DeploymentInformer,
	fooInformer informers.FooInformer) *Controller {

	// Create event broadcaster
	// Add sample-controller types to the default Kubernetes Scheme so Events can be
	// logged for sample-controller types.
	utilruntime.Must(samplescheme.AddToScheme(scheme.Scheme))
	klog.V(4).Info("Creating event broadcaster")
	eventBroadcaster := record.NewBroadcaster()
	eventBroadcaster.StartStructuredLogging(0)
	eventBroadcaster.StartRecordingToSink(&typedcorev1.EventSinkImpl{Interface: kubeclientset.CoreV1().Events("")})
	recorder := eventBroadcaster.NewRecorder(scheme.Scheme, corev1.EventSource{Component: controllerAgentName})

	controller := &Controller{
		kubeclientset:     kubeclientset,
		sampleclientset:   sampleclientset,
		deploymentsLister: deploymentInformer.Lister(),
		deploymentsSynced: deploymentInformer.Informer().HasSynced,
		foosLister:        fooInformer.Lister(),
		foosSynced:        fooInformer.Informer().HasSynced,
		workqueue:         workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), "Foos"),
		recorder:          recorder,
	}

	klog.Info("Setting up event handlers")
	// Set up an event handler for when Foo resources change
	fooInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: controller.enqueueFoo,
		UpdateFunc: func(old, new interface{}) {
			controller.enqueueFoo(new)
		},
	})
	// Set up an event handler for when Deployment resources change. This
	// handler will lookup the owner of the given Deployment, and if it is
	// owned by a Foo resource then the handler will enqueue that Foo resource for
	// processing. This way, we don't need to implement custom logic for
	// handling Deployment resources. More info on this pattern:
	// https://github.com/kubernetes/community/blob/8cafef897a22026d42f5e5bb3f104febe7e29830/contributors/devel/controllers.md
	deploymentInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: controller.handleObject,
		UpdateFunc: func(old, new interface{}) {
			newDepl := new.(*appsv1.Deployment)
			oldDepl := old.(*appsv1.Deployment)
			if newDepl.ResourceVersion == oldDepl.ResourceVersion {
				// Periodic resync will send update events for all known Deployments.
				// Two different versions of the same Deployment will always have different RVs.
				return
			}
			controller.handleObject(new)
		},
		DeleteFunc: controller.handleObject,
	})

	return controller
}
```

3. 定义结构体的run方法

```go
// Run will set up the event handlers for types we are interested in, as well
// as syncing informer caches and starting workers. It will block until stopCh
// is closed, at which point it will shutdown the workqueue and wait for
// workers to finish processing their current work items.
func (c *Controller) Run(workers int, stopCh <-chan struct{}) error {
	defer utilruntime.HandleCrash()
	defer c.workqueue.ShutDown()

	// Start the informer factories to begin populating the informer caches
	klog.Info("Starting Foo controller")

	// Wait for the caches to be synced before starting workers
	klog.Info("Waiting for informer caches to sync")
	if ok := cache.WaitForCacheSync(stopCh, c.deploymentsSynced, c.foosSynced); !ok {
		return fmt.Errorf("failed to wait for caches to sync")
	}

	klog.Info("Starting workers")
	// Launch two workers to process Foo resources
	for i := 0; i < workers; i++ {
		go wait.Until(c.runWorker, time.Second, stopCh)
	}

	klog.Info("Started workers")
	<-stopCh
	klog.Info("Shutting down workers")

	return nil
}
```

4. main.go初始化InformerFactory和controller

```go
/*
Copyright 2017 The Kubernetes Authors.
Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at
    http://www.apache.org/licenses/LICENSE-2.0
Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
*/

package main

import (
	"flag"
	"time"

	kubeinformers "k8s.io/client-go/informers"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/klog/v2"
	// Uncomment the following line to load the gcp plugin (only required to authenticate against GKE clusters).
	// _ "k8s.io/client-go/plugin/pkg/client/auth/gcp"

	clientset "k8s.io/sample-controller/pkg/generated/clientset/versioned"
	informers "k8s.io/sample-controller/pkg/generated/informers/externalversions"
	"k8s.io/sample-controller/pkg/signals"
)

var (
	masterURL  string
	kubeconfig string
)

func init() {
	flag.StringVar(&kubeconfig, "kubeconfig", "", "Path to a kubeconfig. Only required if out-of-cluster.")
	flag.StringVar(&masterURL, "master", "", "The address of the Kubernetes API server. Overrides any value in kubeconfig. Only required if out-of-cluster.")
}

func main() {
	klog.InitFlags(nil)
	flag.Parse()

	// set up signals so we handle the first shutdown signal gracefully
	stopCh := signals.SetupSignalHandler()

	cfg, err := clientcmd.BuildConfigFromFlags(masterURL, kubeconfig)
	if err != nil {
		klog.Fatalf("Error building kubeconfig: %s", err.Error())
	}

	kubeClient, err := kubernetes.NewForConfig(cfg)
	if err != nil {
		klog.Fatalf("Error building kubernetes clientset: %s", err.Error())
	}

	exampleClient, err := clientset.NewForConfig(cfg)
	if err != nil {
		klog.Fatalf("Error building example clientset: %s", err.Error())
	}

	kubeInformerFactory := kubeinformers.NewSharedInformerFactory(kubeClient, time.Second*30)
	exampleInformerFactory := informers.NewSharedInformerFactory(exampleClient, time.Second*30)

	controller := NewController(kubeClient, exampleClient,
		kubeInformerFactory.Apps().V1().Deployments(),
		exampleInformerFactory.Samplecontroller().V1alpha1().Foos())

	// notice that there is no need to run Start methods in a separate goroutine. (i.e. go kubeInformerFactory.Start(stopCh)
	// Start method is non-blocking and runs all registered informers in a dedicated goroutine.
	kubeInformerFactory.Start(stopCh)
	exampleInformerFactory.Start(stopCh)

	if err = controller.Run(2, stopCh); err != nil {
		klog.Fatalf("Error running controller: %s", err.Error())
	}
}

```

**写完这篇之后，发现有大神写了更详细的一篇，如果希望学习更细致的知识点，大家可以移步[自定义资源对象与控制器的实现](https://101.200.83.188/articles/000001640262318a67c149f524b43a6b2796c4ae753cf2b000#)[5]**

## 参考

1. kubebuilder [https://book.kubebuilder.io/cronjob-tutorial/basic-project.html](https://book.kubebuilder.io/cronjob-tutorial/basic-project.html)
2. 使用code-generator生成crd的clientset、informer、listers: [https://xieys.club/code-generator-crd/index.md](https://xieys.club/code-generator-crd/index.md)
3. kubebuilder和code-generator使用分享: [https://www.jianshu.com/p/78652aac89f4](https://www.jianshu.com/p/78652aac89f4)
4. kubernetes-sample-controller: https://github.com/kubernetes/sample-controller
5. 自定义资源对象与控制器的实现：[https://101.200.83.188/articles/000001640262318a67c149f524b43a6b2796c4ae753cf2b000#](https://101.200.83.188/articles/000001640262318a67c149f524b43a6b2796c4ae753cf2b000#)