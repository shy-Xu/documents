## 背景介绍
在专有云领域，sealer可以帮助用户构建出他们自己想要的镜像（sealer称之为"CloudImage"），创建出用户专属的k8s集群。在构建CloudImage的过程中，sealer会帮用户把需要的所有依赖打包，以使得构建出来的CloudImage可以在任何场景中使用。这些依赖其中就包括一些不同CPU架构的docker镜像，helm chart包等，sealer把它们存储到构建出来的CloudImage的私有仓库中，它们之间的关系如下图所示：
![image](https://user-images.githubusercontent.com/53456509/149660147-b220816d-a4bc-4f5a-85f8-dae026d251f2.png)

使用构建好的CloudImage成功启动一个k8s集群后，docker镜像，helm chart包已经存在于私有仓库中，这一方面提升了镜像拉取效率，另一方面减少了与外部网络的通信，保障了集群的安全性。
## image save模式出现之前
一开始，sealer在build 时选择了依赖docker的方式来将镜像存储在CloudImage中，build之前要有运行中的docker daemon进程。在sealer内部，调用docker的sdk来拉取所需的镜像。其工作流程如下图所示：

图中所使用的docker daemon是sealer修改过的，sealer修改后的docker daemon把sea.hub做为代理仓库，所有镜像的拉取都会先拉取到代理仓库sea.hub中，然后再从sea.hub仓库拉取到本地。sea.hub实质上是一个docker容器，其存储镜像的目录与CloudImage的文件系统是相通的。所以，此种情况下，sealer只用执行一个动作就可以把镜像缓存到CloudImage中。
在实际的应用场景中，有些集群使用原生的docker做为容器运行时，此时，sealer在build过程中的工作流程如下图所示：

在原生docker的场景下，sea.hub不在具有代理仓库的角色，仅仅具有普通私有仓库的功能。因此，所有的镜像需要先执行一次拉取动作，保存镜像到本地文件系统中，然后再执行一次推送动作，把镜像存储到sea.hub中，也即存储到CloudImage中。

该存储方式的逻辑原理比较简单，但是也有很多的弊端存在：
1. 依赖过多。需要有运行中的docker daemon，需要有处于运行状态的registry容器。
1. 要适应不同的情况。要针对sealer docker的情景和原生docker的情景做不同的处理，且未来出现新的情景时还可能需要增添对新情景的处理，无法做到容器运行时的统一。
1. 效率低下。其实sealer的最终目的是要把镜像存储到CloudImage中，但无论是sealer docker的场景还是原生docker的场景，整个流程中都有一些不必要的额外步骤。
1. 扩展性不好。不能够支持其他的符合OCI标准的镜像，也不能支持helm chart包。

## image save模式

### 工作流程总览

随着sealer应用场景的增加，用户需求的增多，原有的将镜像存储到CloudImage的方式显得稍有臃肿。sealer希望通过一个简洁而又高效的设计统一所有的应用场景，优化镜像存储到CloudImage的效率。image save模式的工作流程如下图所示：

sealer扮演一个客户端的角色，通过Restful API接口与远端的镜像仓库建立http通信，请求镜像数据。然后内部实现了镜像存储逻辑，将镜像仓库响应的镜像数据直接存储到CloudImage中。该种设计极大的简化了镜像存储逻辑，克服了原有模式的所有弊端。

### OCI镜像介绍
当我们要从一个符合OCI标准的仓库中拉取镜像时，首先要与对应registry的对应repository建立链接，然后根据tag从repository中获取manifest list。manifest list中包含了该tag对应的所有Platform的镜像digest。下图是一个manifest list的部分内容：

选取想要的Platform对应的digest，用该digest发起一个http GET请求，获取到指定镜像的manifest，镜像的manifest中包含了所有layer的digest和config的digest。下图是一个manifest的部分内容：

最后，发起一系列的http GET请求，获取仓库中所有的digest对应的blob，并存储到本地。


### 核心代码实现
