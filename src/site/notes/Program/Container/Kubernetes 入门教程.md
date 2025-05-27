---
{"dg-publish":true,"permalink":"/Program/Container/Kubernetes 入门教程/","noteIcon":"","created":"2024-05-22T16:17:54.137+08:00"}
---




> 前言：本文是一篇 kubernetes（下文用 k8s 代替）的入门文章，将会涉及 k8s 的架构、集群搭建、一个 Redis 的例子，以及如何使用 operator-sdk 开发 operator 的教程。在文章过程中，会穿插引出 Pod、Deployment、StatefulSet 等 k8s 的概念，这些概念通过例子引出来，更容易理解和实践。文章参考了很多博客以及资料，放在最后参考资料部分。



## 一  k8s架构

  

![](/img/user/z-attchements/media/640.jpg)

我们看下 k8s 集群的架构，从左到右，分为两部分，第一部分是 Master 节点（也就是图中的 Control Plane），第二部分是 Node 节点。

  

Master 节点一般包括四个组件，apiserver、scheduler、controller-manager、etcd，他们分别的作用是什么：

  

*   Apiserver：上知天文下知地理，上连其余组件，下接ETCD，提供各类 api 处理、鉴权，和 Node 上的 kubelet 通信等，只有 apiserver 会连接 ETCD。
    

  

*   Controller-manager：控制各类 controller，通过控制器模式，致力于将当前状态转变为期望的状态。
    

  

*   Scheduler：调度，打分，分配资源。
    

  

*   Etcd：整个集群的数据库，也可以不部署在 Master 节点，单独搭建。
    

  

Node 节点一般也包括三个组件，docker，kube-proxy，kubelet

  

*   Docker：具体跑应用的载体。
    
*   Kube-proxy：主要负责网络的打通，早期利用 iptables，现在使用 ipvs技术。
    
*   Kubelet：agent，负责管理容器的生命周期。
    

  

总结一下就是 k8s 集群是一个由两部分组件 Master 和 Node 节点组成的架构，其中 Master 节点是整个集群的大脑，Node 节点来运行 Master 节点调度的应用，我们后续会以一个具体的调度例子来解释这些组件的交互过程。

  

## 二  搭建 k8s 集群

上面说完了 k8s 集群中有哪些组件，接下来我们先看下如何搭建一个 k8s 集群，有以下几种方法（参考文末链接）：


*   当我们安装了 Docker Desktop APP 之后，勾选 k8s 支持就能搭建起来。
*   使用 MiniKube 来搭建，社区提供的一键安装脚本。
*   直接在云平台购买，例如阿里云 ack。
*   使用 kubeadmin，这是 k8s 社区推荐的可以部署生产级别 k8s 的工具。
*   使用二进制，下载各组件安装，此教程需要注意，下载的各组件版本要和博客中保持一致，就可以成功。
    

  

本文后面的例子均采用本地 Docker Desktop APP 搭建的 k8s。

  

```css
➜  ~ kubectl version
Client Version: version.Info{Major:"1", Minor:"21", GitVersion:"v1.21.4", GitCommit:"3cce4a82b44f032d0cd1a1790e6d2f5a55d20aae", GitTreeState:"clean", BuildDate:"2021-08-11T18:16:05Z", GoVersion:"go1.16.7", Compiler:"gc", Platform:"darwin/amd64"}
Server Version: version.Info{Major:"1", Minor:"21", GitVersion:"v1.21.4", GitCommit:"3cce4a82b44f032d0cd1a1790e6d2f5a55d20aae", GitTreeState:"clean", BuildDate:"2021-08-11T18:10:22Z", GoVersion:"go1.16.7", Compiler:"gc", Platform:"linux/amd64"}
```

  

## 三  从需求出发

下面我们从一个实际的需求出发，来看看如何在 k8s 上部署 Redis 服务。

  

*   部署一个Redis服务
*   支持高可用
*   提供统一的 EndPoint 访问地址
    

  

---

### 1  部署单机版

  

如果我们想在 k8s 上部署一个单机版本 Redis，我们执行下面的命令即可：

  

```properties
➜  ~ kubectl run redis --image=redis
pod/redis created
➜  ~ kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE
redis   1/1     Running   0          5s
```

  

可以用 kubectl exec 来进入到 Pod 内部连接 Redis 执行命令：

  

```ruby
➜  ~ kubectl exec -it redis -- bash
root@redis:/data# redis-cli
127.0.0.1:6379> ping
PONG
127.0.0.1:6379>
```

  

那么 Pod 和 Redis 是什么关系呢？这里的 Redis 其实是一个 Docker 进程启动的服务，但是在 k8s 中，它叫 Pod。

  

### 2  Pod 与 Deployment

  

我们来讲下第一个 k8s 的概念 Pod，Pod 是 k8s 中最小的调度单元，一个 Pod 中可以包含多个 Docker，这些 Docker 都会被调度到同一台 Node 上，这些 Docker 共享 NetWork Namespace，并且可以声明共享同一个 Volume 来共享磁盘空间。

  

这样的好处是什么呢？其实在真实的世界中，很多应用是有部署在同一台机器的需求的，比如 Redis 日志采集插件要采集日志，肯定需要和 Redis 部署在同一台机器上才能读到 Redis 的日志，我们前面讲述背景的时候说到了 Docker Swarm 存在一些问题，其中之一就是它只是基于 Docker 调度，虽然也可以设置亲和度让两台 Docker 调度在同一个机器上，但是因为不能一起调度，所以会存在一个Docker 提前被调度到了一个资源少的机器上，从而导致第二个 Docker 调度失败。

  

例如我们一共有 2 台容器，A和B，分别为 Redis 和 日志采集组件，各需要 2g 内存，现在有两台 node，node1 3.5 内存，node2 4g内存，在 Docker Swarm 的调度策略下，先调度 Redis，有可能被调度到了 node1 上，接下来再来调度日志采集组件，发现 node1 只有 1.5g 内存了，调度失败。但是在 k8s 中，调度是按照 pod 来调度的，两个组件在一个 pod 中，调度就不会考虑 node1。

  

虽然 Pod 已经可以运行 Redis 服务了，但是他不具备高可用性，因为一旦一个 Pod 与一个节点（Node）绑定，除非这个绑定发生了变化（pod.spec.node 字段被修改），否则它永远都不会离开这个节点，这也就意味着，如果这个宿主机宕机了，这个 Pod 也不会主动迁移到其他节点上去。为了让服务可以一直在，需要使用 Deployment 这样的控制器。

  

```sql
➜  ~ kubectl create deployment redis-deployment --image=redis
deployment.apps/redis-deployment created
➜  ~ kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
redis                               1/1     Running   0          32m
redis-deployment-866c4c6cf9-8z8k5   1/1     Running   0          8s
➜  ~
```

  

redis-deployment-866c4c6cf9-8z8k5就是刚才通过 kubectl create 创建的新的 Deployment，为了验证高可用，我们把用 kubectl delete pod 把 redis 和 redis-deployment-866c4c6cf9-8z8k5都删掉看会发生什么。

  

```sql
➜  ~ kubectl delete pod redis redis-deployment-866c4c6cf9-8z8k5
pod "redis" deleted
pod "redis-deployment-866c4c6cf9-8z8k5" deleted
➜  ~ kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
redis-deployment-866c4c6cf9-zskkb   1/1     Running   0          10s
➜  ~
```


redis已经消失了，但是redis-deployment-866c4c6cf9-zskkb换了个名字又出现了！

Deployment 可以定义多副本个 Pod，从而为应用提供迁移能力，如果单纯使用 Pod，实际上当应用被调度到某台机器之后，机器宕机应用也无法自动迁移，但是使用 Deployment，则会调用 ReplicaSet（一种控制器） 来保证当前集群中的应用副本数和指定的一致。

  

### 3  k8s 使用 yaml 来描述命令

k8s 中，可以使用 kubectl 来创建简单的服务，但是还有一种方式是对应创建复杂的服务的，就是提供 yaml 文件。例如上面的创建 Pod 的命令，我们可以用下面的 yaml 文件替换，执行 kubectl create 之后，可以看到 redis Pod 又被创建了出来。

  

```properties
➜  ~ cat pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: redis
spec:
  containers:
  - name: redis
    image: redis
➜  ~ kubectl create -f pod.yaml
pod/redis created
➜  ~ kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
redis                               1/1     Running   0          6s
redis-deployment-866c4c6cf9-zskkb   1/1     Running   0          6m32s
```

  

## 四  k8s 组件调用流程

![](/img/user/z-attchements/media/640 1.png)

  
下面我们看下kubectl create deployment redis-deployment --image=redis下发之后，k8s 集群做了什么。
  

*   首先 controller-manager, scheduler, kubelet 都会和 apiserver 开始进行 List-Watch 模型，List 是拿到当前的状态，Watch 是拿到期望状态，然后 k8s 集群会致力于将当前状态达到达期望状态。
*   kubectl 下发命令到 apiserver，鉴权处理之后将创建信息存入 etcd，Deployment 的实现是使用 ReplicaSet 控制器，当 controller-manager 提前拿到当前的状态（pod=0），接着接收到期望状态，需要创建 ReplicaSet（pod=1），就会开始创建 Pod。
*   然后 scheduler 会进行调度，确认 Pod 被创建在哪一台 Node 上。
*   之后 Node 上的 kubelet 真正拉起一个 docker。
    

  

这些步骤中，apiserver 的作用是不言而喻的，所以说上接其余组件，下连 ETCD，但是 apiserver 是可以横向扩容的，然后通过负载均衡，倒是 ETCD 在 k8s 架构中成了瓶颈。

  

最开始看这架构的时候，会想着为啥 apiserver, scheduler, controller-manager 不合成一个组件，其实在 Google Borg 中，borgmaster 就是这样的，功能也是这些功能，但是合在了一起，最后他们也发现集群大了之后 borgmaster 会有些性能上的问题，包括 kubelet 的心跳就是很大一块，所以 k8s 从一开始开源，设计中有三个组件也是更好维护代码吧。

## 五  部署主从版本

上面我们已经部署了 Redis 的单机版，并通过 Deployment 实现了服务持续运行，接下来来看下主从版本如何部署，其中一个比较困难的地方就是如何确定主从的同步关系。

  

### 1  StatefulSet

  
k8s 为有状态应用设计了 StatefulSet 这种控制器，它主要通过下面两个特性来服务有状态应用：
  
*   拓扑状态：实例的创建顺序和编号是顺序的，会按照 name-index 来编号，比如 redis-0，redis-1 等。
*   存储状态：可以通过声明使用外部存储，例如云盘等，将数据保存，从而 Pod 重启，重新调度等都能读到云盘中的数据。
    

下面我们看下 Redis 的 StatefulSet 的例子：

  

```properties
apiVersion: apps/v1
kind: StatefulSet  # 类型为 statefulset
metadata:
  name: redis-sfs  # app 名称
spec:
  serviceName: redis-sfs  # 这里的 service 下面解释
  replicas: 2      # 定义了两个副本
  selector:
    matchLabels:
      app: redis-sfs
  template:
    metadata:
      labels:
        app: redis-sfs
    spec:
      containers:
      - name: redis-sfs 
        image: redis  # 镜像版本
        command:
          - bash
          - "-c"
          - |
            set -ex
            ordinal=`hostname | awk -F '-' '{print $NF}'`   # 使用 hostname 获取序列
            if [[ $ordinal -eq 0 ]]; then     # 如果是 0，作为主
              echo > /tmp/redis.conf
            else
              echo "slaveof redis-sfs-0.redis-sfs 6379" > /tmp/redis.conf # 如果是 1，作为备
            fi
            redis-server /tmp/redis.conf
```

  

接着启动这个 StatefulSet，发现出现了 redis-sfs-0 和 redis-sfs-1 两个 pod，他们正式按照 name-index 的规则来编号的

  

```sql
➜  ~ kubectl create -f server.yaml
statefulset.apps/redis-sfs created
➜  ~ kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
redis                               1/1     Running   0          65m
redis-deployment-866c4c6cf9-zskkb   1/1     Running   0          71m
redis-sfs-0                         1/1     Running   0          33s  # 按照 
redis-sfs-1                         1/1     Running   0          28s
```

  

接着我们继续看下主从关系生效了没，查看 redis-sfs-1 的日志，却发现：

  

```css
➜  ~ kubectl logs -f redis-sfs-1
1:S 05 Nov 2021 08:02:44.243 * Connecting to MASTER redis-sfs-0.redis-sfs:6379
1:S 05 Nov 2021 08:02:50.287 # Unable to connect to MASTER: Resource temporarily unavailable
...
```

### 2  Headless Service

  

似乎 redis-sfs-1 不认识 redis-sfs-0，原因就在于我们还没有让它们互相认识，这个互相认识需要使用 k8s 一个服务叫 Headless Service，Service 是 k8s 项目中用来将一组 Pod 暴露给外界访问的一种机制。比如，一个 Deployment 有 3 个 Pod，那么我就可以定义一个 Service。然后，用户只要能访问到这个 Service，它就能访问到某个具体的 Pod，一般有两种方式：
  
*   VIP：访问 VIP 随机返回一个后端的 Pod
*   DNS：通过 DNS 解析到后端某个 Pod 上
    
Headless Service 就是通过 DNS 的方式，可以解析到某个 Pod 的地址，这个 DNS 地址的规则就是：

  

```css
<pod-name>.<svc-name>.<namespace>.svc.cluster.local
```
  
下面我们创建集群对应的 Headless Service：

```properties
apiVersion: v1
kind: Service
metadata:
  name: redis-sfs
  labels:
    app: redis-sfs
spec:
  clusterIP: None   # 这里的 None 就是 Headless 的意思，表示会主动由 k8s 分配
  ports:
    - port: 6379
      name: redis-sfs
  selector:
    app: redis-sfs
```

  
再次查看，发现 redis-sfs-1 已经主备同步成功了，因为创建 Headless Service 之后，redis-sfs-0.redis-sfs.default.svc.cluster.local 在集群中就是唯一可访问的了。
  

```sql
➜  ~ kubectl create -f service.yaml
service/redis-sfs created
➜  ~ kubectl get service
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP    24d
redis-sfs    ClusterIP   None         <none>        6379/TCP   33s
➜  ~ kubectl logs -f redis-sfs-1
...
1:S 05 Nov 2021 08:23:31.341 * Connecting to MASTER redis-sfs-0.redis-sfs:6379
1:S 05 Nov 2021 08:23:31.345 * MASTER <-> REPLICA sync started
1:S 05 Nov 2021 08:23:31.345 * Non blocking connect for SYNC fired the event.
1:S 05 Nov 2021 08:23:31.346 * Master replied to PING, replication can continue...
1:S 05 Nov 2021 08:23:31.346 * Partial resynchronization not possible (no cached master)
1:S 05 Nov 2021 08:23:31.348 * Full resync from master: 29d1c03da6ee2af173b8dffbb85b6ad504ccc28f:0
1:S 05 Nov 2021 08:23:31.425 * MASTER <-> REPLICA sync: receiving 175 bytes from master to disk
1:S 05 Nov 2021 08:23:31.426 * MASTER <-> REPLICA sync: Flushing old data
1:S 05 Nov 2021 08:23:31.426 * MASTER <-> REPLICA sync: Loading DB in memory
1:S 05 Nov 2021 08:23:31.431 * Loading RDB produced by version 6.2.6
1:S 05 Nov 2021 08:23:31.431 * RDB age 0 seconds
1:S 05 Nov 2021 08:23:31.431 * RDB memory usage when created 1.83 Mb
1:S 05 Nov 2021 08:23:31.431 # Done loading RDB, keys loaded: 0, keys expired: 0.
1:S 05 Nov 2021 08:23:31.431 * MASTER <-> REPLICA sync: Finished with success
^C
➜  ~ kubectl exec -it redis-sfs-1 -- bash
root@redis-sfs-1:/data# redis-cli -h redis-sfs-0.redis-sfs.default.svc.cluster.local
redis-sfs-0.redis-sfs.default.svc.cluster.local:6379> ping
PONG
redis-sfs-0.redis-sfs.default.svc.cluster.local:6379>
```

  

此时无论我们删除哪个 Pod，它都会按照原来的名称被拉起来，从而可以保证准备关系，这个例子只是一个 StatefulSet 的示例，分析下来可以发现，虽然它可以维护主备关系，但是当主挂了的时候，此时备无法切换上来，因为没有组件可以帮我们做这个切换操作，一个办法是用 Redis Sentinel，可以参考这个项目的配置：k8s-redis-ha-master，如果你的 k8s 较新，需要 merge 此 PR.

  


## 六  Operator

虽然有了 StatefulSet，但是这只能对基础版有用，如果想自己定制更加复杂的操作，k8s 的解法是 operator，简而言之，operator 就是定制自己 k8s 对象及对象所对应操作的解法。
  
那什么是对象呢？一个 Redis 集群，一个 etcd 集群，zk 集群，都可以是一个对象，现实中我们想描述什么，就来定义什么，实际上我们定一个是k8s yaml 中的 kind，之前的例子中，我们使用过 Pod，Deployment，StatefulSet，它们是 k8s 默认实现，现在如果要定义自己的对象，有两个流程：

*   定义对象，比如你的集群默认有几个节点，都有啥组件
*   定义对象触发的操作，当创建对象时候要做什么流程，HA 时候要做什么流程等
    


operator 的方式是基于编程实现的，可以用多种语言，用的最多的就是 go 语言，通常大家会借助 operator-sdk 来完成，因为有很多代码会自动生成。相当于 operator 会生成框架，然后我们实现对应的业务逻辑。
  


### 1  准备工作


*   安装好 go 环境
*   安装 operator-sdk
    



### 2  初始化项目

  

然后我们按照官网的 sdk 例子，来一步一步实现一个 memcached 的 operator，这里也可以换成 Redis，但是为了保证和官网一致，我们就按照官网来创建 memcached operator。

  

```kotlin
➜  ~ cd $GOPATH/src
➜  src mkdir memcached-operator
➜  src cd memcached-operator
➜  memcached-operator operator-sdk init --domain yangbodong22011 --repo github.com/yangbodong22011/memcached-operator --skip-go-version-check // 这里需要注意 domain 最好是和你在 https://hub.docker.com 的注册名称相同，因为后续会发布 docker 镜像
Writing kustomize manifests for you to edit...
Writing scaffold for you to edit...
Get controller runtime:
$ go get sigs.k8s.io/controller-runtime@v0.9.2
Update dependencies:
$ go mod tidy
Next: define a resource with:
$ operator-sdk create api
```

  

### 3  创建 API 和 Controller

  

```bash
➜  memcached-operator operator-sdk create api --group cache --version v1alpha1 --kind Memcached --resource --controller
Writing kustomize manifests for you to edit...
Writing scaffold for you to edit...
api/v1alpha1/memcached_types.go
controllers/memcached_controller.go
Update dependencies:
$ go mod tidy
Running make:
$ make generate
go: creating new go.mod: module tmp
Downloading sigs.k8s.io/controller-tools/cmd/controller-gen@v0.6.1
go get: installing executables with 'go get' in module mode is deprecated.
    To adjust and download dependencies of the current module, use 'go get -d'.
    To install using requirements of the current module, use 'go install'.
    To install ignoring the current module, use 'go install' with a version,
    like 'go install example.com/cmd@latest'.
    For more information, see https://golang.org/doc/go-get-install-deprecation
    or run 'go help get' or 'go help install'.
...
go get: added sigs.k8s.io/yaml v1.2.0
/Users/yangbodong/go/src/memcached-operator/bin/controller-gen object:headerFile="hack/boilerplate.go.txt" paths="./..."
➜  memcached-operator
```

上面的步骤实际上生成了一个 operator 的框架，接下来我们首先来定义 memcached 集群都包括啥，将默认实现修改为 Size，表示一个 Memcached 集群中 Memcached 的数量，最后调用 make generate 和 make manifests 来自动生成 deepcopy 和 CRD 资源。

  

```cpp
➜  memcached-operator vim api/v1alpha1/memcached_types.go // 修改下面 Memcached 集群的定义
// MemcachedSpec defines the desired state of Memcached
type MemcachedSpec struct {
    //+kubebuilder:validation:Minimum=0
    // Size is the size of the memcached deployment
    Size int32 `json:"size"`
}

// MemcachedStatus defines the observed state of Memcached
type MemcachedStatus struct {
    // Nodes are the names of the memcached pods
    Nodes []string `json:"nodes"`
}

➜  memcached-operator make generate
/Users/yangbodong/go/src/memcached-operator/bin/controller-gen object:headerFile="hack/boilerplate.go.txt" paths="./..."
➜  memcached-operator make manifests
/Users/yangbodong/go/src/memcached-operator/bin/controller-gen "crd:trivialVersions=true,preserveUnknownFields=false" rbac:roleName=manager-role webhook paths="./..." output:crd:artifacts:config=config/crd/bases
➜  memcached-operator
```

  

### 4  实现 Controller

  

接下来是第二步，定义当创建一个 Memcached 集群时候，具体要干啥。

  

```go
➜  memcached-operator vim controllers/memcached_controller.go

https://raw.githubusercontent.com/operator-framework/operator-sdk/latest/testdata/go/v3/memcached-operator/controllers/memcached_controller.go //
将 example 换成 yangbodong22011，注意，// 注释中的也要换，实际不是注释，而是一种格式


➜  memcached-operator go mod tidy; make manifests
/Users/yangbodong/go/src/memcached-operator/bin/controller-gen "crd:trivialVersions=true,preserveUnknownFields=false" rbac:roleName=manager-role webhook paths="./..." output:crd:artifacts:config=config/crd/bases
```

  

### 5  发布 operator 镜像

  

```cpp
➜  memcached-operator vim Makefile
将 -IMG ?= controller:latest 改为 +IMG ?= $(IMAGE_TAG_BASE):$(VERSION)

➜  memcached-operator docker login  // 提前登录下 docker
Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
Username: yangbodong22011
Password:
WARNING! Your password will be stored unencrypted in /Users/yangbodong/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
➜  memcached-operator sudo make docker-build docker-push 
...
=> => writing image sha256:a7313209e321c84368c5cb7ec820fffcec2d6fcb510219d2b41e3b92a2d5545a                                                             0.0s
 => => naming to docker.io/yangbodong22011/memcached-operator:0.0.1                                                                                      0.0s
fac03a24e25a: Pushed
6d75f23be3dd: Pushed
0.0.1: digest: sha256:242380214f997d98186df8acb9c13db12f61e8d0f921ed507d7087ca4b67ce59 size: 739
```

  
  

### 6  修改镜像和部署

  

```cs
➜  memcached-operator vim config/manager/manager.yaml
image: controller:latest 修改为 yangbodong22011/memcached-operator:0.0.1

➜  memcached-operator vim config/default/manager_auth_proxy_patch.yaml
因为国内访问不了 gcr.io
image: gcr.io/kubebuilder/kube-rbac-proxy:v0.8.0 修改为 kubesphere/kube-rbac-proxy:v0.8.0 


➜  memcached-operator make deploy
...
configmap/memcached-operator-manager-config created
service/memcached-operator-controller-manager-metrics-service created
deployment.apps/memcached-operator-controller-manager created

➜  memcached-operator kubectl get deployment -n memcached-operator-system // ready 说明 operator 已经部署了
NAME                                    READY   UP-TO-DATE   AVAILABLE   AGE
memcached-operator-controller-manager   1/1     1            1           31s
➜  memcached-operator
```

  

### 7  创建 Memcached 集群

  

```sql
➜  memcached-operator cat config/samples/cache_v1alpha1_memcached.yaml
apiVersion: cache.yangbodong22011/v1alpha1
kind: Memcached
metadata:
  name: memcached-sample
spec:
  size: 1
➜  memcached-operator kubectl apply -f config/samples/cache_v1alpha1_memcached.yaml
memcached.cache.yangbodong22011/memcached-sample created
➜  memcached-operator kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
memcached-sample-6c765df685-xhhjc   1/1     Running   0          104s
redis                               1/1     Running   0          177m
redis-deployment-866c4c6cf9-zskkb   1/1     Running   0          3h4m
redis-sfs-0                         1/1     Running   0          112m
redis-sfs-1                         1/1     Running   0          112m
➜  memcached-operator
```

  

可以通过 kubectl logs 来查看 operator 的日志：

  

```cs
➜  ~ kubectl logs -f deployment/memcached-operator-controller-manager -n memcached-operator-system
2021-11-05T09:50:46.042Z    INFO    controller-runtime.manager.controller.memcached    Creating a new Deployment    {"reconciler group": "cache.yangbodong22011", "reconciler kind": "Memcached", "name": "memcached-sample", "namespace": "default", "Deployment.Namespace": "default", "Deployment.Name": "memcached-sample"}
```

  

至此，我们的 operator-sdk 的任务暂时告一段落。

  

## **七  总结**

本文介绍了 k8s 的架构，各组件的功能，以及通过一个循序渐进的 Redis 例子介绍了 k8s 中 Pod, Deployment, StatefulSet 的概念，并通过 operator-sdk 演示了一个完整的 operator制作的例子。

  

## **八  参考资料**

\[1\] 《深入剖析Kubernetes》张磊，CNCF TOC 成员，at 阿里巴巴。  
\[2\] 《Kubernetes 权威指南》第五版  
\[3\] 《Large-scale cluster management at Google with Borg》

https://research.google/pubs/pub43438/

\[4\] https://www.redhat.com/zh/topics/containers/what-is-kubernetes?

\[5\] https://www.infoworld.com/article/3632142/how-docker-broke-in-half.html?

\[6\] https://landscape.cncf.io/

\[7\] https://docs.docker.com/desktop/kubernetes/

\[8\] https://minikube.sigs.k8s.io/docs/start/

\[9\] https://www.aliyun.com/product/kubernetes?

\[10\] https://github.com/kubernetes/kubeadm

\[11\] https://www.cnblogs.com/chiangchou/p/k8s-1.html

\[12\] https://github.com/tarosky/k8s-redis-ha

\[13\] https://sdk.operatorframework.io/docs/installation/

  