# kube-apiserver源码分析

kube-apiserver 共由 3 个组件构成（Aggregator. KubeAPIServer. APIExtensionServer），这些组件依次通过 Delegation 处理请求：

- Aggregator：暴露的功能类似于一个七层负载均衡，将来自用户的请求拦截转发给其他服务器，并且负责整个 APIServer 的 Discovery 功能；也负责处理ApiService，注册对应的扩展api。
- KubeAPIServer ：负责对请求的一些通用处理，认证. 鉴权等，以及处理各个内建资源的 REST 服务；
- APIExtensionServer：主要处理 CustomResourceDefinition（CRD）和 CustomResource（CR）的 REST 请求，也是 Delegation 的最后一环，如果对应 CR 不能被处理的话则会返回 404。

## kube-apiserver启动流程

Apiserver通过`Run`方法启动, 主要逻辑为：

1. 调用`CreateServerChain`构建服务调用链并判断是否启动非安全的`httpserver`，`httpserver`链中包含 apiserver要启动的三个server，以及为每个server注册对应资源的路由；
2. 调用`server.PrepareRun`进行服务运行前的准备，该方法主要完成了健康检查. 存活检查和OpenAPI路由的注册工作；
3. 调用`prepared.Run`启动server；


