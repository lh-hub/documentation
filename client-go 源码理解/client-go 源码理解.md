### client-go 源码理解

项目地址：https://github.com/kubernetes/client-go/blob/kubernetes-1.24.10

直奔最基础的 namespace，看是如何实现的

client-go 获取 namespace 代码如下：

```go
import "k8s.io/client-go/kubernetes"

clientset, err := kubernetes.NewForConfig(config)
namespaceList, err := clientSet.CoreV1().Namespaces().List(context.TODO(),metav1.ListOptions{})
```

实现这一过程代码地址如下，对其进行分析。

https://github.com/kubernetes/client-go/blob/kubernetes-1.24.10/kubernetes/typed/core/v1/namespace.go

```go
package v1

import (
	"context"
	json "encoding/json"
	"fmt"
	"time"

	v1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	types "k8s.io/apimachinery/pkg/types"
	watch "k8s.io/apimachinery/pkg/watch"
	corev1 "k8s.io/client-go/applyconfigurations/core/v1"
	scheme "k8s.io/client-go/kubernetes/scheme"
	rest "k8s.io/client-go/rest"
)

// 定义获取 namespace 对象的接口，namespace 属于 core 组，所以 core_client 负责实现这个 Namespaces() 方法
// 方法返回 namespace 对象的所有操作方法
type NamespacesGetter interface {
	Namespaces() NamespaceInterface
}

// 定义 Namespace 对象的所有操作方法
type NamespaceInterface interface {
	Create(ctx context.Context, namespace *v1.Namespace, opts metav1.CreateOptions) (*v1.Namespace, error)
	Update(ctx context.Context, namespace *v1.Namespace, opts metav1.UpdateOptions) (*v1.Namespace, error)
	UpdateStatus(ctx context.Context, namespace *v1.Namespace, opts metav1.UpdateOptions) (*v1.Namespace, error)
	Delete(ctx context.Context, name string, opts metav1.DeleteOptions) error
	Get(ctx context.Context, name string, opts metav1.GetOptions) (*v1.Namespace, error)
	List(ctx context.Context, opts metav1.ListOptions) (*v1.NamespaceList, error)
	Watch(ctx context.Context, opts metav1.ListOptions) (watch.Interface, error)
	Patch(ctx context.Context, name string, pt types.PatchType, data []byte, opts metav1.PatchOptions, subresources ...string) (result *v1.Namespace, err error)
	Apply(ctx context.Context, namespace *corev1.NamespaceApplyConfiguration, opts metav1.ApplyOptions) (result *v1.Namespace, err error)
	ApplyStatus(ctx context.Context, namespace *corev1.NamespaceApplyConfiguration, opts metav1.ApplyOptions) (result *v1.Namespace, err error)
	NamespaceExpansion
}

// 定义 namespace 对象，拥有属性 client，类型为 rest.Interface
type namespaces struct {
	client rest.Interface
}

// namespace 属于 core 组，为 CoreV1Client 添加 newNamespaces 方法，用来返回 namespace 对象。
// 然后 CoreV1Client 再实现外部可调用的 Namespaces() 方法调用内部方法 newNamespaces() 返回 namespace 对象，如下：
// func (c *CoreV1Client) Namespaces() NamespaceInterface {
// 	return newNamespaces(c)
// }
func newNamespaces(c *CoreV1Client) *namespaces {
	return &namespaces{
		client: c.RESTClient(),
	}
}

// 下面是 namespace 对象所有方法的代码实现
func (c *namespaces) Get(ctx context.Context, name string, options metav1.GetOptions) (result *v1.Namespace, err error) {
	result = &v1.Namespace{}
	err = c.client.Get().
		Resource("namespaces").
		Name(name).
		VersionedParams(&options, scheme.ParameterCodec).
		Do(ctx).
		Into(result)
	return
}

func (c *namespaces) List(ctx context.Context, opts metav1.ListOptions) (result *v1.NamespaceList, err error) {
	var timeout time.Duration
	if opts.TimeoutSeconds != nil {
		timeout = time.Duration(*opts.TimeoutSeconds) * time.Second
	}
	result = &v1.NamespaceList{}
	err = c.client.Get().
		Resource("namespaces").
		VersionedParams(&opts, scheme.ParameterCodec).
		Timeout(timeout).
		Do(ctx).
		Into(result)
	return
}

func (c *namespaces) Watch(ctx context.Context, opts metav1.ListOptions) (watch.Interface, error) {
	var timeout time.Duration
	if opts.TimeoutSeconds != nil {
		timeout = time.Duration(*opts.TimeoutSeconds) * time.Second
	}
	opts.Watch = true
	return c.client.Get().
		Resource("namespaces").
		VersionedParams(&opts, scheme.ParameterCodec).
		Timeout(timeout).
		Watch(ctx)
}

func (c *namespaces) Create(ctx context.Context, namespace *v1.Namespace, opts metav1.CreateOptions) (result *v1.Namespace, err error) {
	result = &v1.Namespace{}
	err = c.client.Post().
		Resource("namespaces").
		VersionedParams(&opts, scheme.ParameterCodec).
		Body(namespace).
		Do(ctx).
		Into(result)
	return
}

func (c *namespaces) Update(ctx context.Context, namespace *v1.Namespace, opts metav1.UpdateOptions) (result *v1.Namespace, err error) {
	result = &v1.Namespace{}
	err = c.client.Put().
		Resource("namespaces").
		Name(namespace.Name).
		VersionedParams(&opts, scheme.ParameterCodec).
		Body(namespace).
		Do(ctx).
		Into(result)
	return
}

func (c *namespaces) UpdateStatus(ctx context.Context, namespace *v1.Namespace, opts metav1.UpdateOptions) (result *v1.Namespace, err error) {
	result = &v1.Namespace{}
	err = c.client.Put().
		Resource("namespaces").
		Name(namespace.Name).
		SubResource("status").
		VersionedParams(&opts, scheme.ParameterCodec).
		Body(namespace).
		Do(ctx).
		Into(result)
	return
}

func (c *namespaces) Delete(ctx context.Context, name string, opts metav1.DeleteOptions) error {
	return c.client.Delete().
		Resource("namespaces").
		Name(name).
		Body(&opts).
		Do(ctx).
		Error()
}

func (c *namespaces) Patch(ctx context.Context, name string, pt types.PatchType, data []byte, opts metav1.PatchOptions, subresources ...string) (result *v1.Namespace, err error) {
	result = &v1.Namespace{}
	err = c.client.Patch(pt).
		Resource("namespaces").
		Name(name).
		SubResource(subresources...).
		VersionedParams(&opts, scheme.ParameterCodec).
		Body(data).
		Do(ctx).
		Into(result)
	return
}

func (c *namespaces) Apply(ctx context.Context, namespace *corev1.NamespaceApplyConfiguration, opts metav1.ApplyOptions) (result *v1.Namespace, err error) {
	if namespace == nil {
		return nil, fmt.Errorf("namespace provided to Apply must not be nil")
	}
	patchOpts := opts.ToPatchOptions()
	data, err := json.Marshal(namespace)
	if err != nil {
		return nil, err
	}
	name := namespace.Name
	if name == nil {
		return nil, fmt.Errorf("namespace.Name must be provided to Apply")
	}
	result = &v1.Namespace{}
	err = c.client.Patch(types.ApplyPatchType).
		Resource("namespaces").
		Name(*name).
		VersionedParams(&patchOpts, scheme.ParameterCodec).
		Body(data).
		Do(ctx).
		Into(result)
	return
}

func (c *namespaces) ApplyStatus(ctx context.Context, namespace *corev1.NamespaceApplyConfiguration, opts metav1.ApplyOptions) (result *v1.Namespace, err error) {
	if namespace == nil {
		return nil, fmt.Errorf("namespace provided to Apply must not be nil")
	}
	patchOpts := opts.ToPatchOptions()
	data, err := json.Marshal(namespace)
	if err != nil {
		return nil, err
	}

	name := namespace.Name
	if name == nil {
		return nil, fmt.Errorf("namespace.Name must be provided to Apply")
	}

	result = &v1.Namespace{}
	err = c.client.Patch(types.ApplyPatchType).
		Resource("namespaces").
		Name(*name).
		SubResource("status").
		VersionedParams(&patchOpts, scheme.ParameterCodec).
		Body(data).
		Do(ctx).
		Into(result)
	return
}
```

再来看组 core 的代码逻辑，地址如下：

https://github.com/kubernetes/client-go/blob/kubernetes-1.24.10/kubernetes/typed/core/v1/core_client.go

对代码进行精简，只保留 namespace 的相对逻辑，方便理解

```go
package v1

import (
	"net/http"

	v1 "k8s.io/api/core/v1"
	"k8s.io/client-go/kubernetes/scheme"
	rest "k8s.io/client-go/rest"
)

// 定义 core 组所有方法的接口，在 kubernetes 模块的总入口，也就是所有组的集合定义处会调用
// 继承 NamespacesGetter，目的是调用 Namespaces() 方法返回 namespace 对象的所有操作方法
type CoreV1Interface interface {
	RESTClient() rest.Interface
	NamespacesGetter
}

// 定义 core 组的组 client
type CoreV1Client struct {
	restClient rest.Interface
}

// 定义外部可调用方法 Namespaces()，结合上面提到的 newNamespaces() 方法，返回 namespace 对象
func (c *CoreV1Client) Namespaces() NamespaceInterface {
	return newNamespaces(c)
}

// 根据配置文件创建 core client
func NewForConfig(c *rest.Config) (*CoreV1Client, error) {
	config := *c
    // 添加 core 组默认配置
	if err := setConfigDefaults(&config); err != nil {
		return nil, err
	}
    // 根据配置创建 rest.HTTPClient
	httpClient, err := rest.HTTPClientFor(&config)
	if err != nil {
		return nil, err
	}
    // 调用方法返回 CoreV1Client
	return NewForConfigAndClient(&config, httpClient)
}

// 返回 CoreV1Client
func NewForConfigAndClient(c *rest.Config, h *http.Client) (*CoreV1Client, error) {
	config := *c
	if err := setConfigDefaults(&config); err != nil {
		return nil, err
	}
	client, err := rest.RESTClientForConfigAndClient(&config, h)
	if err != nil {
		return nil, err
	}
	return &CoreV1Client{client}, nil
}

func NewForConfigOrDie(c *rest.Config) *CoreV1Client {
	client, err := NewForConfig(c)
	if err != nil {
		panic(err)
	}
	return client
}

// 创建新的 CoreV1Client
func New(c rest.Interface) *CoreV1Client {
	return &CoreV1Client{c}
}

func setConfigDefaults(config *rest.Config) error {
	gv := v1.SchemeGroupVersion
	config.GroupVersion = &gv
	config.APIPath = "/api"
	config.NegotiatedSerializer = scheme.Codecs.WithoutConversion()

	if config.UserAgent == "" {
		config.UserAgent = rest.DefaultKubernetesUserAgent()
	}

	return nil
}

func (c *CoreV1Client) RESTClient() rest.Interface {
	if c == nil {
		return nil
	}
	return c.restClient
}
```

再来看 kubernetes 总体 clientset 代码逻辑，地址如下：

https://github.com/kubernetes/client-go/blob/kubernetes-1.24.10/kubernetes/clientset.go

对代码进行精简，只保留 corev1 组上午相对逻辑，方便理解

```go
package kubernetes

import (
	"fmt"
	"net/http"

	corev1 "k8s.io/client-go/kubernetes/typed/core/v1"
	rest "k8s.io/client-go/rest"
	flowcontrol "k8s.io/client-go/util/flowcontrol"
)

type Interface interface {
    // 接口定义 CoreV1() 方法，返回 corev1 下的所有方法，这次分析的是 Namespaces() 方法
	CoreV1() corev1.CoreV1Interface
}

// 定义 kubernetes client 为 Clientset，主要分析 coreV1，类型为 corev1.CoreV1Client
type Clientset struct {
	*discovery.DiscoveryClient
	coreV1                       *corev1.CoreV1Client
}

// 定义 CoreV1()，返回属性 coreV1
func (c *Clientset) CoreV1() corev1.CoreV1Interface {
	return c.coreV1
}

// 根据配置文件，生成 Clientset
func NewForConfig(c *rest.Config) (*Clientset, error) {
	configShallowCopy := *c

	if configShallowCopy.UserAgent == "" {
		configShallowCopy.UserAgent = rest.DefaultKubernetesUserAgent()
	}

	httpClient, err := rest.HTTPClientFor(&configShallowCopy)
	if err != nil {
		return nil, err
	}

	return NewForConfigAndClient(&configShallowCopy, httpClient)
}

// 初始化 Clientset，为每一个组生成对应的 client，这里分析 corev1 client，Clientset.coreV1 = corev1.NewForConfigAndClien()
// 初始化后调用 CoreV1() 返回 Clientset.coreV1，然后返回所有方法，获取对应的对象
func NewForConfigAndClient(c *rest.Config, httpClient *http.Client) (*Clientset, error) {
	configShallowCopy := *c
	if configShallowCopy.RateLimiter == nil && configShallowCopy.QPS > 0 {
		if configShallowCopy.Burst <= 0 {
			return nil, fmt.Errorf("burst is required to be greater than 0 when RateLimiter is not set and QPS is set to greater than 0")
		}
		configShallowCopy.RateLimiter = flowcontrol.NewTokenBucketRateLimiter(configShallowCopy.QPS, configShallowCopy.Burst)
	}

	var cs Clientset
	var err error
	cs.coreV1, err = corev1.NewForConfigAndClient(&configShallowCopy, httpClient)
	if err != nil {
		return nil, err
	}
	return &cs, nil
}

func NewForConfigOrDie(c *rest.Config) *Clientset {
	cs, err := NewForConfig(c)
	if err != nil {
		panic(err)
	}
	return cs
}

func New(c rest.Interface) *Clientset {
	var cs Clientset
	cs.coreV1 = corev1.New(c)

	cs.DiscoveryClient = discovery.NewDiscoveryClient(c)
	return &cs
}
```

再来看最上面的获取 namespace 的代码

```go
import "k8s.io/client-go/kubernetes"

// 调用 NewForConfig 方法，初始化 Clientset，在 Clientset 中为每一个组创建了对应的组 client
clientset, err := kubernetes.NewForConfig(config)
// 调用 clientSet.CoreV1() 方法，返回属性 c.coreV1 为 core 组的 client
// 调用 Namespaces() 方法，返回 namespace 对象，然后调用对象方法，返回结果
namespaceList, err := clientSet.CoreV1().Namespaces().List(context.TODO(),metav1.ListOptions{})
```

