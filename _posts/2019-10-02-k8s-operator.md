---
title:      "如何写一个 kubernetes operator"
date:       2019-10-02 20:36:50 +0800
categories: 技术 
tags:
- kubernetes
- crd
- controller
---

# 0. 引言

kubernetes 已经成为事实上的容器编排标准，其管理的内容都抽象为资源，常见的资源有 pod、deployment、statefulset、service、job 等。无状态服务很容易通过 Deployment 实现容器化；有状态服务一般通过 statusfulset 方式实现容器化，但有状态服务的管理一般很复杂，在集群初始化、扩缩容、故障处理的时候，需要执行各自不同的操作，这些特殊的操作在 kubernetes 中并没有提供。

CoreOS 提出了 operator 的概念，即通过CRD(Custom Resource Definition) 自定义资源，并编写对应的controller 对 CRD 管理。现在已经有很多 operator，如 etcd-operator、mysql-operator、redis-operator 等。

本文将简要介绍如果创建自定义资源 CRD 及对应的控制器 controller，本文例子中的代码见：https://github.com/9sheng/foobar-operator。

# 1. 创建 CRD

假设资源为 FooBar、Group 为 test.example.com，通过 CustomResourceDefinition 创建 CRD，对应的 yaml 如下，我们也可以在 controller 里通过调用 api 创建 CRD。

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: FooBars.test.example.com
spec:
  group: test.example.com
  names:
    kind: Foobar
    listKind: FooBarList
    plural: FooBars
    singular: FooBar
  scope: Namespaced # 为全局变量为 Cluster
  version: v1
```

# 2. 实现 Controller
## 2.1 clientset

自定义资源的设置见这里[foobar_types.go](https://github.com/9sheng/foobar-operator/blob/master/pkg/apis/action/v1/foobar_types.go)，主要定义了 FooBar 这个结构，主要有2个部分：

- FooBarSpec：用户输入的 spec，即期望状态
- FooBarStatus：controller 处理的状态，记录当前状态

此外还有 FooBarList，这个在 list 资源时候用到。

定义好基本数据结构后，使用工具 [code-generator](https://github.com/kubernetes/code-generator) 生成客户端配套代码，主要有clientset、deepcopy、informer，通过这些配套代码，我们可以像使用 client-go 一样处理我们的自定义资源。code-generator 根据 foobar_types.go中的导言注释生成配套代码：

```go
// +genclient  # 生成客户端
// +genclient:nonNamespaced  # Cluster 范围的资源增加该注释 
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object  # 生成 deepcopy
```
生成命令见[这里](https://github.com/9sheng/foobar-operator/blob/master/hack/update-codegen.sh)

## 2.2 controller 处理流程

资源的主要处理逻辑见[foobar_controller.go](https://github.com/9sheng/foobar-operator/blob/master/pkg/controller/foobar/foobar_controller.go)，controller 通过设置 informer 监控资源的新增（AddFunc）、更新（UpdateFunc）、删除（DeleteFunc）三种事件的回调函数监控事件，监控到事件后，controller 并不立即处理相应的对象，而是放到一个队列 workqueue 中，然后由多个 worker 从队列中取出对象进行处理。通过这样的处理方式，有很多好处：

- workqueue 提供了滤重、限速等功能，能有效减少 controller 的工作量，一段时间内，可能收到多个事件，但通过过滤之后，我们只需要处理一次；
- 通过 workqueue，简化了并发处理逻辑，从 workqueue 中获取一个对象时，处理结束前没有 forget 该对象，该对象不能再加入到队列中，因此该对象不会被其他线程取到处理；
- 如果处理对象失败，只需要重新将对象丢到队列即可，后面我们有机会再一次处理，大大降低了错误处理逻辑

FooBarController 中的 `Run` 、`runWorker`、`processNextWorkItem` 比较固定，实现自己的 controller 时，只要复制粘贴即可；具体的处理逻辑放在 `syncHandler` 中，`syncHandler` 一般通过获取 spec 的数据，通过 k8s api 获取相关资源的状态，进行对比处理，使相关的状态达到一致，因涉及到具体处理逻辑，这里不再赘述。

# 3. 注意事项
## 3.1 格式校验
创建了CRD之后，我们可以提交对应的 CR 到 k8s api-server，但 CR 的内容可以是任意格式，k8s 并不会对 CR 的格式校验，如果某一 CR 的格式错误，在 controller 启动时，`WaitForCacheSync` 会因对象格式错误不能返回，我们可以通过 CRD 的 validation 属性对 CR 的格式做一个基本的校验，避免这个问题，更新的 CRD 如下：
```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: FooBars.test.example.com
spec:
  group: test.example.com
  names:
    kind: Foobar
    listKind: FooBarList
    plural: FooBars
    singular: FooBar
  scope: Namespaced
  version: v1
  validation: # 格式校验
    openAPIV3Schema:
      type: object
      properties:
        spec:
          type: object
          required:
          - target
          properties:
            target:
              type: string
            type:
              type: string
```
schema 校验规则见[这里](https://datatracker.ietf.org/doc/draft-handrews-json-schema-validation/?include_text=1)。k8s 在将数据写入 ectd 之前，会进行 schema 校验，如果不通过，则不写入 etcd，并将错误返回给用户。

## 3.2 syncHandler 的返回值

`syncHandler` 返回 `error` 类型，如果 err 不是 nil，则会重新加入到队列中处理。在处理对象时，有各种各样的错误，需要区分错误的类型，只有需要重复处理的错误才能从 `syncHandler` 中返回。

## 3.3 处理 delete 资源

默认情况下，如果一个资源被删除了，controller 会收到删除事件，但 worker 去获取资源时，很可能获取不到资源了，为避免这种情况，k8s 提供 [finalizer](https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definitions/#finalizers) 机制，让 controller 有机会处理删除的资源。如果资源使用了 finalizer，删除资源时，k8s 会标记资源的 DeletionTimestamp 而不立即去删除资源，controller可以通过 `foobar.ObjectMeta.GetDeletionTimestamp().IsZero()` 判断资源是否是删除状态。

## 3.4 更新处理

当处理完一个资源时，controller一般会更新资源的状态，一般通过 Update 方法实现，Update 之后 informer 会收到更新状态，controller 再一次处理该资源，这样就进入了死循环。因此 informer 的 Update 回调函数中需要判断资源的状态，如果资源不需要处理，就不要放入队列。

另一方面，我们的资源接口直接面向用户，用户可以直接更新资源的属性，甚至可以把我们记录在资源的状态直接删除掉（如执行了`kubectl replace`命令)，因为我们在处理 update 事件的时候，需要特别注意以下几点：

- 如果需要清理资源的旧状态，处理失败时怎么办，如果简单放入队列中，下一次处理时，获取不到老的状态；
- controller 重启，未处理成功的状态可能丢失；

带来这种处理复杂性的原因主要是，CRD是直接面向用户的。如果面向用户的 CRD 处理逻辑特别复杂，operator 可以创建内部使用的 CRD，一个 controller 处理用户提交的 CR，创建内部的 CR；另一个 controller 处理内部的 CR，进行实际的业务处理，这样可以降低 operator 的实现逻辑。

## 3.5 subresources

kubernetes 1.16 中提供了 GA 版本的 [subresource 机制](https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definitions/#status-subresource)，通过设置 status subresource，在更新资源的时候，可通过 `UpdateStatus` 方法只更新资源的 status 字段。

`Update` 更新资源时，资源的 `metadata.generation` 会增加；而 `UpdateStatus`不会更新该字段。

## 3.6 多版本支持
**TODO**

# 4. 小结
本文简要介绍了如何使用 CRD，如何编写一个 kubernetes controller，以及实践中的注意事项。
