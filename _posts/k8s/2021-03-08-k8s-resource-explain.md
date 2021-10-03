---
title: "kubectl技巧之查看资源列表,资源版本和资源schema配置"
tags: [k8s, explain, resource, "api-resources", "api-versions"]
list_number: n
---

在kubernetes里 `pod,service,rs,rc,deploy,resource` 等对象都需要使用yaml文件来创建,很多时候我们都是参照照官方示例或者一些第三方示例来编写yaml文件以创建对象.虽然这些示例很有典型性和代表性,能够满足我们大部分时候的需求,然而这往往还是不够的,根据项目不同,实际配置可能远比官方提供的demo配置复杂的多,这就要求我们除了掌握常用的配置外,还需要对其它配置有所了解.如果有一个文档能够速查某一对象的所有配置,不但方便我们学习不同的配置,也可以做为一个小手册以便我们记不起来某些配置时可以速查.

下面我们介绍一些小技巧来快速查看kubernetes api

# 查看所有api资源 <a id="&#x67E5;&#x770B;&#x6240;&#x6709;api&#x8D44;&#x6E90;"></a>

可以通过命令`kubectl api-resources`来查看所有api资源

```bash
[centos@k8s-master ~]$ kubectl api-resources
NAME                              SHORTNAMES   APIGROUP                       NAMESPACED   KIND
bindings                                                                      true         Binding
componentstatuses                 cs                                          false        ComponentStatus
configmaps                        cm                                          true         ConfigMap
endpoints                         ep                                          true         Endpoints
events                            ev                                          true         Event
limitranges                       limits                                      true         LimitRange
namespaces                        ns                                          false        Namespace
nodes                             no                                          false        Node
persistentvolumeclaims            pvc                                         true         PersistentVolumeClaim
persistentvolumes                 pv                                          false        PersistentVolume
pods                              po                                          true         Pod
podtemplates                                                                  true         PodTemplate
replicationcontrollers            rc                                          true         ReplicationController
resourcequotas                    quota                                       true         ResourceQuota
secrets                                                                       true         Secret
serviceaccounts                   sa                                          true         ServiceAccount
services                          svc                                         true         Service
mutatingwebhookconfigurations                  admissionregistration.k8s.io   false        MutatingWebhookConfiguration
validatingwebhookconfigurations                admissionregistration.k8s.io   false        ValidatingWebhookConfiguration
customresourcedefinitions         crd,crds     apiextensions.k8s.io           false        CustomResourceDefinition
apiservices                                    apiregistration.k8s.io         false        APIService
controllerrevisions                            apps                           true         ControllerRevision
daemonsets                        ds           apps                           true         DaemonSet
deployments                       deploy       apps                           true         Deployment
replicasets                       rs           apps                           true         ReplicaSet
statefulsets                      sts          apps                           true         StatefulSet
tokenreviews                                   authentication.k8s.io          false        TokenReview
localsubjectaccessreviews                      authorization.k8s.io           true         LocalSubjectAccessReview
selfsubjectaccessreviews                       authorization.k8s.io           false        SelfSubjectAccessReview
selfsubjectrulesreviews                        authorization.k8s.io           false        SelfSubjectRulesReview
subjectaccessreviews                           authorization.k8s.io           false        SubjectAccessReview
horizontalpodautoscalers          hpa          autoscaling                    true         HorizontalPodAutoscaler
cronjobs                          cj           batch                          true         CronJob
jobs                                           batch                          true         Job
certificatesigningrequests        csr          certificates.k8s.io            false        CertificateSigningRequest
leases                                         coordination.k8s.io            true         Lease
events                            ev           events.k8s.io                  true         Event
daemonsets                        ds           extensions                     true         DaemonSet
deployments                       deploy       extensions                     true         Deployment
ingresses                         ing          extensions                     true         Ingress
networkpolicies                   netpol       extensions                     true         NetworkPolicy
podsecuritypolicies               psp          extensions                     false        PodSecurityPolicy
replicasets                       rs           extensions                     true         ReplicaSet
networkpolicies                   netpol       networking.k8s.io              true         NetworkPolicy
poddisruptionbudgets              pdb          policy                         true         PodDisruptionBudget
podsecuritypolicies               psp          policy                         false        PodSecurityPolicy
clusterrolebindings                            rbac.authorization.k8s.io      false        ClusterRoleBinding
clusterroles                                   rbac.authorization.k8s.io      false        ClusterRole
rolebindings                                   rbac.authorization.k8s.io      true         RoleBinding
roles                                          rbac.authorization.k8s.io      true         Role
priorityclasses                   pc           scheduling.k8s.io              false        PriorityClass
storageclasses                    sc           storage.k8s.io                 false        StorageClass
volumeattachments                              storage.k8s.io                 false        VolumeAttachment

```

除了可以看到资源的对象名称外,还可以看到对象的别名,这时候我们再看到别人的命令如`kubectl get no`这样费解的命令时就可以知道它实际上代表的是`kubectl get nodes`命令

> 查看api的版本,很多yaml配置里都需要指定配置的资源版本,我们经常看到v1,beta1,beta2这样的配置,到底某个资源的最新版本是什么呢?

其实,可以通过`kubectl api-versions`来查看api的版本

```text
[centos@k8s-master ~]$ kubectl api-versions
admissionregistration.k8s.io/v1beta1
apiextensions.k8s.io/v1beta1
apiregistration.k8s.io/v1
apiregistration.k8s.io/v1beta1
apps/v1
apps/v1beta1
apps/v1beta2
authentication.k8s.io/v1
authentication.k8s.io/v1beta1
authorization.k8s.io/v1
authorization.k8s.io/v1beta1
autoscaling/v1
autoscaling/v2beta1
autoscaling/v2beta2
batch/v1
batch/v1beta1
certificates.k8s.io/v1beta1
coordination.k8s.io/v1beta1
events.k8s.io/v1beta1
extensions/v1beta1
networking.k8s.io/v1
policy/v1beta1
rbac.authorization.k8s.io/v1
rbac.authorization.k8s.io/v1beta1
scheduling.k8s.io/v1beta1
storage.k8s.io/v1
storage.k8s.io/v1beta1
v1
```

以上只是整体概况,很多时候我们还想要看到某个api下面都有哪些配置,某一荐配置的含义等,下面罗列一些常用的api范例和一些查看api的技巧

# 常见范例

* [Replica Sets](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/#example)
* [Replication Controller](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/#running-an-example-replicationcontroller)
* [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#creating-a-deployment)
* [Service](https://kubernetes.io/docs/concepts/services-networking/service/#defining-a-service)

# 通过kubectl explain查看api字段

## 通过`kubectl explain <资源名对象名>`查看资源对象拥有的字段

前面说过,可以通过`kubectl api-resources`来查看资源名称,如果想要查看某个资源的字段,可以通过`kubectl explain <资源名对象名>`来查点它都有哪些字段

```text
[centos@k8s-master ~]$ kubectl explain pod
KIND:     Pod
VERSION:  v1

DESCRIPTION:
     Pod is a collection of containers that can run on a host. This resource is
     created by clients and scheduled onto hosts.

FIELDS:
   apiVersion   <string>
     APIVersion defines the versioned schema of this representation of an
     object. Servers should convert recognized schemas to the latest internal
     value, and may reject unrecognized values. More info:
     https://git.k8s.io/community/contributors/devel/api-conventions.md#resources

   kind <string>
     Kind is a string value representing the REST resource this object
     represents. Servers may infer this from the endpoint the client submits
     requests to. Cannot be updated. In CamelCase. More info:
     https://git.k8s.io/community/contributors/devel/api-conventions.md#types-kinds

   metadata     <Object>
     Standard object's metadata. More info:
     https://git.k8s.io/community/contributors/devel/api-conventions.md#metadata

   spec <Object>
     Specification of the desired behavior of the pod. More info:
     https://git.k8s.io/community/contributors/devel/api-conventions.md#spec-and-status

   status       <Object>
     Most recently observed status of the pod. This data may not be up to date.
     Populated by the system. Read-only. More info:
     https://git.k8s.io/community/contributors/devel/api-conventions.md#spec-and-status

[centos@k8s-master ~]$

```

以上Description是对资源对象的简要描述,`Fields`则是对所有字段的描述

## 列出所有api字段

> 通过以上我们能感觉到,以上好像并没有罗列出所有的api字段,实际上以上列出的仅是一级字段,一级字段可能还包含二级的,三级的字段,想要罗列出所有的字段,可以加上`--recursive`来列出所有可能的字段

```text
[centos@k8s-master ~]$ kubectl explain svc --recursive
KIND:     Service
VERSION:  v1

DESCRIPTION:
     Service is a named abstraction of software service (for example, mysql)
     consisting of local port (for example 3306) that the proxy listens on, and
     the selector that determines which pods will answer requests sent through
     the proxy.

FIELDS:
   apiVersion   <string>
   kind <string>
   metadata     <Object>
      annotations       <map[string]string>
      clusterName       <string>
      creationTimestamp <string>
      deletionGracePeriodSeconds        <integer>
      deletionTimestamp <string>
      finalizers        <[]string>
      generateName      <string>
      generation        <integer>
      initializers      <Object>
         pending        <[]Object>
            name        <string>
         result <Object>
            apiVersion  <string>
            code        <integer>
            details     <Object>
               causes   <[]Object>
                  field <string>
                  message       <string>
                  reason        <string>
               group    <string>
               kind     <string>
               name     <string>
               retryAfterSeconds        <integer>
               uid      <string>
            kind        <string>
            message     <string>
            metadata    <Object>
               continue <string>
               resourceVersion  <string>
               selfLink <string>
            reason      <string>
            status      <string>
      labels    <map[string]string>
      name      <string>
      namespace <string>
      ownerReferences   <[]Object>
         apiVersion     <string>
         blockOwnerDeletion     <boolean>
         controller     <boolean>
         kind   <string>
         name   <string>
         uid    <string>
      resourceVersion   <string>
      selfLink  <string>
      uid       <string>
   spec <Object>
      clusterIP <string>
      externalIPs       <[]string>
      externalName      <string>
      externalTrafficPolicy     <string>
      healthCheckNodePort       <integer>
      loadBalancerIP    <string>
      loadBalancerSourceRanges  <[]string>
      ports     <[]Object>
         name   <string>
         nodePort       <integer>
         port   <integer>
         protocol       <string>
         targetPort     <string>
      publishNotReadyAddresses  <boolean>
      selector  <map[string]string>
      sessionAffinity   <string>
      sessionAffinityConfig     <Object>
         clientIP       <Object>
            timeoutSeconds      <integer>
      type      <string>
   status       <Object>
      loadBalancer      <Object>
         ingress        <[]Object>
            hostname    <string>
            ip  <string>
[centos@k8s-master ~]$

```

以上输出的内容是经过格式化了的,我们可以根据缩进很容易看到某一个字段从属于关系

## 查看具体字段

通过上面`kubectl explain service --recursive`可以看到所有的api名称,但是以上仅仅是罗列了所有的api名称,如果想要知道某一个api名称的详细信息,则可以通过`kubectl explain <资源对象名称.api名称>`的方式来查看,比如以下示例可以查看到`service`下的`spec`下的`ports`字段的信息

```bash
[centos@k8s-master ~]$ kubectl explain svc.spec.ports
KIND:     Service
VERSION:  v1

RESOURCE: ports <[]Object>

DESCRIPTION:
     The list of ports that are exposed by this service. More info:
     https://kubernetes.io/docs/concepts/services-networking/service/#virtual-ips-and-service-proxies

     ServicePort contains information on service's port.

FIELDS:
   name <string>
     The name of this port within the service. This must be a DNS_LABEL. All
     ports within a ServiceSpec must have unique names. This maps to the 'Name'
     field in EndpointPort objects. Optional if only one ServicePort is defined
     on this service.

   nodePort     <integer>
     The port on each node on which this service is exposed when type=NodePort
     or LoadBalancer. Usually assigned by the system. If specified, it will be
     allocated to the service if unused or else creation of the service will
     fail. Default is to auto-allocate a port if the ServiceType of this Service
     requires one. More info:
     https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport

   port <integer> -required-
     The port that will be exposed by this service.

   protocol     <string>
     The IP protocol for this port. Supports "TCP", "UDP", and "SCTP". Default
     is TCP.

   targetPort   <string>
     Number or name of the port to access on the pods targeted by the service.
     Number must be in the range 1 to 65535. Name must be an IANA_SVC_NAME. If
     this is a string, it will be looked up as a named port in the target Pod's
     container ports. If this is not specified, the value of the 'port' field is
     used (an identity map). This field is ignored for services with
     clusterIP=None, and should be omitted or set equal to the 'port' field.
     More info:
     https://kubernetes.io/docs/concepts/services-networking/service/#defining-a-service
```

# References

- 原文 [kubectl技巧之查看资源列表,资源版本和资源schema配置](https://www.cnblogs.com/tylerzhou/p/11043285.html), [周国通](https://www.cnblogs.com/tylerzhou/)

