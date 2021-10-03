---
title: "k8s自定义controller三部曲之一:创建CRD（Custom Resource Definition）"
tags: [k8s, crd, controller]
---
**2019年03月31日 19:45:23 [程序员欣宸](https://me.csdn.net/boling_cavalry)**

**版权声明：欢迎转载，请注明 [出处](https://blog.csdn.net/boling_cavalry/article/details/88924194) ，谢谢。**

k8s系统中 `controller` 扮演着重要角色，开发自定义 `controller` 是深入学习和理解 `controller` 的有效途径，《k8s自定义controller三部曲》系列会逐步完成一次完整的自定义controller实战。

# 实战概要

整个三部曲的目标如下：

1. 创建自定义API对象（`Custom Resource Definition`），名为 `Student` ；
2. 用代码生成工具生成 `informer` 和 `client` 相关代码；
3. 创建并运行自定义控制器，k8s环境中所有 `Student` 相关的"增、删、改"操作都会被此控制器监听到，可以根据实际需求在控制器中编写业务代码；

# 环境信息

实战环境的版本信息如下，请确保以下软件都已运行正常（ `Etcd` 只是用来查看数据的，可以选择不装），并且 `kubectl` 工具可以在k8s环境正常操作：

1. 操作系统 ：CentOS Linux release 7.6.1810
2. Kubernetes：1.13
3. Go版本：1.12
4. Etcd：3.3.1（查看数据时用到，也可以不安装）

# 本篇概要
本篇是三部曲的第一篇，我们先自定义一个API对象 `Student` ，然后让k8s接收 `Student` 的定义，这样我们在k8s创建 `Student` 对象时，k8s就能接收并保存了，就像先有了 `pod` 定义，才能创建 `pod` 一样；

三部曲所有文章链接
1. [《k8s自定义controller三部曲之一:创建CRD（Custom Resource Definition）》](https://aimuke.github.io/k8s/2019/08/01/k8s-crd-controller-1/)
2. [《k8s自定义controller三部曲之二:自动生成代码》](https://aimuke.github.io/k8s/2019/08/01/k8s-crd-controller-2/)
3. [《k8s自定义controller三部曲之三：编写controller代码》](https://aimuke.github.io/k8s/2019/08/01/k8s-crd-controller-3/)

**源码下载**

接下来详细讲述应用的编码过程，如果您不想自己写代码，也可以在GitHub下载完整的 [应用源码](https://github.com/zq2599/blog_demos)。这个git项目中有多个文件夹，本章源码在 `k8s_customize_controller` 这个文件夹下。

# 创建CRD

创建 `CRD` 的第一步是通过 [官方文档](https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definitions/#create-a-customresourcedefinition) 做初步了解。

登录可以执行 `kubectl` 命令的机器，创建 `student.yaml` 文件，内容如下：

```yml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  # metadata.name的内容是由"复数名.分组名"构成，如下，students是复数名，bolingcavalry.k8s.io是分组名
  name: students.bolingcavalry.k8s.io
spec:
  # 分组名，在REST API中也会用到的，格式是: /apis/分组名/CRD版本
  group: bolingcavalry.k8s.io
  # list of versions supported by this CustomResourceDefinition
  versions:
    - name: v1
      # 是否有效的开关.
      served: true
      # 只有一个版本能被标注为storage
      storage: true
  # 范围是属于namespace的
  scope: Namespaced
  names:
    # 复数名
    plural: students
    # 单数名
    singular: student
    # 类型名
    kind: Student
    # 简称，就像service的简称是svc
    shortNames:
    - stu
```

在 `student.yaml` 所在目录执行命令 `kubectl apply -f student.yaml`，即可在k8s环境创建 `Student` 的定义，今后如果发起对类型为 `Student` 的对象的处理，k8s的 `api server` 就能识别到该对象类型了，如下所示，可以用 `kubectl get crd` 和 `kubectl describe crd stu` 命令查看更多细节， `stu` 是在  `student.yaml` 中定义的简称：

```sh
[root@master custom_controller]# kubectl apply -f student.yaml
customresourcedefinition.apiextensions.k8s.io/students.bolingcavalry.k8s.io created
[root@master custom_controller]# kubectl get crd
NAME                            CREATED AT
students.bolingcavalry.k8s.io   2019-03-30T13:33:13Z
[root@master custom_controller]# kubectl describe crd stu
Name:         students.bolingcavalry.k8s.io
Namespace:    
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"apiextensions.k8s.io/v1beta1","kind":"CustomResourceDefinition","metadata":{"annotations":{},"name":"students.bolingcavalry...
API Version:  apiextensions.k8s.io/v1beta1
Kind:         CustomResourceDefinition
Metadata:
  Creation Timestamp:  2019-03-30T13:33:13Z
  Generation:          1
  Resource Version:    292010
  Self Link:           /apis/apiextensions.k8s.io/v1beta1/customresourcedefinitions/students.bolingcavalry.k8s.io
  UID:                 5e4ceb6e-52f0-11e9-96e1-000c29f1f9c9
  ...
  ..
  .
```

如果您已配置好 `etcdctl` ，可以访问k8s的 `etcd` 上存储的数据，那么执行以下命令，就可以看到新的 `CRD` 已经保存在 `etcd` 中了：

```sh
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt --key=/etc/kubernetes/pki/etcd/healthcheck-client.key get /registry/apiextensions.k8s.io/customresourcedefinitions/ --prefix
```

上述命令用来查看路径 `/registry/apiextensions.k8s.io/customresourcedefinitions/` 之下的所有键值对，可以见到刚才创建的 `CRD` 信息，如下：

```json
/registry/apiextensions.k8s.io/customresourcedefinitions/students.bolingcavalry.k8s.io{
    "kind": "CustomResourceDefinition",
    "apiVersion": "apiextensions.k8s.io/v1beta1",
    "metadata": {
        "name": "students.bolingcavalry.k8s.io",
        "uid": "5e4ceb6e-52f0-11e9-96e1-000c29f1f9c9",
        "generation": 1,
        "creationTimestamp": "2019-03-30T13:33:13Z",
        "annotations": {
            "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"apiextensions.k8s.io/v1beta1\",\"kind\":\"CustomResourceDefinition\",\"metadata\":{\"annotations\":{},\"name\":\"students.bolingcavalry.k8s.io\"},\"spec\":{\"group\":\"bolingcavalry.k8s.io\",\"names\":{\"kind\":\"Student\",\"plural\":\"students\",\"shortNames\":[\"stu\"],\"singular\":\"student\"},\"scope\":\"Namespaced\",\"versions\":[{\"name\":\"v1\",\"served\":true,\"storage\":true}]}}\n"
        }
    },
    "spec": {
        "group": "bolingcavalry.k8s.io",
        "version": "v1",
        "names": {
            "plural": "students",
            "singular": "student",
            "shortNames": [
                "stu"
            ],
            "kind": "Student",
            "listKind": "StudentList"
        },
        "scope": "Namespaced",
        "versions": [
            {
                "name": "v1",
                "served": true,
                "storage": true
            }
        ],
        "conversion": {
            "strategy": "None"
        }
    },
    "status": {
        "conditions": [
            {
                "type": "NamesAccepted",
                "status": "True",
                "lastTransitionTime": "2019-03-30T13:33:13Z",
                "reason": "NoConflicts",
                "message": "no conflicts found"
            },
            {
                "type": "Established",
                "status": "True",
                "lastTransitionTime": null,
                "reason": "InitialNamesAccepted",
                "message": "the initial names have been accepted"
            }
        ],
        "acceptedNames": {
            "plural": "students",
            "singular": "student",
            "shortNames": [
                "stu"
            ],
            "kind": "Student",
            "listKind": "StudentList"
        },
        "storedVersions": [
            "v1"
        ]
    }
}
```

# 创建Student对象

前面的步骤使得k8s能识别 `Student` 类型了，接下来创建个 `Student` 对象试试；

创建 `object-student.yaml` 文件，内容如下：

```yaml
apiVersion: bolingcavalry.k8s.io/v1
kind: Student
metadata:
  name: object-student
spec:
  name: "张三"
  school: "深圳中学"
```

在 `object-student.yaml` 文件所在目录执行命令 `kubectl apply -f object-student.yaml`，会看到提示创建成功：

```sh
[root@master custom_controller]# kubectl apply -f object-student.yaml 
student.bolingcavalry.k8s.io/object-student created
```

执行命令 `kubectl get stu` 可见已创建成功的 `Student` 对象：

```sh
[root@master custom_controller]# kubectl get stu
NAME             AGE
object-student   15s
```

创建成功的 `Student` 对象存储在 `etcd` 中是什么样的呢，如果您的 `etcdctl` 已经配置好，执行以下命令即可：

```sh
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt --key=/etc/kubernetes/pki/etcd/healthcheck-client.key get /registry/bolingcavalry.k8s.io/students/default/object-student --print-value-only
```

控制台输出的就是该 `Student` 对象存储在 `etcd` 中的内容，如下：

```json
{
    "apiVersion": "bolingcavalry.k8s.io/v1",
    "kind": "Student",
    "metadata": {
        "annotations": {
            "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"bolingcavalry.k8s.io/v1\",\"kind\":\"Student\",\"metadata\":{\"annotations\":{},\"name\":\"object-student\",\"namespace\":\"default\"},\"spec\":{\"name\":\"张三\",\"school\":\"深圳中学\"}}\n"
        },
        "creationTimestamp": "2019-03-31T02:56:25Z",
        "generation": 1,
        "name": "object-student",
        "namespace": "default",
        "uid": "92927d0d-5360-11e9-9d2a-000c29f1f9c9"
    },
    "spec": {
        "name": "张三",
        "school": "深圳中学"
    }
}
```

至此，自定义API对象（也就是 `CRD` ）就创建成功了，此刻我们只是让k8s能识别到 `Student` 这个对象的身份，但是当我们创建 `Student` 对象的时候，还没有触发任何业务（相对于创建 `Pod` 对象的时候，会触发 `kubelet` 在node节点创建docker容器），这也是后面的章节要完成的任务，点击链接进入下一段实战： 《k8s自定义controller三部曲之二:自动生成代码》

# Reference

- [原文-k8s自定义controller三部曲之一:创建CRD（Custom Resource Definition）](https://blog.csdn.net/boling_cavalry/article/details/88917818)
