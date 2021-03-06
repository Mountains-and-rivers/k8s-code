# kube-apiserver源码分析-01

### 代码目录结构

```
|--api  // 存放api规范相关的文档

|------api-rules  //已经存在的违反Api规范的api

|------openapi-spec  //OpenApi规范

|--build // 构建和测试脚本

|------run.sh  //在容器中运行该脚本，后面可接多个命令：make, make cross 等

|------copy-output.sh  //把容器中_output/dockerized/bin目录下的文件拷贝到本地目录

|------make-clean.sh  //清理容器中和本地的_output目录

|------shell.sh  // 容器中启动一个shell终端

|------......

|--cluster  // 自动创建和配置kubernetes集群的脚本，包括networking, DNS, nodes等

|--cmd  // 内部包含各个组件的入口，具体核心的实现部分在pkg目录下

|--hack  // 编译、构建及校验的工具类

|--logo // kubernetes的logo

|--pkg // 主要代码存放类，后面会详细补充该目录下内容

|------kubeapiserver

|------kubectl

|------kubelet

|------proxy

|------registry

|------scheduler

|------security

|------watch

|------......

|--plugin

|------pkg/admission  //认证

|------pkg/auth  //鉴权

|--staging  // 这里的代码都存放在独立的repo中，以引用包的方式添加到项目中

|------k8s.io/api

|------k8s.io/apiextensions-apiserver

|------k8s.io/apimachinery

|------k8s.io/apiserver

|------k8s.io/client-go

|------......

|--test  //测试代码

|--third_party  //第三方代码，protobuf、golang-reflect等

|--translations  //不同国家的语言包，使用poedit查看及编辑
```

kube-apiserver 是 kubernetes 中与 etcd 直接交互的一个组件，其控制着 kubernetes 中核心资源的变化。它主要提供了以下几个功能：

- 提供 [Kubernetes API](https://kubernetes.io/docs/concepts/overview/kubernetes-api/)，包括认证授权、数据校验以及集群状态变更等，供客户端及其他组件调用；
- 代理集群中的一些附加组件组件，如 Kubernetes UI、metrics-server、npd 等；
- 创建 kubernetes 服务，即提供 apiserver 的 Service，kubernetes Service；
- 资源在不同版本之间的转换；

### kube-apiserver 处理流程

kube-apiserver 主要通过对外提供 API 的方式与其他组件进行交互，可以调用 kube-apiserver 的接口 `$ curl -k https://<masterIP>:6443`或者通过其提供的 **swagger-ui** 获取到，其主要有以下三种 API：

- core group：主要在 `/api/v1` 下；
- named groups：其 path 为 `/apis/$NAME/$VERSION`；
- 暴露系统状态的一些 API：如`/metrics` 、`/healthz` 等；

API 的 URL 大致以 `/apis/group/version/namespaces/my-ns/myresource` 组成，其中 API 的结构大致如下图所示：

![image](https://github.com/Mountains-and-rivers/k8s-code/blob/main/images/api-server-01.png)

了解了 kube-apiserver 的 API 后，下面会介绍 kube-apiserver 如何处理一个 API 请求，一个请求完整的流程如下图所示：

![image](https://github.com/Mountains-and-rivers/k8s-code/blob/main/images/api-server-02.png)

此处以一次 POST 请求示例说明，当请求到达 kube-apiserver 时，kube-apiserver 首先会执行在 http filter chain 中注册的过滤器链，该过滤器对其执行一系列过滤操作，主要有认证、鉴权等检查操作。当 filter chain 处理完成后，请求会通过 route 进入到对应的 handler 中，handler 中的操作主要是与 etcd 的交互，在 handler 中的主要的操作如下所示：

![image](https://github.com/Mountains-and-rivers/k8s-code/blob/main/images/api-server-03.png)

![image](https://github.com/Mountains-and-rivers/k8s-code/blob/main/images/api-server-04.png)

![image](https://github.com/Mountains-and-rivers/k8s-code/blob/main/images/api-server-05.png)

**Decoder**

kubernetes 中的多数 resource 都会有一个 `internal version`，因为在整个开发过程中一个 resource 可能会对应多个 version，比如 deployment 会有 `extensions/v1beta1`，`apps/v1`。 为了避免出现问题，kube-apiserver 必须要知道如何在每一对版本之间进行转换（例如，v1⇔v1alpha1，v1⇔v1beta1，v1beta1⇔v1alpha1），因此其使用了一个特殊的`internal version`，`internal version` 作为一个通用的 version 会包含所有 version 的字段，它具有所有 version 的功能。 Decoder 会首先把 creater object 转换到 `internal version`，然后将其转换为 `storage version`，`storage version` 是在 etcd 中存储时的另一个 version。

在解码时，首先从 HTTP path 中获取期待的 version，然后使用 scheme 以正确的 version 创建一个与之匹配的空对象，并使用 JSON 或 protobuf 解码器进行转换，在转换的第一步中，如果用户省略了某些字段，Decoder 会把其设置为默认值。

**Admission**

在解码完成后，需要通过验证集群的全局约束来检查是否可以创建或更新对象，并根据集群配置设置默认值。在 `k8s.io/kubernetes/plugin/pkg/admission` 目录下可以看到 kube-apiserver 可以使用的所有全局约束插件，kube-apiserver 在启动时通过设置 `--enable-admission-plugins` 参数来开启需要使用的插件，通过 `ValidatingAdmissionWebhook` 或 `MutatingAdmissionWebhook` 添加的插件也都会在此处进行工作。

**Validation**

主要检查 object 中字段的合法性。

在 handler 中执行完以上操作后最后会执行与 etcd 相关的操作，POST 操作会将数据写入到 etcd 中，以上在 handler 中的主要处理流程如下所示：

```
v1beta1 ⇒internal⇒|⇒|⇒  v1  ⇒ json/yaml ⇒ etcd 
```

kube-apiserver 共由 3 个组件构成（Aggregator. KubeAPIServer. APIExtensionServer），这些组件依次通过 Delegation 处理请求：

- Aggregator：暴露的功能类似于一个七层负载均衡，将来自用户的请求拦截转发给其他服务器，并且负责整个 APIServer 的 Discovery 功能；也负责处理ApiService，注册对应的扩展api。 CRD 能够自动注册到集群中。

- KubeAPIServer ：负责对请求的一些通用处理，认证. 鉴权等，以及处理各个内建资源的 REST 服务；

- APIExtensionServer：主要处理 CustomResourceDefinition（CRD）和 CustomResource（CR）的 REST 请求，也是 Delegation 的最后一环，如果对应 CR 不能被处理的话则会返回 404。

  ## go接口

  接口继承

  ```
  package main
  
  import "fmt"
  
  type Humaner interface { //子集
  	sayhi()
  }
  
  type Personer interface { //超集
  	Humaner //匿名字段，继承了sayhi()
  	sing(lrc string)
  }
  
  type Student struct {
  	name string
  	id   int
  }
  
  //Student实现了sayhi()
  func (tmp *Student) sayhi() {
  	fmt.Printf("Student[%s, %d] sayhi\n", tmp.name, tmp.id)
  }
  
  func (tmp *Student) sing(lrc string) {
  	fmt.Println("Student在唱着：", lrc)
  }
  
  func main() {
  	//定义一个接口类型的变量
  	var i Personer
  	s := &Student{"mike", 666}
  	i = s
  
  	i.sayhi() //继承过来的方法
  	i.sing("学生哥")
  }
  ```

  接口转换

  ```
  package main
  
  import "fmt"
  
  type Humaner interface { //子集
  	sayhi()
  }
  
  type Personer interface { //超集
  	Humaner //匿名字段，继承了sayhi()
  	sing(lrc string)
  }
  
  type Student struct {
  	name string
  	id   int
  }
  
  //Student实现了sayhi()
  func (tmp *Student) sayhi() {
  	fmt.Printf("Student[%s, %d] sayhi\n", tmp.name, tmp.id)
  }
  
  func (tmp *Student) sing(lrc string) {
  	fmt.Println("Student在唱着：", lrc)
  }
  
  func main() {
  	//超集可以转换为子集，反过来不可以
  	var iPro Personer //超集
  	iPro = &Student{"mike", 666}
  
  	var i Humaner //子集
  
  	//iPro = i //err
  	i = iPro //可以，超集可以转换为子集
  	i.sayhi()
  
  }
  ```

  

## kube-apiserver启动流程

Apiserver通过`Run`方法启动, 主要逻辑为：

1. 调用`CreateServerChain`构建服务调用链并判断是否启动非安全的`httpserver`，`httpserver`链中包含 apiserver要启动的三个server，以及为每个server注册对应资源的路由；

2. 调用`server.PrepareRun`进行服务运行前的准备，该方法主要完成了健康检查. 存活检查和OpenAPI路由的注册工作；

3. 调用`prepared.Run`启动server；

   ```
   // Run runs the specified APIServer.  This should never exit.
   func Run(completeOptions completedServerRunOptions, stopCh <-chan struct{}) error {
   	// To help debugging, immediately log version
   	klog.Infof("Version: %+v", version.Get())
   // 创建调用链
   	server, err := CreateServerChain(completeOptions, stopCh)
   	if err != nil {
   		return err
   	}
     进行一些准备工作， 注册一些hander，执行hook等
   	prepared, err := server.PrepareRun()
   	if err != nil {
   		return err
   	}
    // 开始执行
   	return prepared.Run(stopCh)
   }
   ```

   执行具体的`Run`方法

   ```
   // Run spawns the secure http server. It only returns if stopCh is closed
   // or the secure port cannot be listened on initially.
   func (s preparedGenericAPIServer) Run(stopCh <-chan struct{}) error {
   	delayedStopCh := make(chan struct{})
   
   	go func() {
   		defer close(delayedStopCh)
   
   		<-stopCh
   
   		// As soon as shutdown is initiated, /readyz should start returning failure.		
   		// This gives the load balancer a window defined by ShutdownDelayDuration to detect that /readyz is red
   		// and stop sending traffic to this server.
   		// 当终止时，关闭readiness
   		close(s.readinessStopCh)
   
   		time.Sleep(s.ShutdownDelayDuration)
   	}()
   
   	// close socket after delayed stopCh
   	// 执行非阻塞Run
   	stoppedCh, err := s.NonBlockingRun(delayedStopCh)
   	if err != nil {
   		return err
   	}
   
   	<-stopCh
   
   	// run shutdown hooks directly. This includes deregistering from the kubernetes endpoint in case of kube-apiserver.
   	// 关闭前执行一些hook操作
   	err = s.RunPreShutdownHooks()
   	if err != nil {
   		return err
   	}
   
   	// wait for the delayed stopCh before closing the handler chain (it rejects everything after Wait has been called).
   	<-delayedStopCh
   	// wait for stoppedCh that is closed when the graceful termination (server.Shutdown) is finished.
   	<-stoppedCh
   
   	// Wait for all requests to finish, which are bounded by the RequestTimeout variable.
   	s.HandlerChainWaitGroup.Wait()
   
   	return nil
   }
   ```

   执行`NonBlockingRun`
   k8s.io/kubernetes/staging/src/k8s.io/apiserver/pkg/server/genericapiserver.go

   ```
   // NonBlockingRun spawns the secure http server. An error is
   // returned if the secure port cannot be listened on.
   // The returned channel is closed when the (asynchronous) termination is finished.
   func (s preparedGenericAPIServer) NonBlockingRun(stopCh <-chan struct{}) (<-chan struct{}, error) {
   	// Use an stop channel to allow graceful shutdown without dropping audit events
   	// after http server shutdown.
   	auditStopCh := make(chan struct{})
   
   	// Start the audit backend before any request comes in. This means we must call Backend.Run
   	// before http server start serving. Otherwise the Backend.ProcessEvents call might block.
   	// 1. 判断是否要启动审计日志
   	if s.AuditBackend != nil {
   		if err := s.AuditBackend.Run(auditStopCh); err != nil {
   			return nil, fmt.Errorf("failed to run the audit backend: %v", err)
   		}
   	}
   
   	// Use an internal stop channel to allow cleanup of the listeners on error.
   	 // 2. 启动 https server
   	internalStopCh := make(chan struct{})
   	var stoppedCh <-chan struct{}
   	if s.SecureServingInfo != nil && s.Handler != nil {
   		var err error
   		stoppedCh, err = s.SecureServingInfo.Serve(s.Handler, s.ShutdownTimeout, internalStopCh)
   		if err != nil {
   			close(internalStopCh)
   			close(auditStopCh)
   			return nil, err
   		}
   	}
   
   	// Now that listener have bound successfully, it is the
   	// responsibility of the caller to close the provided channel to
   	// ensure cleanup.
   	go func() {
   		<-stopCh
   		close(internalStopCh)
   		if stoppedCh != nil {
   			<-stoppedCh
   		}
   		s.HandlerChainWaitGroup.Wait()
   		close(auditStopCh)
   	}()
      // 3. 执行 postStartHooks
   	s.RunPostStartHooks(stopCh)
      // 4. 向 systemd 发送 ready 信号
   	if _, err := systemd.SdNotify(true, "READY=1\n"); err != nil {
   		klog.Errorf("Unable to send systemd daemon successful start message: %v\n", err)
   	}
   
   	return stoppedCh, nil
   }
   ```

   ## 调用链分析

   上一节简单分析了Apiserver的启动流程，通过初始化各种配置，封装调用链，启动Server。这节主要分析调用链。

   初始化阶段, 通过`CreateServerChain`创建调用链， 代码在server.go

   ```
   // CreateServerChain creates the apiservers connected via delegation.
   func CreateServerChain(completedOptions completedServerRunOptions, stopCh <-chan struct{}) (*aggregatorapiserver.APIAggregator, error) {
   
     // nodetunneler与node通信，proxy实现代理功能，转发请求给其他apiservice
     // apiserver到cluster的通信可以通过三种方法
     // apiserver到kubelet的endpoint，用于logs功能，exec功能，port-forward功能
     // HTTP连接，即使可以用HTTPS也不做任何其他校验，并不安全
     // ssh tunnel，不推荐使用
     
   	nodeTunneler, proxyTransport, err := CreateNodeDialer(completedOptions)
   	if err != nil {
   		return nil, err
   	}
      // 1. 为 kubeAPIServer 创建配置
   	kubeAPIServerConfig, insecureServingInfo, serviceResolver, pluginInitializer, err := CreateKubeAPIServerConfig(completedOptions, nodeTunneler, proxyTransport)
   	if err != nil {
   		return nil, err
   	}
      // 2. 判断是否配置了 APIExtensionsServer，创建 apiExtensionsConfig 
   	// If additional API servers are added, they should be gated.
   	apiExtensionsConfig, err := createAPIExtensionsConfig(*kubeAPIServerConfig.GenericConfig, kubeAPIServerConfig.ExtraConfig.VersionedInformers, pluginInitializer, completedOptions.ServerRunOptions, completedOptions.MasterCount,
   		serviceResolver, webhook.NewDefaultAuthenticationInfoResolverWrapper(proxyTransport, kubeAPIServerConfig.GenericConfig.EgressSelector, kubeAPIServerConfig.GenericConfig.LoopbackClientConfig))
   	if err != nil {
   		return nil, err
   	}
   	// 3. 初始化 APIExtensionsServer, 通过一个空的delegate初始化
   	apiExtensionsServer, err := createAPIExtensionsServer(apiExtensionsConfig, genericapiserver.NewEmptyDelegate())
   	if err != nil {
   		return nil, err
   	}
     // 4. 初始化 KubeAPIServer
   	kubeAPIServer, err := CreateKubeAPIServer(kubeAPIServerConfig, apiExtensionsServer.GenericAPIServer)
   	if err != nil {
   		return nil, err
   	}
   
   	// aggregator comes last in the chain
   	// 5. 创建 AggregatorConfig
   	aggregatorConfig, err := createAggregatorConfig(*kubeAPIServerConfig.GenericConfig, completedOptions.ServerRunOptions, kubeAPIServerConfig.ExtraConfig.VersionedInformers, serviceResolver, proxyTransport, pluginInitializer)
   	if err != nil {
   		return nil, err
   	}
   	// 6. 初始化 AggregatorServer
   	aggregatorServer, err := createAggregatorServer(aggregatorConfig, kubeAPIServer.GenericAPIServer, apiExtensionsServer.Informers)
   	if err != nil {
   		// we don't need special handling for innerStopCh because the aggregator server doesn't create any go routines
   		return nil, err
   	}
      // 7. 判断是否启动非安全端口的 http server
   	if insecureServingInfo != nil {
   		insecureHandlerChain := kubeserver.BuildInsecureHandlerChain(aggregatorServer.GenericAPIServer.UnprotectedHandler(), kubeAPIServerConfig.GenericConfig)
   		if err := insecureServingInfo.Serve(insecureHandlerChain, kubeAPIServerConfig.GenericConfig.RequestTimeout, stopCh); err != nil {
   			return nil, err
   		}
   	}
   
   	return aggregatorServer, nil
   }
   ```

   创建过程主要有以下步骤：

   1. 根据配置构造apiserver的配置，调用方法`CreateKubeAPIServerConfig`
   2. 根据配置构造扩展的apiserver的配置，调用方法为`createAPIExtensionsConfig`
   3. 创建server，包括扩展的apiserver和原生的apiserver，调用方法为`createAPIExtensionsServer`和`CreateKubeAPIServer`。主要就是将各个handler的路由方法注册到Container中去，完全遵循go-restful的设计模式，即将处理方法注册到Route中去，同一个根路径下的Route注册到WebService中去，WebService注册到Container中，Container负责分发。访问的过程为`Container-->WebService-->Route`
   4. 聚合server的配置和和创建。主要就是将原生的apiserver和扩展的apiserver的访问进行整合，添加后续的一些处理接口。调用方法为`createAggregatorConfig`和`createAggregatorServer`
   5. 创建完成，返回配置的server信息

   以上几个步骤，最核心的就是apiserver如何创建，即如何按照go-restful的模式，添加路由和相应的处理方法。

   ### 配置初始化

   先看apiserver配置的创建`CreateKubeAPIServerConfig->buildGenericConfig->genericapiserver.NewConfig`

   ```
   // BuildGenericConfig takes the master server options and produces the genericapiserver.Config associated with it
   func buildGenericConfig(
   	s *options.ServerRunOptions,
   	proxyTransport *http.Transport,
   ) (
   	genericConfig *genericapiserver.Config,
   	versionedInformers clientgoinformers.SharedInformerFactory,
   	insecureServingInfo *genericapiserver.DeprecatedInsecureServingInfo,
   	serviceResolver aggregatorapiserver.ServiceResolver,
   	pluginInitializers []admission.PluginInitializer,
   	admissionPostStartHook genericapiserver.PostStartHookFunc,
   	storageFactory *serverstorage.DefaultStorageFactory,
   	lastErr error,
   ) {
     // 创建genericConfig,其中包括DefaultBuildHandlerChain，一系列认证授权的中间件
   	genericConfig = genericapiserver.NewConfig(legacyscheme.Codecs)
   	genericConfig.MergedResourceConfig = controlplane.DefaultAPIResourceConfigSource()
     // 初始化各种配置
   	if lastErr = s.GenericServerRunOptions.ApplyTo(genericConfig); lastErr != nil {
   		return
   	}
     // 长连接请求
   	if lastErr = s.InsecureServing.ApplyTo(&insecureServingInfo, &genericConfig.LoopbackClientConfig); lastErr != nil {
   		return
   	}
   	if lastErr = s.SecureServing.ApplyTo(&genericConfig.SecureServing, &genericConfig.LoopbackClientConfig); lastErr != nil {
   		return
   	}
   	if lastErr = s.Features.ApplyTo(genericConfig); lastErr != nil {
   		return
   	}
   	if lastErr = s.APIEnablement.ApplyTo(genericConfig, controlplane.DefaultAPIResourceConfigSource(), legacyscheme.Scheme); lastErr != nil {
   		return
   	}
   	if lastErr = s.EgressSelector.ApplyTo(genericConfig); lastErr != nil {
   		return
   	}
     // 初始化storageFactory， 用来连接etcd
   	genericConfig.OpenAPIConfig = genericapiserver.DefaultOpenAPIConfig(generatedopenapi.GetOpenAPIDefinitions, openapinamer.NewDefinitionNamer(legacyscheme.Scheme, extensionsapiserver.Scheme, aggregatorscheme.Scheme))
   	genericConfig.OpenAPIConfig.Info.Title = "Kubernetes"
   	genericConfig.LongRunningFunc = filters.BasicLongRunningRequestCheck(
   		sets.NewString("watch", "proxy"),
   		sets.NewString("attach", "exec", "proxy", "log", "portforward"),
   	)
   
   	kubeVersion := version.Get()
   	genericConfig.Version = &kubeVersion
   
   	storageFactoryConfig := kubeapiserver.NewStorageFactoryConfig()
   	storageFactoryConfig.APIResourceConfig = genericConfig.MergedResourceConfig
   	completedStorageFactoryConfig, err := storageFactoryConfig.Complete(s.Etcd)
   	if err != nil {
   		lastErr = err
   		return
   	}
   	storageFactory, lastErr = completedStorageFactoryConfig.New()
   	if lastErr != nil {
   		return
   	}
   	if genericConfig.EgressSelector != nil {
   		storageFactory.StorageConfig.Transport.EgressLookup = genericConfig.EgressSelector.Lookup
   	}
   	if lastErr = s.Etcd.ApplyWithStorageFactoryTo(storageFactory, genericConfig); lastErr != nil {
   		return
   	}
   
   	// Use protobufs for self-communication.
   	// Since not every generic apiserver has to support protobufs, we
   	// cannot default to it in generic apiserver and need to explicitly
   	// set it in kube-apiserver.
   	// 内部使用protobufs通信
   	genericConfig.LoopbackClientConfig.ContentConfig.ContentType = "application/vnd.kubernetes.protobuf"
   	// Disable compression for self-communication, since we are going to be
   	// on a fast local network
   	genericConfig.LoopbackClientConfig.DisableCompression = true
   // clientset初始化
   	kubeClientConfig := genericConfig.LoopbackClientConfig
   	clientgoExternalClient, err := clientgoclientset.NewForConfig(kubeClientConfig)
   	if err != nil {
   		lastErr = fmt.Errorf("failed to create real external clientset: %v", err)
   		return
   	}
   	versionedInformers = clientgoinformers.NewSharedInformerFactory(clientgoExternalClient, 10*time.Minute)
   
   	// Authentication.ApplyTo requires already applied OpenAPIConfig and EgressSelector if present
   	// 初始化认证实例，支持多种认证方式：requestheader,token, tls等
   	if lastErr = s.Authentication.ApplyTo(&genericConfig.Authentication, genericConfig.SecureServing, genericConfig.EgressSelector, genericConfig.OpenAPIConfig, clientgoExternalClient, versionedInformers); lastErr != nil {
   		return
   	}
   // 初始化鉴权配置
   	genericConfig.Authorization.Authorizer, genericConfig.RuleResolver, err = BuildAuthorizer(s, genericConfig.EgressSelector, versionedInformers)
   	if err != nil {
   		lastErr = fmt.Errorf("invalid authorization config: %v", err)
   		return
   	}
   	if !sets.NewString(s.Authorization.Modes...).Has(modes.ModeRBAC) {
   		genericConfig.DisabledPostStartHooks.Insert(rbacrest.PostStartHookName)
   	}
   
   	lastErr = s.Audit.ApplyTo(genericConfig)
   	if lastErr != nil {
   		return
   	}
     // 初始化admission webhook的配置
   	admissionConfig := &kubeapiserveradmission.Config{
   		ExternalInformers:    versionedInformers,
   		LoopbackClientConfig: genericConfig.LoopbackClientConfig,
   		CloudConfigFile:      s.CloudProvider.CloudConfigFile,
   	}
   	serviceResolver = buildServiceResolver(s.EnableAggregatorRouting, genericConfig.LoopbackClientConfig.Host, versionedInformers)
   	// 初始化注入插件
   	pluginInitializers, admissionPostStartHook, err = admissionConfig.New(proxyTransport, genericConfig.EgressSelector, serviceResolver)
   	if err != nil {
   		lastErr = fmt.Errorf("failed to create admission plugin initializer: %v", err)
   		return
   	}
   
   	err = s.Admission.ApplyTo(
   		genericConfig,
   		versionedInformers,
   		kubeClientConfig,
   		feature.DefaultFeatureGate,
   		pluginInitializers...)
   	if err != nil {
   		lastErr = fmt.Errorf("failed to initialize admission: %v", err)
   	}
   
   	if utilfeature.DefaultFeatureGate.Enabled(genericfeatures.APIPriorityAndFairness) && s.GenericServerRunOptions.EnablePriorityAndFairness {
   		genericConfig.FlowControl = BuildPriorityAndFairness(s, clientgoExternalClient, versionedInformers)
   	}
   
   	return
   }
   ```

   ### APIExtensionsServer初始化

   `APIExtensionsServer`最先初始化，在调用链的末尾, 处理CR、CRD相关资源.

   其中包含的 controller 以及功能如下所示：

   1. openapiController：将 crd 资源的变化同步至提供的 OpenAPI 文档，可通过访问 /openapi/v2 进行查看；

   2. crdController：负责将 crd 信息注册到 apiVersions 和 apiResources 中，两者的信息可通过 $ kubectl api-versions 和 $ kubectl api-resources 查看；

   3. namingController：检查 crd obj 中是否有命名冲突，可在 crd .status.conditions 中查看；

   4. establishingController：检查 crd 是否处于正常状态，可在 crd .status.conditions 中查看；

   5. nonStructuralSchemaController：检查 crd obj 结构是否正常，可在 crd .status.conditions 中查看；

   6. apiApprovalController：检查 crd 是否遵循 kubernetes API 声明策略，可在 crd .status.conditions 中查看；

   7. finalizingController：类似于 finalizes 的功能，与 CRs 的删除有关；

      APIExtensionsServer 是最先被初始化的，在 `createAPIExtensionsServer` 中调用 `apiextensionsConfig.Complete().New` 来完成 server 的初始化，其主要逻辑为：

      1、首先调用 `c.GenericConfig.New` 按照`go-restful`的模式初始化 Container，在 `c.GenericConfig.New` 中会调用 `NewAPIServerHandler` 初始化 handler，APIServerHandler 包含了 API Server 使用的多种http.Handler 类型，包括 `go-restful` 以及 `non-go-restful`，以及在以上两者之间选择的 Director 对象，`go-restful` 用于处理已经注册的 handler，`non-go-restful` 用来处理不存在的 handler，API URI 处理的选择过程为：`FullHandlerChain-> Director ->{GoRestfulContainer， NonGoRestfulMux}`。在 `c.GenericConfig.New` 中还会调用 `installAPI`来添加包括 `/`、`/debug/*`、`/metrics`、`/version` 等路由信息。三种 server 在初始化时首先都会调用 `c.GenericConfig.New` 来初始化一个 genericServer，然后进行 API 的注册；

      2、调用 `s.GenericAPIServer.InstallAPIGroup` 在路由中注册 API Resources，此方法的调用链非常深，主要是为了将需要暴露的 API Resource 注册到 server 中，以便能通过 http 接口进行 resource 的 REST 操作，其他几种 server 在初始化时也都会执行对应的 `InstallAPI`；

      3、初始化 server 中需要使用的 controller，主要有 `openapiController`、`crdController`、`namingController`、`establishingController`、`nonStructuralSchemaController`、`apiApprovalController`、`finalizingControlle`r；

      4、将需要启动的 controller 以及 informer 添加到 PostStartHook 中；

      k8s.io/kubernetes/cmd/kube-apiserver/app/apiextensions.go

      ```
      func createAPIExtensionsServer(apiextensionsConfig *apiextensionsapiserver.Config, delegateAPIServer genericapiserver.DelegationTarget) (*apiextensionsapiserver.CustomResourceDefinitions, error) {
      	return apiextensionsConfig.Complete().New(delegateAPIServer)
      }
      ```

   ```
   k8s.io/kubernetes/staging/src/k8s.io/apiextensions-apiserver/pkg/apiserver/apiserver.go
   ```

   ```
   // New returns a new instance of CustomResourceDefinitions from the given config.
   func (c completedConfig) New(delegationTarget genericapiserver.DelegationTarget) (*CustomResourceDefinitions, error) {
     // 初始化 genericServer
   	genericServer, err := c.GenericConfig.New("apiextensions-apiserver", delegationTarget)
   	if err != nil {
   		return nil, err
   	}
   
   	s := &CustomResourceDefinitions{
   		GenericAPIServer: genericServer,
   	}
     // 初始化apigroup, 即需要暴露的api，这里extension apiserver只注册了cr于crd相关的
   	apiResourceConfig := c.GenericConfig.MergedResourceConfig
   	apiGroupInfo := genericapiserver.NewDefaultAPIGroupInfo(apiextensions.GroupName, Scheme, metav1.ParameterCodec, Codecs)
   	if apiResourceConfig.VersionEnabled(v1beta1.SchemeGroupVersion) {
   		storage := map[string]rest.Storage{}
   		// customresourcedefinitions
   		customResourceDefinitionStorage, err := customresourcedefinition.NewREST(Scheme, c.GenericConfig.RESTOptionsGetter)
   		if err != nil {
   			return nil, err
   		}
   		storage["customresourcedefinitions"] = customResourceDefinitionStorage
   		storage["customresourcedefinitions/status"] = customresourcedefinition.NewStatusREST(Scheme, customResourceDefinitionStorage)
   
   		apiGroupInfo.VersionedResourcesStorageMap[v1beta1.SchemeGroupVersion.Version] = storage
   	}
   	if apiResourceConfig.VersionEnabled(v1.SchemeGroupVersion) {
   		storage := map[string]rest.Storage{}
   		// customresourcedefinitions
   		customResourceDefintionStorage, err := customresourcedefinition.NewREST(Scheme, c.GenericConfig.RESTOptionsGetter)
   		if err != nil {
   			return nil, err
   		}
   		storage["customresourcedefinitions"] = customResourceDefintionStorage
   		storage["customresourcedefinitions/status"] = customresourcedefinition.NewStatusREST(Scheme, customResourceDefintionStorage)
   
   		apiGroupInfo.VersionedResourcesStorageMap[v1.SchemeGroupVersion.Version] = storage
   	}
       // 注册apigroup
   	if err := s.GenericAPIServer.InstallAPIGroup(&apiGroupInfo); err != nil {
   		return nil, err
   	}
      // clientset创建
   	crdClient, err := clientset.NewForConfig(s.GenericAPIServer.LoopbackClientConfig)
   	if err != nil {
   		// it's really bad that this is leaking here, but until we can fix the test (which I'm pretty sure isn't even testing what it wants to test),
   		// we need to be able to move forward
   		return nil, fmt.Errorf("failed to create clientset: %v", err)
   	}
   	s.Informers = externalinformers.NewSharedInformerFactory(crdClient, 5*time.Minute)
      // 创建各种handler
   	delegateHandler := delegationTarget.UnprotectedHandler()
   	if delegateHandler == nil {
   		delegateHandler = http.NotFoundHandler()
   	}
   
   	versionDiscoveryHandler := &versionDiscoveryHandler{
   		discovery: map[schema.GroupVersion]*discovery.APIVersionHandler{},
   		delegate:  delegateHandler,
   	}
   	groupDiscoveryHandler := &groupDiscoveryHandler{
   		discovery: map[string]*discovery.APIGroupHandler{},
   		delegate:  delegateHandler,
   	}
   	establishingController := establish.NewEstablishingController(s.Informers.Apiextensions().V1().CustomResourceDefinitions(), crdClient.ApiextensionsV1())
   	crdHandler, err := NewCustomResourceDefinitionHandler(
   		versionDiscoveryHandler,
   		groupDiscoveryHandler,
   		s.Informers.Apiextensions().V1().CustomResourceDefinitions(),
   		delegateHandler,
   		c.ExtraConfig.CRDRESTOptionsGetter,
   		c.GenericConfig.AdmissionControl,
   		establishingController,
   		c.ExtraConfig.ServiceResolver,
   		c.ExtraConfig.AuthResolverWrapper,
   		c.ExtraConfig.MasterCount,
   		s.GenericAPIServer.Authorizer,
   		c.GenericConfig.RequestTimeout,
   		time.Duration(c.GenericConfig.MinRequestTimeout)*time.Second,
   		apiGroupInfo.StaticOpenAPISpec,
   		c.GenericConfig.MaxRequestBodyBytes,
   	)
   	if err != nil {
   		return nil, err
   	}
   	s.GenericAPIServer.Handler.NonGoRestfulMux.Handle("/apis", crdHandler)
   	s.GenericAPIServer.Handler.NonGoRestfulMux.HandlePrefix("/apis/", crdHandler)
   
   	discoveryController := NewDiscoveryController(s.Informers.Apiextensions().V1().CustomResourceDefinitions(), versionDiscoveryHandler, groupDiscoveryHandler)
   	namingController := status.NewNamingConditionController(s.Informers.Apiextensions().V1().CustomResourceDefinitions(), crdClient.ApiextensionsV1())
   	nonStructuralSchemaController := nonstructuralschema.NewConditionController(s.Informers.Apiextensions().V1().CustomResourceDefinitions(), crdClient.ApiextensionsV1())
   	apiApprovalController := apiapproval.NewKubernetesAPIApprovalPolicyConformantConditionController(s.Informers.Apiextensions().V1().CustomResourceDefinitions(), crdClient.ApiextensionsV1())
   	finalizingController := finalizer.NewCRDFinalizer(
   		s.Informers.Apiextensions().V1().CustomResourceDefinitions(),
   		crdClient.ApiextensionsV1(),
   		crdHandler,
   	)
   	openapiController := openapicontroller.NewController(s.Informers.Apiextensions().V1().CustomResourceDefinitions())
      // 加入到启动hook中
   	s.GenericAPIServer.AddPostStartHookOrDie("start-apiextensions-informers", func(context genericapiserver.PostStartHookContext) error {
   		s.Informers.Start(context.StopCh)
   		return nil
   	})
   	s.GenericAPIServer.AddPostStartHookOrDie("start-apiextensions-controllers", func(context genericapiserver.PostStartHookContext) error {
   		// OpenAPIVersionedService and StaticOpenAPISpec are populated in generic apiserver PrepareRun().
   		// Together they serve the /openapi/v2 endpoint on a generic apiserver. A generic apiserver may
   		// choose to not enable OpenAPI by having null openAPIConfig, and thus OpenAPIVersionedService
   		// and StaticOpenAPISpec are both null. In that case we don't run the CRD OpenAPI controller.
   		if s.GenericAPIServer.OpenAPIVersionedService != nil && s.GenericAPIServer.StaticOpenAPISpec != nil {
   			go openapiController.Run(s.GenericAPIServer.StaticOpenAPISpec, s.GenericAPIServer.OpenAPIVersionedService, context.StopCh)
   		}
   
   		go namingController.Run(context.StopCh)
   		go establishingController.Run(context.StopCh)
   		go nonStructuralSchemaController.Run(5, context.StopCh)
   		go apiApprovalController.Run(5, context.StopCh)
   		go finalizingController.Run(5, context.StopCh)
   
   		discoverySyncedCh := make(chan struct{})
   		go discoveryController.Run(context.StopCh, discoverySyncedCh)
   		select {
   		case <-context.StopCh:
   		case <-discoverySyncedCh:
   		}
   
   	return nil
   	})
   // we don't want to report healthy until we can handle all CRDs that have already been registered.  Waiting for the informer
   	// to sync makes sure that the lister will be valid before we begin.  There may still be races for CRDs added after startup,
   	// but we won't go healthy until we can handle the ones already present.
   	s.GenericAPIServer.AddPostStartHookOrDie("crd-informer-synced", func(context genericapiserver.PostStartHookContext) error {
   		return wait.PollImmediateUntil(100*time.Millisecond, func() (bool, error) {
   			return s.Informers.Apiextensions().V1().CustomResourceDefinitions().Informer().HasSynced(), nil
   		}, context.StopCh)
   	})
   
   	return s, nil
   }
   ```

   c.GenericConfig.New来初始化genericapiserver,包裹一些默认链，创建handler

   ```
   // New creates a new server which logically combines the handling chain with the passed server.
   // name is used to differentiate for logging. The handler chain in particular can be difficult as it starts delgating.
   // delegationTarget may not be nil.
   func (c completedConfig) New(name string, delegationTarget DelegationTarget) (*GenericAPIServer, error) {
   	if c.Serializer == nil {
   		return nil, fmt.Errorf("Genericapiserver.New() called with config.Serializer == nil")
   	}
   	if c.LoopbackClientConfig == nil {
   		return nil, fmt.Errorf("Genericapiserver.New() called with config.LoopbackClientConfig == nil")
   	}
   if c.EquivalentResourceRegistry == nil {
   		return nil, fmt.Errorf("Genericapiserver.New() called with config.EquivalentResourceRegistry == nil")
   }
      // 包裹了DefaultBuildHandlerChain
   	handlerChainBuilder := func(handler http.Handler) http.Handler {
   		return c.BuildHandlerChainFunc(handler, c.Config)
   	}
   	// 创建apiserverhandler
   	apiServerHandler := NewAPIServerHandler(name, c.Serializer, handlerChainBuilder, delegationTarget.UnprotectedHandler())
   	...
   
   	return s, nil
   }
   ```

   APIServerHandler包含多种http.Handler类型，包括go-restful以及non-go-restful，以及在以上两者之间选择的Director对象，go-restful用于处理已经注册的handler，non-go-restful用来处理不存在的handler，API URI处理的选择过程为：FullHandlerChain-> Director ->{GoRestfulContainer， NonGoRestfulMux}。NewAPIServerHandler

   ```
   func NewAPIServerHandler(name string, s runtime.NegotiatedSerializer, handlerChainBuilder HandlerChainBuilderFn, notFoundHandler http.Handler) *APIServerHandler {
   // non-go-restful路由
   	nonGoRestfulMux := mux.NewPathRecorderMux(name)
   	if notFoundHandler != nil {
   		nonGoRestfulMux.NotFoundHandler(notFoundHandler)
   	}
   // go-resetful路由
   	gorestfulContainer := restful.NewContainer()
   	gorestfulContainer.ServeMux = http.NewServeMux()
   	gorestfulContainer.Router(restful.CurlyRouter{}) // e.g. for proxy/{kind}/{name}/{*}
   	gorestfulContainer.RecoverHandler(func(panicReason interface{}, httpWriter http.ResponseWriter) {
   		logStackOnRecover(s, panicReason, httpWriter)
   	})
   	gorestfulContainer.ServiceErrorHandler(func(serviceErr restful.ServiceError, request *restful.Request, response *restful.Response) {
   		serviceErrorHandler(s, serviceErr, request, response)
   	})
   // 选择器, 根据path选择是否执行go-restful，注册过的path执行go-restful
   director := director{
   		name:               name,
   	goRestfulContainer: gorestfulContainer,
   		nonGoRestfulMux:    nonGoRestfulMux,
   }
   
   return &APIServerHandler{
   		FullHandlerChain:   handlerChainBuilder(director),
   	GoRestfulContainer: gorestfulContainer,
   		NonGoRestfulMux:    nonGoRestfulMux,
   		Director:           director,
   	}
   }
   ```

   以上是APIExtensionsServer的初始化流程，初始化Server, 调用s.GenericAPIServer.InstallAPIGroup注册api。此方法的调用链非常深，主要是为了将需要暴露的API Resource注册到 server 中，以便能通过 http 接口进行 resource 的 REST 操作，其他几种 server 在初始化时也都会执行对应的 InstallAPI方法。

   ### KubeAPIServer初始化

   KubeAPIServer 主要是提供对 API Resource 的操作请求，为 kubernetes 中众多 API 注册路由信息，暴露 RESTful API 并且对外提供 kubernetes service，使集群中以及集群外的服务都可以通过 RESTful API 操作 kubernetes 中的资源。

   与`APIExtensionsServer`，`KubeAPIServer`初始化流程如下

   1. `CreateKubeAPIServer`调用`kubeAPIServerConfig.Complete().New`来初始化

2. `New`函数创建默认的`apigroup`(pod,deployment等内部资源), 调用`InstallAPIs`注册
   
5. 启动相关controller, 加入到`poststarthook`

   ##### kubeAPIServerConfig.Complete().New

   主要逻辑为：

   1、调用 `c.GenericConfig.New` 初始化 GenericAPIServer，其主要实现在上文已经分析过；

   2、判断是否支持 logs 相关的路由，如果支持，则添加 `/logs` 路由；

   3、调用 `m.InstallLegacyAPI` 将核心 API Resource 添加到路由中，对应到 apiserver 就是以 `/api` 开头的 resource；

   4、调用 `m.InstallAPIs` 将扩展的 API Resource 添加到路由中，在 apiserver 中即是以 `/apis` 开头的 resource；

   k8s.io/kubernetes/cmd/kube-apiserver/app/server.go

   ```
   // CreateKubeAPIServer creates and wires a workable kube-apiserver
   func CreateKubeAPIServer(kubeAPIServerConfig *controlplane.Config, delegateAPIServer genericapiserver.DelegationTarget) (*controlplane.Instance, error) {
   	kubeAPIServer, err := kubeAPIServerConfig.Complete().New(delegateAPIServer)
   	if err != nil {
   		return nil, err
   	}
   
   	return kubeAPIServer, nil
   }
   
   
   	if err := config.GenericConfig.AddPostStartHook("start-kube-apiserver-admission-initializer", admissionPostStartHook); err != nil {
   		return nil, nil, nil, nil, err
   	}
   ```

   k8s.io\Kubernetes\pkg\controlplane\instance.go

   ```
   // New returns a new instance of Master from the given config.
   // Certain config fields will be set to a default value if unset.
   // Certain config fields must be specified, including:
   //   KubeletClientConfig
   func (c completedConfig) New(delegationTarget genericapiserver.DelegationTarget) (*Instance, error) {
   	if reflect.DeepEqual(c.ExtraConfig.KubeletClientConfig, kubeletclient.KubeletClientConfig{}) {
   		return nil, fmt.Errorf("Master.New() called with empty config.KubeletClientConfig")
   	}
   // 1、初始化 GenericAPIServer
   	s, err := c.GenericConfig.New("kube-apiserver", delegationTarget)
   	if err != nil {
   		return nil, err
   	}
   // 2、注册 logs 相关的路由
   	if c.ExtraConfig.EnableLogsSupport {
   		routes.Logs{}.Install(s.Handler.GoRestfulContainer)
   	}
   
   	if utilfeature.DefaultFeatureGate.Enabled(features.ServiceAccountIssuerDiscovery) {
   		// Metadata and keys are expected to only change across restarts at present,
   		// so we just marshal immediately and serve the cached JSON bytes.
   		md, err := serviceaccount.NewOpenIDMetadata(
   			c.ExtraConfig.ServiceAccountIssuerURL,
   			c.ExtraConfig.ServiceAccountJWKSURI,
   			c.GenericConfig.ExternalAddress,
   			c.ExtraConfig.ServiceAccountPublicKeys,
   		)
   		if err != nil {
   			// If there was an error, skip installing the endpoints and log the
   			// error, but continue on. We don't return the error because the
   			// metadata responses require additional, backwards incompatible
   			// validation of command-line options.
   			msg := fmt.Sprintf("Could not construct pre-rendered responses for"+
   				" ServiceAccountIssuerDiscovery endpoints. Endpoints will not be"+
   				" enabled. Error: %v", err)
   			if c.ExtraConfig.ServiceAccountIssuerURL != "" {
   				// The user likely expects this feature to be enabled if issuer URL is
      				// set and the feature gate is enabled. In the future, if there is no
   				// longer a feature gate and issuer URL is not set, the user may not
      				// expect this feature to be enabled. We log the former case as an Error
      				// and the latter case as an Info.
      				klog.Error(msg)
      			} else {
      				klog.Info(msg)
      			}
      		} else {
      			routes.NewOpenIDMetadataServer(md.ConfigJSON, md.PublicKeysetJSON).
      				Install(s.Handler.GoRestfulContainer)
      		}
      	}
      
      	m := &Instance{
      		GenericAPIServer:          s,
      		ClusterAuthenticationInfo: c.ExtraConfig.ClusterAuthenticationInfo,
      	}
      
      	// install legacy rest storage
      	// 3、安装 LegacyAPI
      	if c.ExtraConfig.APIResourceConfigSource.VersionEnabled(apiv1.SchemeGroupVersion) {
      		legacyRESTStorageProvider := corerest.LegacyRESTStorageProvider{
      			StorageFactory:              c.ExtraConfig.StorageFactory,
      			ProxyTransport:              c.ExtraConfig.ProxyTransport,
      			KubeletClientConfig:         c.ExtraConfig.KubeletClientConfig,
      			EventTTL:                    c.ExtraConfig.EventTTL,
      			ServiceIPRange:              c.ExtraConfig.ServiceIPRange,
      			SecondaryServiceIPRange:     c.ExtraConfig.SecondaryServiceIPRange,
      			ServiceNodePortRange:        c.ExtraConfig.ServiceNodePortRange,
      			LoopbackClientConfig:        c.GenericConfig.LoopbackClientConfig,
      			ServiceAccountIssuer:        c.ExtraConfig.ServiceAccountIssuer,
      			ExtendExpiration:            c.ExtraConfig.ExtendExpiration,
      			ServiceAccountMaxExpiration: c.ExtraConfig.ServiceAccountMaxExpiration,
      			APIAudiences:                c.GenericConfig.Authentication.APIAudiences,
      		}
      		if err := m.InstallLegacyAPI(&c, c.GenericConfig.RESTOptionsGetter, legacyRESTStorageProvider); err != nil {
      			return nil, err
      		}
      	}
      
      	// The order here is preserved in discovery.
      	// If resources with identical names exist in more than one of these groups (e.g. "deployments.apps"" and "deployments.extensions"),
      	// the order of this list determines which group an unqualified resource name (e.g. "deployments") should prefer.
      	// This priority order is used for local discovery, but it ends up aggregated in `k8s.io/kubernetes/cmd/kube-apiserver/app/aggregator.go
      	// with specific priorities.
      	// TODO: describe the priority all the way down in the RESTStorageProviders and plumb it back through the various discovery
      	// handlers that we have.
      	restStorageProviders := []RESTStorageProvider{
      		apiserverinternalrest.StorageProvider{},
      		authenticationrest.RESTStorageProvider{Authenticator: c.GenericConfig.Authentication.Authenticator, APIAudiences: c.GenericConfig.Authentication.APIAudiences},
      		authorizationrest.RESTStorageProvider{Authorizer: c.GenericConfig.Authorization.Authorizer, RuleResolver: c.GenericConfig.RuleResolver},
      		autoscalingrest.RESTStorageProvider{},
   		batchrest.RESTStorageProvider{},
   ```

   ```
   	certificatesrest.RESTStorageProvider{},
   	coordinationrest.RESTStorageProvider{},
   	discoveryrest.StorageProvider{},
   	extensionsrest.RESTStorageProvider{},
   	networkingrest.RESTStorageProvider{},
   	noderest.RESTStorageProvider{},
   	policyrest.RESTStorageProvider{},
   	rbacrest.RESTStorageProvider{Authorizer: c.GenericConfig.Authorization.Authorizer},
   	schedulingrest.RESTStorageProvider{},
   	storagerest.RESTStorageProvider{},
   	flowcontrolrest.RESTStorageProvider{},
   	// keep apps after extensions so legacy clients resolve the extensions versions of shared resource names.
   	// See https://github.com/kubernetes/kubernetes/issues/42392
   	appsrest.StorageProvider{},
   	admissionregistrationrest.RESTStorageProvider{},
   	eventsrest.RESTStorageProvider{TTL: c.ExtraConfig.EventTTL},
   }
   
   // 4、安装 APIs
   if err := m.InstallAPIs(c.ExtraConfig.APIResourceConfigSource, c.GenericConfig.RESTOptionsGetter, restStorageProviders...); err != nil {
   	return nil, err
   }
    
   if c.ExtraConfig.Tunneler != nil {
   	m.installTunneler(c.ExtraConfig.Tunneler, corev1client.NewForConfigOrDie(c.GenericConfig.LoopbackClientConfig).Nodes())
   }
    
   m.GenericAPIServer.AddPostStartHookOrDie("start-cluster-authentication-info-controller", func(hookContext genericapiserver.PostStartHookContext) error {
   	kubeClient, err := kubernetes.NewForConfig(hookContext.LoopbackClientConfig)
   	if err != nil {
   		return err
   	}
   	controller := clusterauthenticationtrust.NewClusterAuthenticationTrustController(m.ClusterAuthenticationInfo, kubeClient)
    
   	// prime values and start listeners
   	if m.ClusterAuthenticationInfo.ClientCA != nil {
   		if notifier, ok := m.ClusterAuthenticationInfo.ClientCA.(dynamiccertificates.Notifier); ok {
   			notifier.AddListener(controller)
   		}
   		if controller, ok := m.ClusterAuthenticationInfo.ClientCA.(dynamiccertificates.ControllerRunner); ok {
   			// runonce to be sure that we have a value.
   			if err := controller.RunOnce(); err != nil {
   				runtime.HandleError(err)
   			}
   			go controller.Run(1, hookContext.StopCh)
   		}
   	}
   	if m.ClusterAuthenticationInfo.RequestHeaderCA != nil {
   		if notifier, ok := m.ClusterAuthenticationInfo.RequestHeaderCA.(dynamiccertificates.Notifier); ok {
   			notifier.AddListener(controller)
   		}
   		if controller, ok := m.ClusterAuthenticationInfo.RequestHeaderCA.(dynamiccertificates.ControllerRunner); ok {
   			// runonce to be sure that we have a value.
   			if err := controller.RunOnce(); err != nil {
   				runtime.HandleError(err)
   			}
   			go controller.Run(1, hookContext.StopCh)
   		}
   	}
    
   	go controller.Run(1, hookContext.StopCh)
   	return nil
   })
    
   return m, nil
   ```
    }
   ```
   
    ##### m.InstallLegacyAPI
   
    此方法的主要功能是将 core API 注册到路由中，是 apiserver 初始化流程中最核心的方法之一，不过其调用链非常深，下面会进行深入分析。将 API 注册到路由其最终的目的就是对外提供 RESTful API 来操作对应 resource，注册 API 主要分为两步，第一步是为 API 中的每个 resource 初始化 RESTStorage 以此操作后端存储中数据的变更，第二步是为每个 resource 根据其 verbs 构建对应的路由。`m.InstallLegacyAPI` 的主要逻辑为：
   
    - 1、调用 `legacyRESTStorageProvider.NewLegacyRESTStorage` 为 LegacyAPI 中各个资源创建 RESTStorage，RESTStorage 的目的是将每种资源的访问路径及其后端存储的操作对应起来；
    - 2、初始化 `bootstrap-controller`，并将其加入到 PostStartHook 中，`bootstrap-controller` 是 apiserver 中的一个 controller，主要功能是创建系统所需要的一些 namespace 以及创建 kubernetes service 并定期触发对应的 sync 操作，apiserver 在启动后会通过调用 PostStartHook 来启动 `bootstrap-controller`；
    - 3、在为资源创建完 RESTStorage 后，调用 `m.GenericAPIServer.InstallLegacyAPIGroup` 为 APIGroup 注册路由信息，`InstallLegacyAPIGroup`方法的调用链非常深，主要为`InstallLegacyAPIGroup--> installAPIResources --> InstallREST --> Install --> registerResourceHandlers`，最终核心的路由构造在`registerResourceHandlers`方法内，该方法比较复杂，其主要功能是通过上一步骤构造的 REST Storage 判断该资源可以执行哪些操作（如 create、update等），将其对应的操作存入到 action 中，每一个 action 对应一个标准的 REST 操作，如 create 对应的 action 操作为 POST、update 对应的 action 操作为PUT。最终根据 actions 数组依次遍历，对每一个操作添加一个 handler 方法，注册到 route 中去，再将 route 注册到 webservice 中去，webservice 最终会注册到 container 中，遵循 go-restful 的设计模式；
   
    关于 `legacyRESTStorageProvider.NewLegacyRESTStorage` 以及 `m.GenericAPIServer.InstallLegacyAPIGroup` 方法的详细说明在后文中会继续进行讲解。
   
    k8s.io\Kubernetes\pkg\controlplane\instance.go
   
   ```
    // InstallLegacyAPI will install the legacy APIs for the restStorageProviders if they are enabled.
    func (m *Instance) InstallLegacyAPI(c *completedConfig, restOptionsGetter generic.RESTOptionsGetter, legacyRESTStorageProvider corerest.LegacyRESTStorageProvider) error {
    	legacyRESTStorage, apiGroupInfo, err := legacyRESTStorageProvider.NewLegacyRESTStorage(restOptionsGetter)
    	if err != nil {
    		return fmt.Errorf("error building core storage: %v", err)
    	}

   ```
   controllerName := "bootstrap-controller"
   coreClient := corev1client.NewForConfigOrDie(c.GenericConfig.LoopbackClientConfig)
   bootstrapController := c.NewBootstrapController(legacyRESTStorage, coreClient, coreClient, coreClient, coreClient.RESTClient())
   m.GenericAPIServer.AddPostStartHookOrDie(controllerName, bootstrapController.PostStartHook)
   m.GenericAPIServer.AddPreShutdownHookOrDie(controllerName, bootstrapController.PreShutdownHook)
    
   if err := m.GenericAPIServer.InstallLegacyAPIGroup(genericapiserver.DefaultLegacyAPIPrefix, &apiGroupInfo); err != nil {
   	return fmt.Errorf("error in registering group versions: %v", err)
   }
   return nil
   ```
    }
   ```
   
    `InstallAPIs` 与 `InstallLegacyAPI` 的主要流程是类似的
   ```

   ### AggregatorServer初始化

   `Aggregator`通过`APIServices`对象关联到某个`Service`来进行请求的转发，其关联的`Service`类型进一步决定了请求转发形式。`Aggregator`包括一个`GenericAPIServer`和维护自身状态的`Controller`。其中 `GenericAPIServer`主要处理`apiregistration.k8s.io`组下的`APIService`资源请求。

   `Aggregator`除了处理资源请求外还包含几个controller：

   1. apiserviceRegistrationController：负责`APIServices`中资源的注册与删除；
   2. availableConditionController：维护`APIServices`的可用状态，包括其引用`Service`是否可用等；
   3. autoRegistrationController：用于保持API中存在的一组特定的`APIServices`；
   4. crdRegistrationController：负责将`CRD GroupVersions`自动注册到`APIServices`中；
   5. openAPIAggregationController：将`APIServices`资源的变化同步至提供的`OpenAPI`文档；
      kubernetes中的一些附加组件，比如metrics-server就是通过Aggregator的方式进行扩展的，实际环境中可以通过使用apiserver-builder工具轻松以Aggregator的扩展方式创建自定义资源。

   初始化AggregatorServer的主要逻辑为：

   1. 调用`aggregatorConfig.Complete().NewWithDelegate`创建`aggregatorServer`
   2. 初始化`crdRegistrationController`和`autoRegistrationController`，`crdRegistrationController`负责注册CRD，`autoRegistrationController`负责将 CRD 对应的 APIServices自动注册到apiserver中，CRD 创建后可通过`$ kubectl get apiservices`查看是否注册到 apiservices中
   3. 将`autoRegistrationController`和`crdRegistrationController`加入到PostStartHook中

   首先，初始化配置`createAggregatorConfig`

   ```
   func createAggregatorConfig(
   	kubeAPIServerConfig genericapiserver.Config,
   	commandOptions *options.ServerRunOptions,
   	externalInformers kubeexternalinformers.SharedInformerFactory,
   	serviceResolver aggregatorapiserver.ServiceResolver,
   	proxyTransport *http.Transport,
   	pluginInitializers []admission.PluginInitializer,
   ) (*aggregatorapiserver.Config, error) {
   	// make a shallow copy to let us twiddle a few things
   	// most of the config actually remains the same.  We only need to mess with a couple items related to the particulars of the aggregator
   	genericConfig := kubeAPIServerConfig
   	genericConfig.PostStartHooks = map[string]genericapiserver.PostStartHookConfigEntry{}
   	genericConfig.RESTOptionsGetter = nil
   
   	// override genericConfig.AdmissionControl with kube-aggregator's scheme,
   	// because aggregator apiserver should use its own scheme to convert its own resources.
   	// 取消admission的配置，aggregator自行处理请求，不需要admissions
   	err := commandOptions.Admission.ApplyTo(
   		&genericConfig,
   		externalInformers,
   		genericConfig.LoopbackClientConfig,
   		feature.DefaultFeatureGate,
   		pluginInitializers...)
   	if err != nil {
   		return nil, err
   	}
   
   	// copy the etcd options so we don't mutate originals.
   	etcdOptions := *commandOptions.Etcd
   	etcdOptions.StorageConfig.Paging = utilfeature.DefaultFeatureGate.Enabled(features.APIListChunking)
   	etcdOptions.StorageConfig.Codec = aggregatorscheme.Codecs.LegacyCodec(v1beta1.SchemeGroupVersion, v1.SchemeGroupVersion)
   	etcdOptions.StorageConfig.EncodeVersioner = runtime.NewMultiGroupVersioner(v1beta1.SchemeGroupVersion, schema.GroupKind{Group: v1beta1.GroupName})
   	genericConfig.RESTOptionsGetter = &genericoptions.SimpleRestOptionsFactory{Options: etcdOptions}
   
   	// override MergedResourceConfig with aggregator defaults and registry
   	// 取消admission的配置，aggregator自行处理请求，不需要admissions
   	if err := commandOptions.APIEnablement.ApplyTo(
   		&genericConfig,
   		aggregatorapiserver.DefaultAPIResourceConfigSource(),
   		aggregatorscheme.Scheme); err != nil {
   		return nil, err
   	}
   
   	aggregatorConfig := &aggregatorapiserver.Config{
   		GenericConfig: &genericapiserver.RecommendedConfig{
   			Config:                genericConfig,
   			SharedInformerFactory: externalInformers,
   		},
   		ExtraConfig: aggregatorapiserver.ExtraConfig{
   			ProxyClientCertFile: commandOptions.ProxyClientCertFile,
   			ProxyClientKeyFile:  commandOptions.ProxyClientKeyFile,
   			ServiceResolver:     serviceResolver,
   			// 代理请求的具体实现
   			ProxyTransport:      proxyTransport,
   		},
   	}
   
   	// we need to clear the poststarthooks so we don't add them multiple times to all the servers (that fails)
   	// 加入PostStartHook
   	aggregatorConfig.GenericConfig.PostStartHooks = map[string]genericapiserver.PostStartHookConfigEntry{}
   
   	return aggregatorConfig, nil
   }
   ```

   始化AggregatorcreateAggregatorServer初始化Aggregator

   ```
   func createAggregatorServer(aggregatorConfig *aggregatorapiserver.Config, delegateAPIServer genericapiserver.DelegationTarget, apiExtensionInformers apiextensionsinformers.SharedInformerFactory) (*aggregatorapiserver.APIAggregator, error) {
      //初始化 aggregatorServer
   	aggregatorServer, err := aggregatorConfig.Complete().NewWithDelegate(delegateAPIServer)
   	if err != nil {
   		return nil, err
   	}
   
   	// create controllers for auto-registration
   		// 创建auto-registration controller
   	apiRegistrationClient, err := apiregistrationclient.NewForConfig(aggregatorConfig.GenericConfig.LoopbackClientConfig)
   	if err != nil {
   		return nil, err
   	}
   	autoRegistrationController := autoregister.NewAutoRegisterController(aggregatorServer.APIRegistrationInformers.Apiregistration().V1().APIServices(), apiRegistrationClient)
   	apiServices := apiServicesToRegister(delegateAPIServer, autoRegistrationController)
   	crdRegistrationController := crdregistration.NewCRDRegistrationController(
   		apiExtensionInformers.Apiextensions().V1().CustomResourceDefinitions(),
   		autoRegistrationController)
   
   	err = aggregatorServer.GenericAPIServer.AddPostStartHook("kube-apiserver-autoregistration", func(context genericapiserver.PostStartHookContext) error {
   	// 启动controller
   		go crdRegistrationController.Run(5, context.StopCh)
   		go func() {
   			// let the CRD controller process the initial set of CRDs before starting the autoregistration controller.
   			// this prevents the autoregistration controller's initial sync from deleting APIServices for CRDs that still exist.
   			// we only need to do this if CRDs are enabled on this server.  We can't use discovery because we are the source for discovery.
   			if aggregatorConfig.GenericConfig.MergedResourceConfig.AnyVersionForGroupEnabled("apiextensions.k8s.io") {
   				crdRegistrationController.WaitForInitialSync()
   			}
   			autoRegistrationController.Run(5, context.StopCh)
   		}()
   		return nil
   	})
   	if err != nil {
   		return nil, err
   	}
   
   	err = aggregatorServer.GenericAPIServer.AddBootSequenceHealthChecks(
   		makeAPIServiceAvailableHealthCheck(
   			"autoregister-completion",
   			apiServices,
   			aggregatorServer.APIRegistrationInformers.Apiregistration().V1().APIServices(),
   		),
   	)
   	if err != nil {
   		return nil, err
   	}
   
   	return aggregatorServer, nil
   }
   ```

   ##### aggregatorConfig.Complete().NewWithDelegate

   `aggregatorConfig.Complete().NewWithDelegate` 是初始化 aggregatorServer 的方法，主要逻辑为：

   - 1、调用 `c.GenericConfig.New` 初始化 GenericAPIServer，其内部的主要功能在上文已经分析过；
   - 2、调用 `apiservicerest.NewRESTStorage` 为 APIServices 资源创建 RESTStorage，RESTStorage 的目的是将每种资源的访问路径及其后端存储的操作对应起来；
   - 3、调用 `s.GenericAPIServer.InstallAPIGroup` 为 APIGroup 注册路由信息；

至此，启动步骤以前分析完了，三个组件的流量大体时一样的，通过`Complete().New()`初始化配置，创建所需的controller, 调用`InstallAPIGroup`注册`apigroup`。

k8s.io\Kubernetes\cmd\kube-apiserver\app\aggregator.go

```
func createAggregatorServer(aggregatorConfig *aggregatorapiserver.Config, delegateAPIServer genericapiserver.DelegationTarget, apiExtensionInformers apiextensionsinformers.SharedInformerFactory) (*aggregatorapiserver.APIAggregator, error) {
	aggregatorServer, err := aggregatorConfig.Complete().NewWithDelegate(delegateAPIServer)
	if err != nil {
		return nil, err
	}

	// create controllers for auto-registration
	apiRegistrationClient, err := apiregistrationclient.NewForConfig(aggregatorConfig.GenericConfig.LoopbackClientConfig)
	if err != nil {
		return nil, err
	}
	autoRegistrationController := autoregister.NewAutoRegisterController(aggregatorServer.APIRegistrationInformers.Apiregistration().V1().APIServices(), apiRegistrationClient)
	apiServices := apiServicesToRegister(delegateAPIServer, autoRegistrationController)
	crdRegistrationController := crdregistration.NewCRDRegistrationController(
		apiExtensionInformers.Apiextensions().V1().CustomResourceDefinitions(),
		autoRegistrationController)

```



k8s.io\Kubernetes\vendor\k8s.io\kube-aggregator\pkg\apiserver\apiserver.go

```
// NewWithDelegate returns a new instance of APIAggregator from the given config.
func (c completedConfig) NewWithDelegate(delegationTarget genericapiserver.DelegationTarget) (*APIAggregator, error) {
	// Prevent generic API server to install OpenAPI handler. Aggregator server
	// has its own customized OpenAPI handler.
	openAPIConfig := c.GenericConfig.OpenAPIConfig
	c.GenericConfig.OpenAPIConfig = nil
// 1、初始化 genericServer
	genericServer, err := c.GenericConfig.New("kube-aggregator", delegationTarget)
	if err != nil {
		return nil, err
	}

	apiregistrationClient, err := clientset.NewForConfig(c.GenericConfig.LoopbackClientConfig)
	if err != nil {
		return nil, err
	}
	informerFactory := informers.NewSharedInformerFactory(
		apiregistrationClient,
		5*time.Minute, // this is effectively used as a refresh interval right now.  Might want to do something nicer later on.
	)

	s := &APIAggregator{
		GenericAPIServer:           genericServer,
		delegateHandler:            delegationTarget.UnprotectedHandler(),
		proxyTransport:             c.ExtraConfig.ProxyTransport,
		proxyHandlers:              map[string]*proxyHandler{},
		handledGroups:              sets.String{},
		lister:                     informerFactory.Apiregistration().V1().APIServices().Lister(),
		APIRegistrationInformers:   informerFactory,
		serviceResolver:            c.ExtraConfig.ServiceResolver,
		openAPIConfig:              openAPIConfig,
		egressSelector:             c.GenericConfig.EgressSelector,
		proxyCurrentCertKeyContent: func() (bytes []byte, bytes2 []byte) { return nil, nil },
	}
// 2、为 API 注册路由
	apiGroupInfo := apiservicerest.NewRESTStorage(c.GenericConfig.MergedResourceConfig, c.GenericConfig.RESTOptionsGetter)
	if err := s.GenericAPIServer.InstallAPIGroup(&apiGroupInfo); err != nil {
		return nil, err
	}

	enabledVersions := sets.NewString()
	for v := range apiGroupInfo.VersionedResourcesStorageMap {
		enabledVersions.Insert(v)
	}
	if !enabledVersions.Has(v1.SchemeGroupVersion.Version) {
		return nil, fmt.Errorf("API group/version %s must be enabled", v1.SchemeGroupVersion.String())
	}
// 3、初始化 apiserviceRegistrationController、availableController
	apisHandler := &apisHandler{
		codecs:         aggregatorscheme.Codecs,
		lister:         s.lister,
		discoveryGroup: discoveryGroup(enabledVersions),
	}
	s.GenericAPIServer.Handler.NonGoRestfulMux.Handle("/apis", apisHandler)
	s.GenericAPIServer.Handler.NonGoRestfulMux.UnlistedHandle("/apis/", apisHandler)

	apiserviceRegistrationController := NewAPIServiceRegistrationController(informerFactory.Apiregistration().V1().APIServices(), s)
	if len(c.ExtraConfig.ProxyClientCertFile) > 0 && len(c.ExtraConfig.ProxyClientKeyFile) > 0 {
		aggregatorProxyCerts, err := dynamiccertificates.NewDynamicServingContentFromFiles("aggregator-proxy-cert", c.ExtraConfig.ProxyClientCertFile, c.ExtraConfig.ProxyClientKeyFile)
		if err != nil {
			return nil, err
		}
		if err := aggregatorProxyCerts.RunOnce(); err != nil {
			return nil, err
		}
		aggregatorProxyCerts.AddListener(apiserviceRegistrationController)
		s.proxyCurrentCertKeyContent = aggregatorProxyCerts.CurrentCertKeyContent
// 4、添加 PostStartHook
		s.GenericAPIServer.AddPostStartHookOrDie("aggregator-reload-proxy-client-cert", func(context genericapiserver.PostStartHookContext) error {
			go aggregatorProxyCerts.Run(1, context.StopCh)
			return nil
		})
	}

	availableController, err := statuscontrollers.NewAvailableConditionController(
		informerFactory.Apiregistration().V1().APIServices(),
		c.GenericConfig.SharedInformerFactory.Core().V1().Services(),
		c.GenericConfig.SharedInformerFactory.Core().V1().Endpoints(),
		apiregistrationClient.ApiregistrationV1(),
		c.ExtraConfig.ProxyTransport,
		(func() ([]byte, []byte))(s.proxyCurrentCertKeyContent),
		s.serviceResolver,
		c.GenericConfig.EgressSelector,
	)
	if err != nil {
		return nil, err
	}

	s.GenericAPIServer.AddPostStartHookOrDie("start-kube-aggregator-informers", func(context genericapiserver.PostStartHookContext) error {
		informerFactory.Start(context.StopCh)
		c.GenericConfig.SharedInformerFactory.Start(context.StopCh)
		return nil
	})
	s.GenericAPIServer.AddPostStartHookOrDie("apiservice-registration-controller", func(context genericapiserver.PostStartHookContext) error {
		handlerSyncedCh := make(chan struct{})
		go apiserviceRegistrationController.Run(context.StopCh, handlerSyncedCh)
		select {
		case <-context.StopCh:
		case <-handlerSyncedCh:
		}

		return nil
	})
	s.GenericAPIServer.AddPostStartHookOrDie("apiservice-status-available-controller", func(context genericapiserver.PostStartHookContext) error {
		// if we end up blocking for long periods of time, we may need to increase threadiness.
		go availableController.Run(5, context.StopCh)
		return nil
	})

	return s, nil
}
```

以上是对 AggregatorServer 初始化流程的分析，可以看出，在创建 APIExtensionsServer、KubeAPIServer 以及 AggregatorServer 时，其模式都是类似的，首先调用 `c.GenericConfig.New` 按照`go-restful`的模式初始化 Container，然后为 server 中需要注册的资源创建 RESTStorage，最后将 resource 的 APIGroup 信息注册到路由中。

## 调用链

```
|-->CreateNodeDialer
|
|-->CreateKubeAPIServerConfig
|
CreateServerChain--|--> createAPIExtensionsConfig
|
||--> c.GenericConfig.New
|--> createAPIExtensionsServer --> apiextensionsConfig.Complete().New--|
||--> s.GenericAPIServer.InstallAPIGroup
|
||--> c.GenericConfig.New--> legacyRESTStorageProvider.NewLegacyRESTStorage
||
|-->CreateKubeAPIServer--> kubeAPIServerConfig.Complete().New--|--> m.InstallLegacyAPI
||
||--> m.InstallAPIs
|
|
|--> createAggregatorConfig
|
||--> c.GenericConfig.New
||
|--> createAggregatorServer --> aggregatorConfig.Complete().NewWithDelegate--|--> apiservicerest.NewRESTStorage
|
|--> s.GenericAPIServer.InstallAPIGroup
```

## 请求分析

上面我们分析了apiserver的调用链，大体如下
`DefaultHandlerChain->{handler/crdhandler/proxy}->admission->validation->etcd`

1. 请求进入时，会经过`defaultchain`做一些认证鉴权工作
2. 然后通过`route`执行对应的handler，如果为aggration api, 将直接转发请求到对应service
3. handler处理完，经过admission与validation，做一些修改和检查，用户在这部分可以自定义webhook
4. 最后存入etcd

## 总结

本文大体对apiserver的启动流程，以及初始化过程做了分析，由于apiserver实现复杂，中间一些细节没涉及到，还需要对着代码研究研究
