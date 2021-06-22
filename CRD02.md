**前言**

默认读者有kubernetes基础概念的背景知识，因此基础概念例如有状态、pod、Replica Sets、Deployments、statefulsets等不在此文详细阐述。 可以看我之前的一些关于kubernetes的文章：

[kubernetes 1.3管中窥豹- RS（Replica Sets）](https://sq.163yun.com/blog/article/174654699965485056)

[kubernetes pod&container生命周期浅析](https://sq.163yun.com/blog/article/173143805960052736)

[kubernetes1.6管中窥豹-StatefulSets概念、约束、原理及实例](https://sq.163yun.com/blog/article/173570389082607616)

本文主要介绍一个1.7+以后出现的新概念CRD（CustomResourceDefinition ）的概念、使用场景及实例。

| CRD历史来源 |                                          |
| ----------- | ---------------------------------------- |
| k8s1.6~1.7  | TRP（CRD的前身）：Third Party Resource。 |
| k8s1.7+     | CRD： Custom Resource Definition。       |



**CRD的概念**

Custom resources：是对K8S API的扩展，代表了一个特定的kubetnetes的定制化安装。在一个运行中的集群中，自定义资源可以动态注册到集群中。注册完毕以后，用户可以通过kubelet创建和访问这个自定义的对象，类似于操作pod一样。

Custom controllers：Custom resources可以让用户简单的存储和获取结构化数据。只有结合控制器才能变成一个真正的declarative API（被声明过的API）。控制器可以把资源更新成用户想要的状态，并且通过一系列操作维护和变更状态。定制化控制器是用户可以在运行中的集群内部署和更新的一个控制器，它独立于集群本身的生命周期。 定制化控制器可以和任何一种资源一起工作，当和定制化资源结合使用时尤其有效。

Operator模式 是一个customer controllers和Custom resources结合的例子。它可以允许开发者将特殊应用编码至kubernetes的扩展API内。

如何添加一个Custom resources到我的kubernetes集群呢？ kubernetes提供了两种方式将Custom resources添加到集群。

**1.** Custom Resource Definitions (CRDs)：更易用、不需要编码。但是缺乏灵活性。

**2.** API Aggregation：需要编码，允许通过聚合层的方式提供更加定制化的实现。

本文重点讲解Custom Resource Definitions (CRD）的使用方法。



**CRD使用场景**

假设需要在原生kubernetes完成一些额外的功能开发，而且想使用restful风格的API 去调用这个功能。比如像开发容器保存镜像或者重启镜像等操作（一般都通过命令发送给容器内部完成）。就可以在原生kubernetes上增加一个CRD，命名为task。通过task触发一些逻辑。当使用apiserver下发这个task创建时，Custom controllers watch到这个task就开始处理相关业务逻辑，然后通过agent调用docker命令完成容器保存镜像功能并且重启容器。当Custom controllers watch到pod重新运行的时候，本次操作完成。 流程图如下：





**CRD使用示例**

如果想要完成上述场景的功能，需要定义一个CRD，并且创建出来。 举例如下，以下是一个可以在1.9集群环境中创建成功的crd yaml文件。

```
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  # name must match the spec fields below, and be in the form: <plural>.<group>
  name: tasks.163yun.com
spec:
  # group name to use for REST API: /apis/<group>/<version>
  group: 163yun.com
  # version name to use for REST API: /apis/<group>/<version>
  version: v1
  # either Namespaced or Cluster
  scope: Namespaced
  names:
    # plural name to be used in the URL: /apis/<group>/<version>/<plural>
    plural: tasks
    # singular name to be used as an alias on the CLI and for display
    singular: task
    # kind is normally the CamelCased singular type. Your resource manifests use this.
    kind: Task
    # shortNames allow shorter string to match your resource on the CLI
    shortNames:
    - task
```

其中几个关键要素是name、group、scope以及names里面的一些定义。

name：用于定义CRD的名字，后缀需要跟group一致，例如tasks.163yun.com,前缀需要跟names中的plural一致。

group以及version用于标识restAPI：即/apis//。 上文中的接口前面一部分就是/apis/163yun.com/v1

scope: 表明作用于，可以是基于namespace的，也可以是基于集群的。 如果是基于namespace的。则API格式为：/apis/{group}/v1/namespaces/{namespace}/{spec.names.plural}/… 如果是基于cluster的。则API格式为：/apis/{group}/v1/{spec.names.plural}/… 上文创建的CRD的API则为：/apis/163yun.com/v1/namespaces/{namespace}/tasks

names：描述了一些自定义资源的名字以及类型的名字（重点是plural定义以及kind定义，因为会在url或者查询资源中用的到）。

当以上面的模板创建出来一个CRD后，可以查到crd的信息：

```
root@hzyq-k8s-qa-16-master:~/cxq# kubectl  get CustomResourceDefinition tasks.163yun.com   -o yaml 
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  creationTimestamp: 2018-03-16T06:17:03Z
  generation: 1
  name: tasks.163yun.com
  resourceVersion: "35442027"
  selfLink: /apis/apiextensions.k8s.io/v1beta1/customresourcedefinitions/tasks.163yun.com
  uid: a538030c-28e1-11e8-b047-fa163ef9812d
spec:
  group: 163yun.com
  names:
    kind: Task
    listKind: TaskList
    plural: tasks
    shortNames:
    - task
    singular: task
  scope: Namespaced
  version: v1
status:
  acceptedNames:
    kind: Task
    listKind: TaskList
    plural: tasks
    shortNames:
    - task
    singular: task
  conditions:
  - lastTransitionTime: 2018-03-16T06:17:03Z
    message: no conflicts found
    reason: NoConflicts
    status: "True"
    type: NamesAccepted
  - lastTransitionTime: 2018-03-16T06:17:03Z
    message: the initial names have been accepted
    reason: InitialNamesAccepted
    status: "True"
    type: Established
```

这时查询crd中定义的task类型的资源，是空的：

```
root@hzyq-k8s-qa-16-master:~/cxq# kubectl  get  task   -o yaml 
apiVersion: v1
items: []
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
```

这时需要自定义一个task类型的模板，创建，以使得task可以触发自定义的一些事件

```
root@hzyq-k8s-qa-16-master:~/cxq# cat cxq-task.yaml 
apiVersion: "163yun.com/v1"
kind: Task
metadata:
  name: cxq-nce-task
spec:
  cronSpec: "* * * * /5"
  image: my-awesome-cron-image
root@hzyq-k8s-qa-16-master:~/cxq# kubectl create -f cxq-task.yaml 
task "cxq-nce-task" created
root@hzyq-k8s-qa-16-master:~/cxq#  kubectl get task -o yaml
apiVersion: v1
items:
- apiVersion: 163yun.com/v1
  kind: Task
  metadata:
    clusterName: ""
    creationTimestamp: 2018-03-16T06:43:10Z
    labels: {}
    name: cxq-nce-task
    namespace: default
    resourceVersion: "35444352"
    selfLink: /apis/163yun.com/v1/namespaces/default/tasks/cxq-nce-task
    uid: 4ad237aa-28e5-11e8-b047-fa163ef9812d
  spec:
    cronSpec: '* * * * /5'
    image: my-awesome-cron-image
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
```

注意由于CRD和自定义的Task资源本身是namespace无关的，因此对所有namespace都可见。如果指定了scope为namespace。那么创建的自定义task也需要指定namespace，如果不指定的话，系统会自动将task分配到default 的namespace下。

这时可以通过task的url去对自定义的task资源进行操作，或者watch。结合上一节的流程图来完成实现自定义功能的目的。以下是调用watch接口对task资源进行长连接监控（变化）。



```
root@hzyq-k8s-qa-16-master:~/cxq# curl http://10.180.156.79:8080/apis/163yun.com/v1/namespaces/default/tasks?watch=true
{"type":"ADDED","object":{"apiVersion":"163yun.com/v1","kind":"Task","metadata":{"clusterName":"","creationTimestamp":"2018-03-16T06:43:10Z","labels":{},"name":"cxq-nce-task","namespace":"default","resourceVersion":"35444352","selfLink":"/apis/163yun.com/v1/namespaces/default/tasks/cxq-nce-task","uid":"4ad237aa-28e5-11e8-b047-fa163ef9812d"},"spec":{"cronSpec":"* * * * /5","image":"my-awesome-cron-image"}}}
```

**总结**

CRD提供了一种无须编码就可以扩展原生kubenetes API接口的方式。很适合在云应用中扩展kubernetes的自定义接口和功能。在一些无须太多定制和灵活的开发场景下，CRD的方式已经足够使用。如果想更为灵活的添加逻辑就需要参考API Aggregation的使用了。
