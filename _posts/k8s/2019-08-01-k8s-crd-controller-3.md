---
title: "k8s自定义controller三部曲之三：编写controller代码"
tags: [k8s, crd, controller]
---

**2019年04月01日 00:15:25 [程序员欣宸](https://me.csdn.net/boling_cavalry)**
 
**版权声明：欢迎转载，请注明 [出处](https://blog.csdn.net/boling_cavalry/article/details/88934063)，谢谢。**

本文是《k8s自定义controller三部曲》的终篇，前面的章节中，我们创建了 `CRD` ([创建CRD](https://aimuke.github.io/k8s/2019/08/01/k8s-crd-controller-1/))，再通过自动生成代码的工具将 `controller` 所需的 `informer` 、 `client` 等依赖全部准备好([自动生成代码](https://aimuke.github.io/k8s/2019/08/01/k8s-crd-controller-2/))，到了本章，就该编写 `controller` 的代码了，也就是说，现在已经能监听到 `Student` 对象的增删改等事件，接下来就是根据这些事件来做不同的事情，满足个性化的业务需求；

三部曲所有文章链接
1. [《k8s自定义controller三部曲之一:创建CRD（Custom Resource Definition）》](https://aimuke.github.io/k8s/2019/08/01/k8s-crd-controller-1/)
2. [《k8s自定义controller三部曲之二:自动生成代码》](https://aimuke.github.io/k8s/2019/08/01/k8s-crd-controller-2/)
3. [《k8s自定义controller三部曲之三：编写controller代码》](https://aimuke.github.io/k8s/2019/08/01/k8s-crd-controller-3/)

**源码下载**

接下来详细讲述应用的编码过程，如果您不想自己写代码，也可以在GitHub下载完整的 [应用源码](https://github.com/zq2599/blog_demos)。这个git项目中有多个文件夹，本章源码在 `k8s_customize_controller` 这个文件夹下。

# 开始实战

回顾一下，上一章通过自动代码生成工具生成代码后，源码目录的内容如下：

```sh
[root@golang k8s_customize_controller]# tree
.
└── pkg
    ├── apis
    │   └── bolingcavalry
    │       ├── register.go
    │       └── v1
    │           ├── doc.go
    │           ├── register.go
    │           ├── types.go
    │           └── zz_generated.deepcopy.go
    └── client
        ├── clientset
        │   └── versioned
        │       ├── clientset.go
        │       ├── doc.go
        │       ├── fake
        │       │   ├── clientset_generated.go
        │       │   ├── doc.go
        │       │   └── register.go
        │       ├── scheme
        │       │   ├── doc.go
        │       │   └── register.go
        │       └── typed
        │           └── bolingcavalry
        │               └── v1
        │                   ├── bolingcavalry_client.go
        │                   ├── doc.go
        │                   ├── fake
        │                   │   ├── doc.go
        │                   │   ├── fake_bolingcavalry_client.go
        │                   │   └── fake_student.go
        │                   ├── generated_expansion.go
        │                   └── student.go
        ├── informers
        │   └── externalversions
        │       ├── bolingcavalry
        │       │   ├── interface.go
        │       │   └── v1
        │       │       ├── interface.go
        │       │       └── student.go
        │       ├── factory.go
        │       ├── generic.go
        │       └── internalinterfaces
        │           └── factory_interfaces.go
        └── listers
            └── bolingcavalry
                └── v1
                    ├── expansion_generated.go
                    └── student.go

21 directories, 27 files
```

# controller.go

本章要编写的第一个go文件就是 `controller.go` ，在 `k8s_customize_controller` 目录下创建 `controller.go` ，代码内容如下：

```go
package main

import (
	"fmt"
	"time"

	"github.com/golang/glog"
	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/api/errors"
	"k8s.io/apimachinery/pkg/util/runtime"
	utilruntime "k8s.io/apimachinery/pkg/util/runtime"
	"k8s.io/apimachinery/pkg/util/wait"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/kubernetes/scheme"
	typedcorev1 "k8s.io/client-go/kubernetes/typed/core/v1"
	"k8s.io/client-go/tools/cache"
	"k8s.io/client-go/tools/record"
	"k8s.io/client-go/util/workqueue"

	bolingcavalryv1 "github.com/zq2599/k8s-controller-custom-resource/pkg/apis/bolingcavalry/v1"
	clientset "github.com/zq2599/k8s-controller-custom-resource/pkg/client/clientset/versioned"
	studentscheme "github.com/zq2599/k8s-controller-custom-resource/pkg/client/clientset/versioned/scheme"
	informers "github.com/zq2599/k8s-controller-custom-resource/pkg/client/informers/externalversions/bolingcavalry/v1"
	listers "github.com/zq2599/k8s-controller-custom-resource/pkg/client/listers/bolingcavalry/v1"
)

const controllerAgentName = "student-controller"

const (
	SuccessSynced = "Synced"

	MessageResourceSynced = "Student synced successfully"
)

// Controller is the controller implementation for Student resources
type Controller struct {
	// kubeclientset is a standard kubernetes clientset
	kubeclientset kubernetes.Interface
	// studentclientset is a clientset for our own API group
	studentclientset clientset.Interface

	studentsLister listers.StudentLister
	studentsSynced cache.InformerSynced

	workqueue workqueue.RateLimitingInterface

	recorder record.EventRecorder
}

// NewController returns a new student controller
func NewController(
	kubeclientset kubernetes.Interface,
	studentclientset clientset.Interface,
	studentInformer informers.StudentInformer) *Controller {

	utilruntime.Must(studentscheme.AddToScheme(scheme.Scheme))
	glog.V(4).Info("Creating event broadcaster")
	eventBroadcaster := record.NewBroadcaster()
	eventBroadcaster.StartLogging(glog.Infof)
	eventBroadcaster.StartRecordingToSink(&typedcorev1.EventSinkImpl{Interface: kubeclientset.CoreV1().Events("")})
	recorder := eventBroadcaster.NewRecorder(scheme.Scheme, corev1.EventSource{Component: controllerAgentName})

	controller := &Controller{
		kubeclientset:    kubeclientset,
		studentclientset: studentclientset,
		studentsLister:   studentInformer.Lister(),
		studentsSynced:   studentInformer.Informer().HasSynced,
		workqueue:        workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), "Students"),
		recorder:         recorder,
	}

	glog.Info("Setting up event handlers")
	// Set up an event handler for when Student resources change
	studentInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: controller.enqueueStudent,
		UpdateFunc: func(old, new interface{}) {
			oldStudent := old.(*bolingcavalryv1.Student)
			newStudent := new.(*bolingcavalryv1.Student)
			if oldStudent.ResourceVersion == newStudent.ResourceVersion {
                //版本一致，就表示没有实际更新的操作，立即返回
				return
			}
			controller.enqueueStudent(new)
		},
		DeleteFunc: controller.enqueueStudentForDelete,
	})

	return controller
}

//在此处开始controller的业务
func (c *Controller) Run(threadiness int, stopCh <-chan struct{}) error {
	defer runtime.HandleCrash()
	defer c.workqueue.ShutDown()

	glog.Info("开始controller业务，开始一次缓存数据同步")
	if ok := cache.WaitForCacheSync(stopCh, c.studentsSynced); !ok {
		return fmt.Errorf("failed to wait for caches to sync")
	}

	glog.Info("worker启动")
	for i := 0; i < threadiness; i++ {
		go wait.Until(c.runWorker, time.Second, stopCh)
	}

	glog.Info("worker已经启动")
	<-stopCh
	glog.Info("worker已经结束")

	return nil
}

func (c *Controller) runWorker() {
	for c.processNextWorkItem() {
	}
}

// 取数据处理
func (c *Controller) processNextWorkItem() bool {

	obj, shutdown := c.workqueue.Get()

	if shutdown {
		return false
	}

	// We wrap this block in a func so we can defer c.workqueue.Done.
	err := func(obj interface{}) error {
		defer c.workqueue.Done(obj)
		var key string
		var ok bool

		if key, ok = obj.(string); !ok {

			c.workqueue.Forget(obj)
			runtime.HandleError(fmt.Errorf("expected string in workqueue but got %#v", obj))
			return nil
		}
		// 在syncHandler中处理业务
		if err := c.syncHandler(key); err != nil {
			return fmt.Errorf("error syncing '%s': %s", key, err.Error())
		}

		c.workqueue.Forget(obj)
		glog.Infof("Successfully synced '%s'", key)
		return nil
	}(obj)

	if err != nil {
		runtime.HandleError(err)
		return true
	}

	return true
}

// 处理
func (c *Controller) syncHandler(key string) error {
	// Convert the namespace/name string into a distinct namespace and name
	namespace, name, err := cache.SplitMetaNamespaceKey(key)
	if err != nil {
		runtime.HandleError(fmt.Errorf("invalid resource key: %s", key))
		return nil
	}

	// 从缓存中取对象
	student, err := c.studentsLister.Students(namespace).Get(name)
	if err != nil {
		// 如果Student对象被删除了，就会走到这里，所以应该在这里加入执行
		if errors.IsNotFound(err) {
			glog.Infof("Student对象被删除，请在这里执行实际的删除业务: %s/%s ...", namespace, name)

			return nil
		}

		runtime.HandleError(fmt.Errorf("failed to list student by: %s/%s", namespace, name))

		return err
	}

	glog.Infof("这里是student对象的期望状态: %#v ...", student)
	glog.Infof("实际状态是从业务层面得到的，此处应该去的实际状态，与期望状态做对比，并根据差异做出响应(新增或者删除)")

	c.recorder.Event(student, corev1.EventTypeNormal, SuccessSynced, MessageResourceSynced)
	return nil
}

// 数据先放入缓存，再入队列
func (c *Controller) enqueueStudent(obj interface{}) {
	var key string
	var err error
	// 将对象放入缓存
	if key, err = cache.MetaNamespaceKeyFunc(obj); err != nil {
		runtime.HandleError(err)
		return
	}

	// 将key放入队列
	c.workqueue.AddRateLimited(key)
}

// 删除操作
func (c *Controller) enqueueStudentForDelete(obj interface{}) {
	var key string
	var err error
	// 从缓存中删除指定对象
	key, err = cache.DeletionHandlingMetaNamespaceKeyFunc(obj)
	if err != nil {
		runtime.HandleError(err)
		return
	}
	//再将key放入队列
	c.workqueue.AddRateLimited(key)
}
```

上述代码有以下几处关键点：
- 创建 `controller` 的 `NewController` 方法中，定义了收到 `Student` 对象的增删改消息时的具体处理逻辑，除了同步本地缓存，就是将该对象的 `key` 放入消息中；
- 实际处理消息的方法是 `syncHandler` ，这里面可以添加实际的业务代码，来响应 `Student` 对象的增删改情况，达到业务目的；

# signals

接下来可以写 `main.go` 了，不过在此之前把处理系统信号量的辅助类先写好，然后在 main.go 中会用到（处理例如 `ctrl+c` 的退出），在 `$GOPATH/src/k8s_customize_controller/pkg` 目录下新建目录 `signals` ；

## signal_posix.go
在 `signals` 目录下新建文件 `signal_posix.go` ：

```go
// +build !windows

package signals

import (
	"os"
	"syscall"
)

var shutdownSignals = []os.Signal{os.Interrupt, syscall.SIGTERM}
```

## signal_windows.go

在 `signals` 目录下新建文件 `signal_windows.go` ：

```go
package signals

import (
	"os"
)

var shutdownSignals = []os.Signal{os.Interrupt}
```

## signal.go

在 `signals` 目录下新建文件 `signal.go` ：

```go
package signals

import (
        "os"
        "os/signal"
)

var onlyOneSignalHandler = make(chan struct{})

func SetupSignalHandler() (stopCh <-chan struct{}) {
        close(onlyOneSignalHandler) // panics when called twice

        stop := make(chan struct{})
        c := make(chan os.Signal, 2)
        signal.Notify(c, shutdownSignals...)
        go func() {
                <-c
                close(stop)
                <-c
                os.Exit(1) // second signal. Exit directly.
        }()

        return stop
}
```

# main.go
接下来可以编写 `main.go` 了，在 `k8s_customize_controller` 目录下创建 `main.go` 文件，内容如下，关键位置已经加了注释，就不再赘述了：

```go
package main

import (
	"flag"
	"time"

	"github.com/golang/glog"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
	// Uncomment the following line to load the gcp plugin (only required to authenticate against GKE clusters).
	// _ "k8s.io/client-go/plugin/pkg/client/auth/gcp"

	clientset "k8s_customize_controller/pkg/client/clientset/versioned"
	informers "k8s_customize_controller/pkg/client/informers/externalversions"
	"k8s_customize_controller/pkg/signals"
)

var (
	masterURL  string
	kubeconfig string
)

func main() {
	flag.Parse()

	// 处理信号量
	stopCh := signals.SetupSignalHandler()

    // 处理入参
	cfg, err := clientcmd.BuildConfigFromFlags(masterURL, kubeconfig)
	if err != nil {
		glog.Fatalf("Error building kubeconfig: %s", err.Error())
	}

	kubeClient, err := kubernetes.NewForConfig(cfg)
	if err != nil {
		glog.Fatalf("Error building kubernetes clientset: %s", err.Error())
	}

	studentClient, err := clientset.NewForConfig(cfg)
	if err != nil {
		glog.Fatalf("Error building example clientset: %s", err.Error())
	}

	studentInformerFactory := informers.NewSharedInformerFactory(studentClient, time.Second*30)

    //得到controller
	controller := NewController(kubeClient, studentClient,
		studentInformerFactory.Bolingcavalry().V1().Students())

    //启动informer
	go studentInformerFactory.Start(stopCh)

    //controller开始处理消息
	if err = controller.Run(2, stopCh); err != nil {
		glog.Fatalf("Error running controller: %s", err.Error())
	}
}

func init() {
	flag.StringVar(&kubeconfig, "kubeconfig", "", "Path to a kubeconfig. Only required if out-of-cluster.")
	flag.StringVar(&masterURL, "master", "", "The address of the Kubernetes API server. Overrides any value in kubeconfig. Only required if out-of-cluster.")
}

```

# 编译和运行
至此，所有代码已经编写完毕，接下来是编译构建；

- 为项目添加依赖具体的可以参照k8s的 [官方demo](https://github.com/kubernetes/sample-controller) ，这个代码中同时提供了 `godep` 和 `vendor` 两种方式来处理上面的包依赖问题。也可以直接在 `$GOPATH/src/k8s_customize_controller` 目录下添加依赖文件，执行以下命令：

```sh
go get k8s.io/client-go/kubernetes/scheme \
&& go get github.com/golang/glog \
&& go get k8s.io/kube-openapi/pkg/util/proto \
&& go get k8s.io/utils/buffer \
&& go get k8s.io/utils/integer \
&& go get k8s.io/utils/trace
```

- 解决了包依赖问题后，在 `$GOPATH/src/k8s_customize_controller` 目录下执行命令 `go build`，即可在当前目录生成 `k8s_customize_controller` 文件；

- 将文件 `k8s_customize_controller` 复制到 `k8s` 环境中，记得通过 `chmod a+x` 命令给其可执行权限；

- 执行命令 `./k8s_customize_controller -kubeconfig=$HOME/.kube/config -alsologtostderr=true` ，会立即启动 `controller`，看到控制台输出如下：

```sh
[root@master 31]# ./k8s_customize_controller -kubeconfig=$HOME/.kube/config -alsologtostderr=true
I0331 23:27:17.909265   21540 controller.go:72] Setting up event handlers
I0331 23:27:17.909450   21540 controller.go:96] 开始controller业务，开始一次缓存数据同步
I0331 23:27:18.110448   21540 controller.go:101] worker启动
I0331 23:27:18.110516   21540 controller.go:106] worker已经启动
I0331 23:27:18.110653   21540 controller.go:181] 这里是student对象的期望状态: &v1.Student{TypeMeta:v1.TypeMeta{Kind:"Student", APIVersion:"bolingcavalry.k8s.io/v1"}, ObjectMeta:v1.ObjectMeta{Name:"object-student", GenerateName:"", Namespace:"default", SelfLink:"/apis/bolingcavalry.k8s.io/v1/namespaces/default/students/object-student", UID:"92927d0d-5360-11e9-9d2a-000c29f1f9c9", ResourceVersion:"310395", Generation:1, CreationTimestamp:v1.Time{Time:time.Time{wall:0x0, ext:63689597785, loc:(*time.Location)(0x1f9c200)}}, DeletionTimestamp:(*v1.Time)(nil), DeletionGracePeriodSeconds:(*int64)(nil), Labels:map[string]string(nil), Annotations:map[string]string{"kubectl.kubernetes.io/last-applied-configuration":"{\"apiVersion\":\"bolingcavalry.k8s.io/v1\",\"kind\":\"Student\",\"metadata\":{\"annotations\":{},\"name\":\"object-student\",\"namespace\":\"default\"},\"spec\":{\"name\":\"张三\",\"school\":\"深圳中学\"}}\n"}, OwnerReferences:[]v1.OwnerReference(nil), Initializers:(*v1.Initializers)(nil), Finalizers:[]string(nil), ClusterName:"", ManagedFields:[]v1.ManagedFieldsEntry(nil)}, Spec:v1.StudentSpec{name:"", school:""}} ...
I0331 23:27:18.111105   21540 controller.go:182] 实际状态是从业务层面得到的，此处应该去的实际状态，与期望状态做对比，并根据差异做出响应(新增或者删除)
I0331 23:27:18.111187   21540 controller.go:145] Successfully synced 'default/object-student'
I0331 23:27:18.112263   21540 event.go:209] Event(v1.ObjectReference{Kind:"Student", Namespace:"default", Name:"object-student", UID:"92927d0d-5360-11e9-9d2a-000c29f1f9c9", APIVersion:"bolingcavalry.k8s.io/v1", ResourceVersion:"310395", FieldPath:""}): type: 'Normal' reason: 'Synced' Student synced successfully
```

至此，自定义controller已经启动成功了，并且从缓存中获取到了上一章中创建的对象的信息，接下来我们在k8s环境对 `Student` 对象做增删改，看看controller是否能做出响应；

# 验证controller
新开一个窗口连接到k8s环境，新建一个名为 `new-student.yaml` 的文件，内容如下：

```yml
apiVersion: bolingcavalry.k8s.io/v1
kind: Student
metadata:
  name: new-student
spec:
  name: "李四"
  school: "深圳小学"
```

在 `new-student.yaml` 所在目录执行命令 `kubectl apply -f new-student.yaml`

返回 `controller` 所在的控制台窗口，发现新输出了如下内容，可见新增 student对象的事件已经被controller监听并处理：

```sh
I0331 23:43:03.789894   21540 controller.go:181] 这里是student对象的期望状态: &v1.Student{TypeMeta:v1.TypeMeta{Kind:"", APIVersion:""}, ObjectMeta:v1.ObjectMeta{Name:"new-student", GenerateName:"", Namespace:"default", SelfLink:"/apis/bolingcavalry.k8s.io/v1/namespaces/default/students/new-student", UID:"abcd77d6-53cb-11e9-9d2a-000c29f1f9c9", ResourceVersion:"370653", Generation:1, CreationTimestamp:v1.Time{Time:time.Time{wall:0x0, ext:63689643783, loc:(*time.Location)(0x1f9c200)}}, DeletionTimestamp:(*v1.Time)(nil), DeletionGracePeriodSeconds:(*int64)(nil), Labels:map[string]string(nil), Annotations:map[string]string{"kubectl.kubernetes.io/last-applied-configuration":"{\"apiVersion\":\"bolingcavalry.k8s.io/v1\",\"kind\":\"Student\",\"metadata\":{\"annotations\":{},\"name\":\"new-student\",\"namespace\":\"default\"},\"spec\":{\"name\":\"李四\",\"school\":\"深圳小学\"}}\n"}, OwnerReferences:[]v1.OwnerReference(nil), Initializers:(*v1.Initializers)(nil), Finalizers:[]string(nil), ClusterName:"", ManagedFields:[]v1.ManagedFieldsEntry(nil)}, Spec:v1.StudentSpec{name:"", school:""}} ...
I0331 23:43:03.790076   21540 controller.go:182] 实际状态是从业务层面得到的，此处应该去的实际状态，与期望状态做对比，并根据差异做出响应(新增或者删除)
I0331 23:43:03.790120   21540 controller.go:145] Successfully synced 'default/new-student'
I0331 23:43:03.790141   21540 event.go:209] Event(v1.ObjectReference{Kind:"Student", Namespace:"default", Name:"new-student", UID:"abcd77d6-53cb-11e9-9d2a-000c29f1f9c9", APIVersion:"bolingcavalry.k8s.io/v1", ResourceVersion:"370653", FieldPath:""}): type: 'Normal' reason: 'Synced' Student synced successfully
```

接下来您也可以尝试修改和删除已有的Student对象，观察controller控制台的输出，确定是否已经监听到所有student变化的事件，例如删除的事件日志如下：

```
I0331 23:44:37.236090   21540 controller.go:171] Student对象被删除，请在这里执行实际的删除业务: default/new-student ...
I0331 23:44:37.236118   21540 controller.go:145] Successfully synced 'default/new-student'
```

# 小结
至此， `controller` 的编码和验证就全部完成了，现在小结一下自定义 `controller` 开发的整个过程：

1. 创建 `CRD（Custom Resource Definition）`，令k8s明白我们自定义的API对象；
2. 编写代码，将 `CRD` 的情况写入对应的代码中，然后通过自动代码生成工具，将 `controller` 之外的 `informer` ， `client` 等内容较为固定的代码通过工具生成；
3. 编写 `controller` ，在里面判断实际情况是否达到了API对象的声明情况，如果未达到，就要进行实际业务处理，而这也是 `controller` 的通用做法；

实际编码过程并不复杂，动手编写的文件如下：

```sh
├── controller.go
├── main.go
└── pkg
    ├── apis
    │   └── bolingcavalry
    │       ├── register.go
    │       └── v1
    │           ├── doc.go
    │           ├── register.go
    │           └── types.go
    └── signals
        ├── signal.go
        ├── signal_posix.go
        └── signal_windows.go
```

以上就是k8s自定义controller的整个开发过程，希望在您的开发过程中本文能提供一些参考；


# Reference

- [原文-k8s自定义controller三部曲之三：编写controller代码](https://blog.csdn.net/boling_cavalry/article/details/88934063)
