# API 扩展

Kubernetes 的架构非常灵活，提供了从 API、认证授权、准入控制、网络、存储、运行时以及云平台等一系列的[扩展机制](https://kubernetes.io/docs/concepts/extend-kubernetes/extend-cluster/)，方便用户无侵入的扩展集群的功能。

从 API 的角度来说，可以通过 Aggregation 和 CustomResourceDefinition（CRD） 等扩展 Kubernetes API。

- API Aggregation 允许在不修改 Kubernetes 核心代码的同时将第三方服务注册到 Kubernetes API 中，这样就可以通过 Kubernetes API 来访问外部服务。
- CustomResourceDefinition 则可以在集群中新增资源对象，并可以与已有资源对象（如 Pod、Deployment 等）相同的方式来管理它们。

CRD 相比 Aggregation 更易用，两者对比如下

| CRDs                                                         | Aggregated API                                               |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| 无需编程即可使用 CRD 管理资源                                | 需要使用 Go 来构建 Aggregated APIserver                      |
| 不需要运行额外服务，但一般需要一个 CRD 控制器同步和管理这些资源 | 需要独立的第三方服务                                         |
| 任何缺陷都会在 Kubernetes 核心中修复                         | 可能需要定期从 Kubernetes 社区同步缺陷修复方法并重新构建 Aggregated APIserver. |
| 无需额外管理版本                                             | 需要第三方服务来管理版本                                     |

更多的特性对比

| Feature               | Description                                                  | CRDs                                                         | Aggregated API                   |
| --------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | -------------------------------- |
| Validation            | Help users prevent errors and allow you to evolve your API independently of your clients. These features are most useful when there are many clients who can’t all update at the same time. | Yes. Most validation can be specified in the CRD using [OpenAPI v3.0 validation](https://kubernetes.io/docs/tasks/access-kubernetes-api/extend-api-custom-resource-definitions/#validation). Any other validations supported by addition of a Validating Webhook. | Yes, arbitrary validation checks |
| Defaulting            | See above                                                    | Yes, via a Mutating Webhook; Planned, via CRD OpenAPI schema. | Yes                              |
| Multi-versioning      | Allows serving the same object through two API versions. Can help ease API changes like renaming fields. Less important if you control your client versions. | No, but planned                                              | Yes                              |
| Custom Storage        | If you need storage with a different performance mode (for example, time-series database instead of key-value store) or isolation for security (for example, encryption secrets or different | No                                                           | Yes                              |
| Custom Business Logic | Perform arbitrary checks or actions when creating, reading, updating or deleting an object | Yes, using Webhooks.                                         | Yes                              |
| Scale Subresource     | Allows systems like HorizontalPodAutoscaler and PodDisruptionBudget interact with your new resource | [Yes](https://kubernetes.io/docs/tasks/access-kubernetes-api/extend-api-custom-resource-definitions/#scale-subresource) | Yes                              |
| Status Subresource    | Finer-grained access control: user writes spec section, controller writes status section.Allows incrementing object Generation on custom resource data mutation (requires separate spec and status sections in the resource) | [Yes](https://kubernetes.io/docs/tasks/access-kubernetes-api/extend-api-custom-resource-definitions/#status-subresource) | Yes                              |
| Other Subresources    | Add operations other than CRUD, such as “logs” or “exec”.    | No                                                           | Yes                              |
| strategic-merge-patch | The new endpoints support PATCH with `Content-Type: application/strategic-merge-patch+json`. Useful for updating objects that may be modified both locally, and by the server. For more information, see [“Update API Objects in Place Using kubectl patch”](https://kubernetes.io/docs/tasks/run-application/update-api-object-kubectl-patch/) | No, but similar functionality planned                        | Yes                              |
| Protocol Buffers      | The new resource supports clients that want to use Protocol Buffers | No                                                           | Yes                              |
| OpenAPI Schema        | Is there an OpenAPI (swagger) schema for the types that can be dynamically fetched from the server? Is the user protected from misspelling field names by ensuring only allowed fields are set? Are types enforced (in other words, don’t put an `int` in a `string` field?) | No, but planned                                              | Yes                              |

# Aggregation Layer

API Aggregation 允许在不修改 Kubernetes 核心代码的同时扩展 Kubernetes API，即将第三方服务注册到 Kubernetes API 中，这样就可以通过 Kubernetes API 来访问外部服务。

> 备注：另外一种扩展 Kubernetes API 的方法是使用 [CustomResourceDefinition (CRD)](https://feisky.gitbooks.io/kubernetes/content/concepts/customresourcedefinition.html)。

## 何时使用 Aggregation

| 满足以下条件时使用 API Aggregation                           | 满足以下条件时使用独立 API                                   |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| Your API is [Declarative](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#declarative-apis). | Your API does not fit the [Declarative](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#declarative-apis) model. |
| You want your new types to be readable and writable using `kubectl`. | `kubectl` support is not required                            |
| You want to view your new types in a Kubernetes UI, such as dashboard, alongside built-in types. | Kubernetes UI support is not required.                       |
| You are developing a new API.                                | You already have a program that serves your API and works well. |
| You are willing to accept the format restriction that Kubernetes puts on REST resource paths, such as API Groups and Namespaces. (See the [API Overview](https://kubernetes.io/docs/concepts/overview/kubernetes-api/).) | You need to have specific REST paths to be compatible with an already defined REST API. |
| Your resources are naturally scoped to a cluster or to namespaces of a cluster. | Cluster or namespace scoped resources are a poor fit; you need control over the specifics of resource paths. |
| You want to reuse [Kubernetes API support features](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#common-features). | You don’t need those features.                               |

## 开启 API Aggregation

kube-apiserver 增加以下配置

```sh
--requestheader-client-ca-file=<path to aggregator CA cert>
--requestheader-allowed-names=aggregator
--requestheader-extra-headers-prefix=X-Remote-Extra-
--requestheader-group-headers=X-Remote-Group
--requestheader-username-headers=X-Remote-User
--proxy-client-cert-file=<path to aggregator proxy cert>
--proxy-client-key-file=<path to aggregator proxy key>
```

如果 `kube-proxy` 没有在 Master 上面运行，还需要配置

```sh
--enable-aggregator-routing=true
```

## 创建扩展 API

1. 确保开启 APIService API（默认开启，可用 `kubectl get apiservice` 命令验证）
2. 创建 RBAC 规则
3. 创建一个 namespace，用来运行扩展的 API 服务
4. 创建 CA 和证书，用于 https
5. 创建一个存储证书的 secret
6. 创建一个部署扩展 API 服务的 deployment，并使用上一步的 secret 配置证书，开启 https 服务
7. 创建一个 ClusterRole 和 ClusterRoleBinding
8. 创建一个非 namespace 的 apiservice，注意设置 `spec.caBundle`
9. 运行 `kubectl get <resource-name>`，正常应该返回 `No resources found.`

可以使用 [apiserver-builder](https://github.com/kubernetes-incubator/apiserver-builder) 工具自动化上面的步骤。

```sh
# 初始化项目
$ cd GOPATH/src/github.com/my-org/my-project
$ apiserver-boot init repo --domain <your-domain>
$ apiserver-boot init glide

# 创建资源
$ apiserver-boot create group version resource --group <group> --version <version> --kind <Kind>

# 编译
$ apiserver-boot build executables
$ apiserver-boot build docs

# 本地运行
$ apiserver-boot run local

# 集群运行
$ apiserver-boot run in-cluster --name nameofservicetorun --namespace default --image gcr.io/myrepo/myimage:mytag
$ kubectl create -f sample/<type>.yaml
```

## 示例

见 [sample-apiserver](https://github.com/kubernetes/sample-apiserver) 和 [apiserver-builder/example](https://github.com/kubernetes-incubator/apiserver-builder/tree/master/example)。
