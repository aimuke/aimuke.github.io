---
title: "k8s自定义controller三部曲之二:自动生成代码"
tags: [k8s, crd, controller]
---

**2019年03月31日 19:45:23 [程序员欣宸](https://me.csdn.net/boling_cavalry)**

**版权声明：欢迎转载，请注明 [出处](https://blog.csdn.net/boling_cavalry/article/details/88924194) ，谢谢。**

本文是《k8s自定义controller三部曲》的第二篇，[上一篇](https://aimuke.github.io/k8s/2019/08/01/k8s-crd-controller-1/) 我们在k8s环境注册了API对象 `Student` ，此时如果创建 `Student` 对象就会在 `etcd` 保存该对象信息；

三部曲所有文章链接
1. [《k8s自定义controller三部曲之一:创建CRD（Custom Resource Definition）》](https://aimuke.github.io/k8s/2019/08/01/k8s-crd-controller-1/)
2. [《k8s自定义controller三部曲之二:自动生成代码》](https://aimuke.github.io/k8s/2019/08/01/k8s-crd-controller-2/)
3. [《k8s自定义controller三部曲之三：编写controller代码》](https://aimuke.github.io/k8s/2019/08/01/k8s-crd-controller-3/)

**源码下载**

接下来详细讲述应用的编码过程，如果您不想自己写代码，也可以在GitHub下载完整的 [应用源码](https://github.com/zq2599/blog_demos)。这个git项目中有多个文件夹，本章源码在 `k8s_customize_controller` 这个文件夹下。

# 为什么要做controller
但如果仅仅是在 `etcd` 保存 `Student` 对象是没有什么意义的，试想通过 `deployment` 创建 `pod` 时，如果只在 `etcd` 创建 `pod` 对象，而不去 `node` 节点创建容器，那这个 `pod` 对象只是一条数据而已，没有什么实质性作用，其他对象如 `service` 、 `pv``也是如此；

`controller` 的作用就是监听指定对象的新增、删除、修改等变化，针对这些变化做出相应的响应（例如新增pod的响应为创建docker容器），关于controller的详细设计，最好的参考就是Harry (Lei) Zhang老师在twitter上的 [分享](https://twitter.com/resouer/status/1009996649832185856) ，如下图，地址是：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190331114526283.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly94aW5jaGVuLmJsb2cuY3Nkbi5uZXQ=,size_16,color_FFFFFF,t_70)

如上图，API对象的变化会通过 `Informer` 存入队列（ `WorkQueue` ），在 `Controller` 中消费队列的数据做出响应，响应相关的具体代码就是我们要做的真正业务逻辑；

# 自动生成代码是什么
从上图可以发现整个逻辑还是比较复杂的，为了简化我们的自定义controller开发，k8s的大师们利用自动代码生成工具将controller之外的事情都做好了，我们只要专注于controller的开发就好。
关于controller的代码我们留待下一章完成，本章关注的是如何自动生成代码的实战，将上图中controller之外的事情都完成了；

# 开始实战
接下来要做的事情就是编写API对象Student相关的声明的定义代码，然后用代码生成工具结合这些代码，自动生成 `Client` 、 `Informet` 、 `WorkQueue` 相关的代码；

## 创建项目

创建demo项目 `k8s_customize_controller`

```sh
cd $GOPATH/src
mkdir k8s_customize_controller
cd k8s_customize_controller
```

## 项目注册信息

建立自定义数据目录

```sh
mkdir -p pkg/apis/bolingcavalry
```

在新建的 `bolingcavalry` 目录下创建文件 `register.go`，内容如下：

```go
package bolingcavalry

const (
        GroupName = "bolingcavalry.k8s.io"
        Version   = "v1"
)
```

## 版本详细信息

在新建的 `bolingcavalry` 目录下创建名为 `v1` 的文件夹；

### doc.go

在新建的 `v1` 文件夹下创建文件 `doc.go`，内容如下：

```go
// +k8s:deepcopy-gen=package

// +groupName=bolingcavalry.k8s.io
package v1
```

上述代码中的两行注释，都是代码生成工具会用到的，一个是声明为整个 `v1` 包下的类型定义生成 `DeepCopy` 方法，另一个声明了这个包对应的`API`的组名，和`CRD`中的组名一致；

### types.go

在`v1`文件夹下创建文件`types.go`，里面定义了`Student`对象的具体内容：

```go
package v1

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// +genclient
// +genclient:noStatus
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

type Student struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`
	Spec              StudentSpec `json:"spec"`
}

type StudentSpec struct {
	name   string `json:"name"`
	school string `json:"school"`
}

// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// StudentList is a list of Student resources
type StudentList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata"`

	Items []Student `json:"items"`
}
```

从上述源码可见，Student对象的内容已经被设定好，主要有name和school这两个字段，表示学生的名字和所在学校，因此创建Student对象的时候内容就要和这里匹配了；

### register.go

在`v1`目录下创建`register.go`文件，此文件的作用是通过 `addKnownTypes` 方法使得client可以知道Student类型的API对象：

```go
package v1

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/runtime/schema"

	"k8s_customize_controller/pkg/apis/bolingcavalry"
)

var SchemeGroupVersion = schema.GroupVersion{
	Group:   bolingcavalry.GroupName,
	Version: bolingcavalry.Version,
}

var (
	SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
	AddToScheme   = SchemeBuilder.AddToScheme
)

func Resource(resource string) schema.GroupResource {
	return SchemeGroupVersion.WithResource(resource).GroupResource()
}

func Kind(kind string) schema.GroupKind {
	return SchemeGroupVersion.WithKind(kind).GroupKind()
}

func addKnownTypes(scheme *runtime.Scheme) error {
	scheme.AddKnownTypes(
		SchemeGroupVersion,
		&Student{},
		&StudentList{},
	)

	// register the type in the scheme
	metav1.AddToGroupVersion(scheme, SchemeGroupVersion)
	return nil
}
```

## 生成自动代码
至此，为自动生成代码做的准备工作已经完成了，目前为止，`$GOPATH/src`目录下的文件和目录结构是这样的：

```sh
[root@golang src]# tree
.
└── k8s_customize_controller
    └── pkg
        └── apis
            └── bolingcavalry
                ├── register.go
                └── v1
                    ├── doc.go
                    ├── register.go
                    └── types.go

5 directories, 4 files
```

执行以下命令，会先下载依赖包，再下载代码生成工具，再执行代码生成工作：

```sh
cd $GOPATH/src \
&& go get -u k8s.io/apimachinery/pkg/apis/meta/v1 \
&& go get -u k8s.io/code-generator/... \
&& cd $GOPATH/src/k8s.io/code-generator \
&& ./generate-groups.sh all \
k8s_customize_controller/pkg/client \
k8s_customize_controller/pkg/apis \
bolingcavalry:v1
```

如果代码写得没有问题，会看到以下输出：

```
Generating deepcopy funcs
Generating clientset for bolingcavalry:v1 at k8s_customize_controller/pkg/client/clientset
Generating listers for bolingcavalry:v1 at k8s_customize_controller/pkg/client/listers
Generating informers for bolingcavalry:v1 at k8s_customize_controller/pkg/client/informers
```

此时再去 `$GOPATH/src/k8s_customize_controller` 目录下执行 `tree` 命令，可见已生成了很多内容：

```sh
[root@master k8s_customize_controller]# tree
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

如上所示，`zz_generated.deepcopy.go` 就是 `DeepCopy` 代码文件， `client` 目录下的内容都是客户端相关代码，在开发 `controller` 时会用到；
 `client` 目录下的 `clientset、informers、listers` 的身份和作用可以和前面的图结合来理解；

至此，自动生成代码的步骤已经完成，我们已经为后面编写 `controller` 做好了充分的准备，接下来的章节就是编写 `controller` 了，点击链接继续实战：《k8s自定义controller三部曲之三：编写controller代码》

# Reference

- [原文-
k8s自定义controller三部曲之二:自动生成代码](https://blog.csdn.net/boling_cavalry/article/details/88924194)
