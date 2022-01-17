## 背景介绍
在专有云领域，sealer可以帮助用户构建出他们自己想要的镜像（sealer称之为"CloudImage"），创建出用户专属的k8s集群。在构建CloudImage的过程中，sealer会帮用户把需要的所有依赖打包，以使得构建出来的CloudImage可以在任何场景中使用。这些依赖其中就包括一些不同CPU架构的docker镜像，helm chart包等，sealer把它们存储到构建出来的CloudImage的私有仓库中，它们之间的关系如下图所示：
![image](https://user-images.githubusercontent.com/53456509/149660147-b220816d-a4bc-4f5a-85f8-dae026d251f2.png)

使用构建好的CloudImage成功启动一个k8s集群后，docker镜像，helm chart包已经存在于私有仓库中，这一方面提升了镜像拉取效率，另一方面减少了与外部网络的通信，保障了集群的安全性。
## image save模式出现之前
一开始，sealer在build 时选择了依赖docker的方式来将镜像存储在CloudImage中，build之前要有运行中的docker daemon进程。在sealer内部，调用docker的sdk来拉取所需的镜像。其工作流程如下图所示：

![image](https://user-images.githubusercontent.com/53456509/149660455-edda865b-eb97-408a-ba5a-4f9615b28f76.png)

图中所使用的docker daemon是sealer修改过的，sealer修改后的docker daemon把sea.hub做为代理仓库，所有镜像的拉取都会先拉取到代理仓库sea.hub中，然后再从sea.hub仓库拉取到本地。sea.hub实质上是一个docker容器，其存储镜像的目录与CloudImage的文件系统是相通的。所以，此种情况下，sealer只用执行一个动作就可以把镜像缓存到CloudImage中。
在实际的应用场景中，有些集群使用原生的docker做为容器运行时，此时，sealer在build过程中的工作流程如下图所示：

![image](https://user-images.githubusercontent.com/53456509/149660482-1781178c-4387-48f1-9a5f-9e62971b6001.png)

在原生docker的场景下，sea.hub不在具有代理仓库的角色，仅仅具有普通私有仓库的功能。因此，所有的镜像需要先执行一次拉取动作，保存镜像到本地文件系统中，然后再执行一次推送动作，把镜像存储到sea.hub中，也即存储到CloudImage中。

原生docker的场景相比sealer docker场景，还有另外一个问题：建立起k8s集群后，用户所创建的pod中容器所使用的镜像可能并不是CloudImage内置的私有仓库的镜像。例如，在一个pod的yaml文件中声明改pod使用`mysql:latest`镜像，本意是想使用sea.hub/library/mysql:latest 镜像，实际情况下使用的镜像是docker.io/library/mysql:latest 。针对这个问题，sealer在以原生docker做为容器运行时的CloudImage中整合了webhook技术，来控制pod中镜像的来源，sealer所选择的webhook工具是kyverno，kyverno在k8s集群中的角色是一个动态准许加入控制器(dynamic admission controller)，它可以拦截到k8s集群中组件与kube-apiserver之间的通信，同时对k8s集群管理者暴露了一组规则和策略，通过自定义规则和策略，达到动态控制集群资源的效果。

![DD557B1F-FA9F-439f-A184-4B96AD6E2CDF](https://user-images.githubusercontent.com/53456509/149702962-e2b2b23e-638f-44b2-8f77-8e58b239b032.png)

kyverno暴露的策略主要包括两种，一种是修改策略，一种是验证策略。k8s集群管理者通过验证策略来拒绝不符合要求的资源加入k8s集群中，通过修改策略来控制k8s集群的资源符合自己的需求。sealer使用的是kyverno中的修改策略。具体如下：

```
  apiVersion : kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: redirect-registry
spec:
  background: false
  rules:
  - name: prepend-registry-containers
    match:
      resources:
        kinds:
        - Pod
    mutate:
      foreach:
      - list: "request.object.spec.containers"
        patchStrategicMerge:
          spec:
            containers:
            - name: "{{ element.name }}"
              image: "sea.hub:5000/{{ images.containers.{{element.name}}.path}}:{{images.containers.{{element.name}}.tag}}"
  - name: prepend-registry-initcontainers
    match:
      resources:
        kinds:
        - Pod
    mutate:
      foreach:
      - list: "request.object.spec.initContainers"
        patchStrategicMerge:
          spec:
            initContainers:
            - name: "{{ element.name }}"
              image: "sea.hub:5000/{{ images.initContainers.{{element.name}}.path}}:{{images.initContainers.{{element.name}}.tag}}"
```

该策略匹配所有的pod，对于每一个kind字段的值是pod的请求，修改其中的spec.containers字段和spec.initContainers字段，将image的路径中registry值修改为sea.hub:5000，也即CloudImage中的私有仓库的URL地址。因此，用户所创建的所有pod，在kyverno的控制下，都会使用私有仓库中的镜像。

## image save模式

原有的镜像缓存模式逻辑原理比较简单，容易实现。但是也有很多的弊端存在：
1. 依赖过多。需要有运行中的docker daemon，需要有处于运行状态的registry容器。
1. 要适应不同的情况。要针对sealer docker的情景和原生docker的情景做不同的处理，且未来出现新的情景时还可能需要增添对新情景的处理，无法做到容器运行时的统一。
1. 效率低下。其实sealer的最终目的是要把镜像存储到CloudImage中，但无论是sealer docker的场景还是原生docker的场景，整个流程中都有一些不必要的额外步骤。
1. 扩展性不好。不能够支持其他的符合OCI标准的镜像，也不能支持helm chart包。

针对以上问题，sealer演进出了一种新的镜像缓存模式，给用户提供更加优良的体验。
### 工作流程总览

随着sealer应用场景的增加，用户需求的增多，原有的将镜像存储到CloudImage的方式显得稍有臃肿。sealer希望通过一个简洁而又高效的设计统一所有的应用场景，优化镜像存储到CloudImage的效率。image save模式的工作流程如下图所示：

![image](https://user-images.githubusercontent.com/53456509/149660641-8fc54011-cf1d-4fce-add4-c2d63f748337.png)

sealer扮演一个客户端的角色，通过Restful API接口与远端的镜像仓库建立http通信，请求镜像数据。然后内部实现了镜像存储逻辑，将镜像仓库响应的镜像数据直接存储到CloudImage中。该种设计极大的简化了镜像存储逻辑，克服了原有模式的所有弊端。

### OCI镜像介绍
当我们要从一个符合OCI标准的仓库中拉取镜像时，首先要与对应registry的对应repository建立链接，然后根据tag从repository中获取manifest list。manifest list中包含了该tag对应的所有Platform的镜像digest。下图是一个manifest list的部分内容：

![image](https://user-images.githubusercontent.com/53456509/149660663-bd30d612-4270-4493-b263-9ed3f1ec2d17.png)

选取想要的Platform对应的digest，用该digest发起一个http GET请求，获取到指定镜像的manifest，镜像的manifest中包含了所有layer的digest和config的digest。下图是一个manifest的部分内容：

![image](https://user-images.githubusercontent.com/53456509/149660674-3152f910-37a5-41da-9b9f-991cf8a4d520.png)

最后，发起一系列的http GET请求，获取仓库中所有的digest对应的blob，并存储到CloudImage中。

注意，docker镜像和OCI镜像的manifest list的mediaType值以及manifest 的mediaType值并不相同，出于兼容性的考虑，sealer对两者都提供了支持，他们之间的区别如下:

#### OCI使用的mediaType

```
manifest list mediaType： application/vnd.oci.image.index.v1+json
manifest mediaType: application/vnd.oci.image.manifest.v1+json
```

#### docker使用的mediaType

```
manifest list mediaType: application/vnd.docker.distribution.manifest.list.v2+json
manifest mediaType: application/vnd.docker.distribution.manifest.v2+json
```

此外，sealer对于mediaType为空但其余部分符合OCI规定的镜像标准的镜像也提供支持，例如helm chart包，下图是一个helm chart包的manifest示例：

![image](https://user-images.githubusercontent.com/53456509/149661231-092a0a9f-b278-4a20-8ef8-0e9dd71790b6.png)

仔细观察，图中虽然有两个mediaType字段，但并不是manifest的mediaType值，而是两个blob的mediaType值。对于mediaType为空的情况，sealer会首先判断其是否是一个包含了多个manifest的list，然后获取相应的manifest的digest，进而获取所有blob的digest，最后存储blob数据到CloudImage中。

### 核心代码实现

image save模块的功能在实现时，主要参考了开源项目 https://github.com/distribution/distribution 的实现原理，将distribution中关于镜像的拉取逻辑的代码和镜像的存储逻辑的代码分离出来，加以修改然后进行复用。首先是registry结构体，该结构体的定义如下：

```
type proxyingRegistry struct {
	embedded       distribution.Namespace // provides local registry functionality
	scheduler      *scheduler.TTLExpirationScheduler
	remoteURL      url.URL
	authChallenger authChallenger
}
```
其中，`remoteURL`是远端仓库的地址，`authChallenger`用于与远端仓库建立链接时候的认证，`scheduler`并没有在sealer中使用，`embedded`是一个`distribution.Namespace`接口类型的对象，在sealer中起到存储镜像数据的作用。
`distribution.Namespace`接口类型的详细定义如下：
```
type Namespace interface {
	Scope() Scope

	Repository(ctx context.Context, name reference.Named) (Repository, error)

	Repositories(ctx context.Context, repos []string, last string) (n int, err error)

	Blobs() BlobEnumerator

	BlobStatter() BlobStatter
}
```
`Repository`函数接收两个参数，一个是`context.Context`类型，代表当前运行环境上下文，一个是`reference.Named`类型，代表`repository`的名字，例如: `library/ubuntu`,`Repository`函数能够根据`reference.Named`参数的值返回相应的`distribution.Repository`接口类型的对象，对于`proxyingRegistry`这个结构体对象而言，它调用`Repository`函数返回的对象其实是一个名为`proxiedRepository`的结构体，如下所示：

```
type proxiedRepository struct {
	blobStore distribution.BlobStore
	manifests distribution.ManifestService
	name      reference.Named
	tags      distribution.TagService
}
```

其中，`name`就是该`repository`的名字，而`tags`实现了接下来存储镜像数据时对`tag`的处理逻辑,`manifests`,`blobStore`分别实现了接下来存储镜像数据时对`manifest`,`blob`的处理逻辑。sealer对于它们的使用方式大致相同，因此选取`manifest`处理逻辑进行说明，`tag`和`blob`的处理逻辑可类比`manifest`。

sealer中，`manifest`处理代码的核心代码如下：

```
manifest, err := localManifests.Get(ctx, dgst, options...)
	if err != nil {
		if err := authChallenger.tryEstablishChallenges(ctx); err != nil {
			return nil, err
		}

		manifest, err = remoteManifests.Get(ctx, dgst, options...)
		if err != nil {
			return nil, err
		}
		fromRemote = true
	}

	if fromRemote {
		_, err = localManifests.Put(ctx, manifest)
		if err != nil {
			return nil, err
		}
  }
  
  return manifest, err
```

首先，尝试从本地文件系统中获取digest对应的manifest，如果成功得话，就证明本地已经存在该manifest，不会再去向远端仓库发起http Get请求，直接返回得到得manifest。如果失败，则首先与远端仓库进行认证，认证成功之后，发起http Get请求，从远端仓库中获取digest对应的manifest，然后把`fromRemote`这个bool变量的值置为`true`。接下来，会进入由`fromRemote`做为判断条件的if语句块的内部，把获取到的manifest存储到本地文件系统中。


