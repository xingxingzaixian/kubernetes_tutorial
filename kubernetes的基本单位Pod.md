      Pod 是 kubernetes 中的基本单位，容器本身不会直接分配到主机上，而是会封装到 Pod 对象中。一个 Pod 通常表示单个应用程序，有一个或者多个相关的容器组成，这些容器的生命周期都是相同的，而且会作为一个整体在同一个 node 上调度起来，这些容器共享环境、存储卷和 IP 控件。尽管 Pod 中可能存在多个容器，但是在 kubernetes 中是以 Pod 为最小单位进行调度、伸缩并共享资源、管理生命周期。

# Pod 的基本操作

​      我们先来看一下 Pod 的创建、查询、修改和删除操作。

## 创建 Pod

```yaml
# expod.yml

apiVersion: v1
kind: Pod
metadata:
  name: expod
spec:
  containers:
  - name: expod-container
    image: busybox
    imagePullPolicy: IfNotPresent
    command: ['sh', '-c']
    args: ['echo "Hello Kubernetes!"; sleep 3600']
```

简单的模板含义：

- apiVersion 表示 API 版本，v1 表示使用 kubernetes API 的稳定版本。
- kind 表示要创建的资源对象。
- metadata 表示该资源对象的元数据。可以拥有多个元数据，name 表示当前资源的名称。
- spec 表示该资源对象的具体设置。其中 containers 表示容器的集合，我们这里设置了一个简单的容器。
  - name: 要创建的容器名称。
  - image: 容器的镜像地址。
  - imagePullPolicy: 镜像的下载策略，支持3种策略：Always、Never、IfNotPresent。
  - command: 容器的启动命令列表，不配置的话就使用镜像内部的命令。
  - args: 启动参数列表

运行命令，创建 Pod。

```shell
kubectl apply -f expod.yml
```

创建成功后，查询一下当前运行的所有 Pod

```shell
kubectl get pod
```

![image-20201004205558756](https://tva1.sinaimg.cn/large/007S8ZIlly1gjdliw0lafj308601qq2w.jpg)

## 查询 Pod

Pod 信息查询的命令有多个，查询的详细度也不一样：

- 查询默认命名空间 default 的所有 Pod

  ```shell
  kubectl get pod
  ```

- 查询指定 Pod 的信息

  ```shell
  kubectl get pod expod
  ```

- 对 Pod 状态进行持续监控

  ```shell
  kubectl get pod -w
  ```

- 显示更多概要信息

  ```shell
  kubectl get pod -o wide
  ```

  ![image-20201004210238444](https://tva1.sinaimg.cn/large/007S8ZIlly1gjdlprtnd5j30js01pq30.jpg)

  比简要信息多显示集群内 IP 地址、所属 node

- 按照指定格式输出 Pod 信息

  ```shell
  kubectl get pod expod -o yaml
  ```

- 最详细信息显示，包括 Event

  ```shell
  kubectl describe pod expod
  ```

  这是最常用的资源信息查询命令，显示信息比较全面，包括资源的基本信息、容器信息、准备情况、存储卷信息和相关的事件列表。在资源部署的时候，如果遇到问题，可以用这个命令查询详情，分析错误原因。

- 查询Pod的日志信息

  ```shell
  kubectl logs Pod名称
  ```

## 修改 Pod

修改已存在的 Pod 属性可以使用 replace 命令

```shell
kubectl replace -f pod的yaml文件
```

我们修改一下前面创建 Pod 的 yaml 文件，把输出 “Hello Kubernetes!" 修改为 "Hello Kubernetes replaced!"。

```yaml
# expod.yml

apiVersion: v1
kind: Pod
metadata:
  name: expod
spec:
  containers:
  - name: expod-container
    image: busybox
    imagePullPolicy: IfNotPresent
    command: ['sh', '-c']
    args: ['echo "Hello Kubernetes replaced!"; sleep 3600']
```

Pod 有很多属性是无法修改的，如果一定要修改，需要加上 --force 参数，相当于重建 Pod。

```shell
kubectl replace -f expod.yml --force
```

可以看一下命令输出结果

![image-20201004212102053](https://tva1.sinaimg.cn/large/007S8ZIlly1gjdm8x30upj30cg01smx4.jpg)

先删除了 expod 这个 Pod，然后创建一个替换的 expod 的 Pod，我们再查看以下 Pod 的日志。

```shell
kubectl logs expod
```

![image-20201004212241007](https://tva1.sinaimg.cn/large/007S8ZIlly1gjdmamv8thj308m015q2t.jpg)

可以看到 Pod 信息已经修改了。

## 删除 Pod

删除 Pod 非常简单，执行以下命令即可：

```shell
kubectl delete pod expod
```

如果我们是通过 Pod 模板文件创建的，推荐使用基于模板文件的删除命令。

```shell
kubectl delete -f expod.yml
```

# Pod 与容器

我们可能已经发现了，Pod 模板和 Docker-Compose 非常相似，但是 Pod 模板可以配置的参数更多、更复杂。

## Pod 创建容器的方式

在 Pod 模板的 Containers 部分，指明容器的部署方式，在部署的过程中会转换成对应的容器运行命令，就以我们最开始的 Pod 模板为例:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: expod
spec:
  containers:
  - name: expod-container
    image: busybox
    imagePullPolicy: IfNotPresent
    command: ['sh', '-c']
    args: ['echo "Hello Kubernetes!"; sleep 3600']
```

在 kubernetes 进行调度的时候，会执行如下命令：

```shell
docker run --name expod-container busybox sh -c 'echo "Hello Kubernetes!"; sleep 3600'
```

command 和 args 的设置会分别覆盖 Docker 镜像中定义的 EntryPoint 与 CMD。

## Pod 组织容器的方式

​        Pod 是由各个容器组成的一个整体，同时调度到某台 Node 上，容器之间可以共享资源、网络环境和依赖，并拥有相同的生命周期。

### 容器如何组成一个 Pod

​      每个 Pod 都包含一个或一组密切相关的业务容器，但是还有一个成为”根容器“的特殊 Pause 容器。Pause 容器是属于 Kubernetes 的一部分，如果一组业务容器作为一个整体，我们很难对整个容器进行判断，假如一个业务组容器当中的某个容器挂载了能代表整个 Pod 都挂载了吗？如果引入一个和业务无关的 Pause 容器，用它作为 Pod 的根容器，用它的状态代表整组容器的状态，就能解决这个问题。而且一个 Pod 中的所有容器都共享 Pause 容器的 IP 地址及其挂载的存储卷，这样也简化了容器之间的通信和数据共享问题，类似于 Docker 容器网络的 Container 模式。

​      我们在开始创建的 Pod，可以登录上对应的 Node 机器，查看容器信息。

```shell
kubectl get pod -o wide
```

![image-20201004231006469](https://tva1.sinaimg.cn/large/007S8ZIlly1gjdpeer9x5j30jt01uq32.jpg)

登录 k8s-node2 以后，执行 `docker ps` 命令查看详细信息：

![image-20201004231345216](https://tva1.sinaimg.cn/large/007S8ZIlly1gjdpi70xqyj316u04kq4j.jpg)

可以看到有三个 pause 容器，其中两个是 flannel 和 proxy 容器，还有一个是我们的 expod 的容器，它与另一个 expod 容器共同组成了一个 Pod。

Pod 中的容器共享两种资源-存储和网络。

- 存储

  在 Pod 里面，我们可以指定一个或者多个存储卷，Pod 中的所有容器都可以访问这些存储卷，存储卷可以分为临时和持久化两种。

  ![容器和存储卷的关系](https://tva1.sinaimg.cn/large/007S8ZIlly1gjek1o9c4uj30ee08rgls.jpg)

- 网络

  在 kubernetes 集群中，每个 Pod 都分配了唯一的 IP 地址，Pod 中的容器都共享网络命名空间，包括 IP 地址和网络端口。在 Pod 内部各容器之间可以通过 localhost 互相通信。我们可以对比一下 Docker 和 kubernetes 在网络空间上的差异。

  Docker的网络空间

  ![image-20201005171009547](https://tva1.sinaimg.cn/large/007S8ZIlly1gjekm6snmaj30bp0bfjrq.jpg)

从图中可以看出，容器之间通过docker0网卡连接，每个容器拥有独立的内部网络地址

kubernetes 的网络空间

![image-20201005173155744](https://tva1.sinaimg.cn/large/007S8ZIlly1gjel8uhcpzj30bk0bnt91.jpg)

从图中可以看出，Pod 中的所有容器共享一个网络地址

### Pod 之间如何通信

Pod 之间的通信主要分为两种情况：

- 同一个 Node 上 Pod 之间的通信

  ​        同一个 Node 上的 Pod 使用的都是相同的网桥( docker0 )进行连接，相当于 Docker 的网络空间，只不过是以 Pod 为基础。每个 Pod 都有一个全局 IP 地址，同一个 Node 内不同 Pod 之间通过 veth 连接在同一个 docker0 网桥上，其 IP 地址都是从 docker0 网桥上动态获取的，并且关联在同一个 docker0 网桥上，地址段也相同，所以它们之间能直接通信。

- 不同 Node 之间的 Pod 通信

  ​        要实现不同 Node 的 Pod 之间通信，首先要保证 Pod 在一个 kubernetes 集群中拥有全局唯一的 IP 地址。又因为一个 Node 上的 Pod 是通过 Docker 网桥与外部进行通信的，所以只要将不同 Node 上的 Docker 网桥配置成不同的 IP 地址段就可以实现这个功能。

  ​        Flannel 会配置 Docker 网桥，通过修改 Docker 的启动参数 bip 来实现这一点，这样就使得集群中机器的 Docker 网桥就得到了全局唯一的 IP 地址段，机器上所创建的容器也就拥有了全局唯一的 IP 地址。

  跨 Node 的 Pod 之间的通信

  ![image-20201006103818808](https://tva1.sinaimg.cn/large/007S8ZIlly1gjfewsoh8sj30km0dvmxy.jpg)

  我们在搭建 kubernetes 集群的时候，在每个 Node 机器上都创建了一个 kube-flannel 容器，这是在每个 Node 机器上使用 Flannel 虚拟网卡接管容器并跨主机通信，当一个节点的容器访问另一个节点的容器时，源节点上的数据会从 docker0 网桥路由到 flannel0 网卡，在目的节点处会从 flannel0 网卡路由到 docker0 网桥，然后再转发给目标容器。Flannel 重新规划了容器集群的网络，这样既保证了容器 IP 的全局唯一性，又让不同机器上的容器能通过内网 IP 地址互相通信。但是，容器的 IP 地址并不是固定的，Flannel 只分配子网段，所以容器的 IP 地址是在此网段的范围内进行动态分配的。

  ​        因为 Pod 的 IP 地址本身是虚拟 IP，所以只有 kubernetes 集群内部的机器( Master 和 Node )和其他 Pod 可以直接访问这个 IP 地址，集群之外的机器无法直接访问 Pod 的 IP 地址。我们创建了一个 Nginx Pod：

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: nginx
  spec:
    containers:
    - name: exnginx
      image: nginx
      imagePullPolicy: IfNotPresent
      ports:
      - name: nginxport
        containerPort: 80
  ```

  通过命令创建后查询以下 IP 地址：

  ```shell
  [root@k8s-master]# kubectl apply -f exnginx.yml
  [root@k8s-master]# kubectl get pod -o wide
  ```

  ![image-20201006105652067](https://tva1.sinaimg.cn/large/007S8ZIlly1gjffg3n5g5j30k202e3yq.jpg)

  集群中的所有机器和 Pod 都可以访问这个虚拟地址和 containerPort 暴露的端口

  ```shell
  [root@k8s-master]# curl http://10.244.1.6
  ```

  ![image-20201006110000973](https://tva1.sinaimg.cn/large/007S8ZIlly1gjffjd3sjcj30ft0efmyi.jpg)

  同时我们看到我们有两个 Pod，一个在 Node1 上，一个在 Node2 上，而 Node2 上的 Pod IP 地址是10.244.2.6，Node1 上的 Pod IP 地址是10.244.1.6，登录到两台机器上，使用 ifconfig flannel.1 命令查看集群子网段。

  k8s-node1

  ![image-20201006110403352](https://tva1.sinaimg.cn/large/007S8ZIlly1gjffnkerfej30eq058gmc.jpg)

  K8s-node2

  ![image-20201006110436365](https://tva1.sinaimg.cn/large/007S8ZIlly1gjffo4zn7sj30er05it9g.jpg)

  要使集群外的机器访问 Pod 提供的服务，后面我们会介绍 Service 和 Ingress 来发布服务。

# Pod 的生命周期

## Pod 的状态

| 状态值    | 说明                                                         |
| --------- | ------------------------------------------------------------ |
| Pending   | Master 已接收到 Pod 的创建信息，但是尚有一个或多个容器镜像未能创建 |
| Running   | Pod 已绑定到 Node，所有的容器均已创建，至少有一个容器在运行或者正在启动、重启 |
| Completed | Pod 中的所有容器都已成功终止，并且不会重新启动               |
| Failed    | Pod 中的所有容器都已终止，并且至少有一个容器表现出失败的终止状态，也就是说，容器要么一非0状态退出，要么被系统终止 |
| Unknown   | 由于某种原因，无法获得 Pod 的状态，通常情况是 Pod 所在的宿主机通信出错 |

一旦开始在集群节点中创建 Pod，首先就会进入 Pending 状态，只要 Pod 中的所有容器都已启动并正常运行，则 Pod 接下来会进入 Running 状态，如果 Pod 被要求终止，且所有容器终止退出时的状态码都为0，Pod 就会进入 Succeeded 状态。

如果进入 Failed 状态，通常有以下3种原因。

- Pod 启动时，只要有一个容器运行失败，Pod 将会从 Pending 状态进入 Failed 状态。
- Pod 正处于 Running 状态，若 Pod 中的一个容器突然损坏或者在退出时状态码不为0，Pod 将会从 Running 进入 Failed 状态。
- 在要求 Pod 正常关闭的时候，只要有一个容器退出的状态码不为0，Pod 就会进入 Failed 状态。

## Pod 的重启策略

​        在配置 Pod 的模板文件中有个 spec 模块，其中有一个名为 restartPolicy 的字段，字段的值为 Always、OnFailure、Never。Node 上的 kubelet 通过 restartPolicy 执行重启操作，由 kubelet 重新启动的已退出容器将会以递增延迟的方式（10s，20s，40s，...）尝试重新启动，上限时间为 5min，延时的累加值会在成功运行 10min 后重置，一旦 Pod 绑定到某个节点上，就绝对不会重新绑定到另一个节点上。

重启策略对 Pod 状态的影响如下：

- 假设有1个运行中的 Pod，包含1个容器，容器退出成功后。
  - Always：重启容器，Pod 状态仍为 Running。
  - OnFailure：Pod 状态变为 Completed。
  - Never：Pod 状态变为 Completed。
- 假设有1个运行中的 Pod，包含1个容器，容器退出失败后。
  - Always：重启容器，Pod 状态仍为 Running。
  - OnFailure：重启容器，Pod 状态仍为 Running。
  - Never：Pod 状态变为 Failed。
- 假设有1个运行中的 Pod，包含2个容器，第1个容器退出失败后。
  - Always：重启容器，Pod 状态仍为 Running。
  - OnFailure：重启容器，Pod 状态仍为 Running。
  - Never：不会重启容器，Pod 状态仍为 Completed。
- 假设第1个容器没有运行起来，而第2个容器也退出了。
  - Always：重启容器，Pod 状态仍为 Running。
  - OnFailure：重启容器，Pod 状态仍为 Running。
  - Never：Pod 状态变为 Failed。
- 假设有1个运行中的 Pod，包含1个容器，容器发生内存溢出后。
  - Always：重启容器，Pod 状态仍为 Running。
  - OnFailure：重启容器，Pod 状态仍为 Running。
  - Never：记录失败事件，Pod 状态变为 Failed。

## Pod 的创建于销毁过程

Pod 的创建过程：

- kubectl 命令将转换为对 API Server 的调用。
- API Server 验证请求并将其保存到 etcd 中。
- etcd 通知 API Server。
- API Server 调用调度器。
- 调度器决定在哪个节点上运行 Pod，并将其返回给 API Server。
- API Server 将其对应节点保存到 etcd 中。
- etcd 通知 API Server。
- API Server 在相应的节点中调用 kubelet。
- kubelet 与容器运行时 API 发生交互，与容器守护进程通信以创建容器。
- kubelet 将 Pod 状态更新到 API Server 中。
- API Server 把最新的状态保存到 etcd 中。

Pod 的销毁过程：

- 用户发送删除 Pod 的命令。
- 将会更新 API Server 中的 Pod 对象，设定 Pod 被”销毁“完成的大致时间（默认 30s），超出这个宽限时间 Pod 将被强制终止。
- 同时触发以下操作：
  - Pod 被标记为 Terminating。
  - kubelet 发现 Pod 已标记为 Terminating 后，将会执行 Pod 关闭过程。
  - Endpoint 控制器监控到 Pod 即将删除，将溢出所有 Service 对象中与该 Pod 相关的 Endpoint。
- 如果 Pod 定义了 preStop 回调，则这会在 Pod 中执行，如果宽限时间到了 preStop 还在运行，则会通知 API Server增加少量宽限时间（2s）。
- Pod 中的进程接收到 TERM 信号。
- 如果宽限时间过期，Pod 中的进行仍在运行，则会被 SIGKILL 信号终止。
- kubelet 通过 API Server 设置宽限时间为 0（立即删除），完成 Pod 的删除操作，Pod 从 API 中移除。

删除操作的延迟时间默认为 30s。kubectl delete 命令支持--grace-period=秒，用户可以自定义延迟时间。如果这个值设置为0，则表示强制删除 Pod，但是在使用--grace-period=0 时需要增加选项 --force 才能执行强制删除。

## Pod 的生命周期时间

​        Pod 在整个运行过程中，会有两个大的阶段，第一阶段是初始化容器运行阶段，第二阶段是正式容器运行阶段，每个阶段都会有不同的事件

### 初始化容器运行阶段

​        Pod 中可以包含一个或者多个初始化容器，这些容器是在应用程序容器正式运行之前运行的，主要负责一些初始化工作，所有初始化容器执行完后才能执行应用程序容器，因此初始化容器不能是长期运行的容器，而是执行完一定操作后就必须结束的。如果是多个初始化容器，只能顺序执行，不能同时运行。在应用程序容器开始前，所有初始化容器都必须正常结束。

​        初始化容器执行失败时，如果 restartPolicy 是 OnFailure 或者 Always，那么会重复执行失败的初始化容器，直到成功；如果 restartPolicy 是 Never，则不会重启失败的初始化容器。

下面我们举一个例子，在部署应用程序前，检测 db 是否就绪，并执行以下初始化脚本。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: expodinitcontainer
spec:
  containers:
  - name: maincontainer
    image: busybox
    command: ['sh', '-c']
    args: ['echo "maincontainer is running!"; sleep 3600']
  initContainers:
  - name: initdbcheck
    image: busybox
    command: ['sh', '-c']
    args: ['echo "checking db!"; sleep 30; echo "checking done!"']
  - name: initscript
    image: busybox
    command: ['sh', '-c']
    args: ['echo "init script exec!"; sleep 30; echo "init script exec done!"']
```

这里包含一个主容器，两个初始化容器，每个初始化容器执行 30s，接下来我们创建一下 Pod，再查看他们的状态：

```shell
kubectl apply -f expodinitcontainers.yml
```

在 30s 内执行第一个初始化容器，所以 Pod 状态是 Init:0/2，在 30s-60s 之间执行第二个初始化容器，所以 Pod 状态是 Init:1/2，当所有初始化容器执行完毕后，状态会先变为 PodInitializing，然后变为 Running 状态。通过 kubectl describe 命令查看 Pod 详细信息，可以看到先执行 initdbcheck，然后执行 initscript，最后才运行 maincontainer。

![image-20201006173236009](https://tva1.sinaimg.cn/large/007S8ZIlly1gjfqvw9jjnj30pe06swg0.jpg)

### 正式容器运行阶段

​        正式容器创建成功后，就会触发 PostStart 事件，在容器运行的过程中，可以设置存活探针和就绪探针来持续监测容器的健康状况，在容器结束前，会触发 PreStop 事件。

- PostStart：容器刚创建成功后，触发此事件，如果回调执行失败，则容器会被终止，然后根据重启策略决定是否要重启该容器。
- PreStop：容器开始和结束前，触发此事件，无论执行结果如何，都会结束容器。

回调的方式有两种：Exec 执行一段脚本和 HttpGet 执行特定的请求。

- Exec 回调会执行特定的命令，如果 Exec 执行的命令最后正常退出，则代表执行成功；否则就认为执行异常，配置方式如下：

  ```yaml
  postStart/preStop:
    exec:
      command: xxxxx  # 命令列表
  ```

- HttpGet 回调会执行特定的 Http 请求，通过返回的状态码来判断该请求执行是否成功，配置方式如下：

  ```yaml
  postStart/preStop:
    httpGet:
      host: xxxx # 请求的 IP 地址或域名
      port: xx # 请求的端口号
      path: xxxxxx # 请求的路径，比如github.com/xingxingzaixian，"/xingxingzaixian"就是路径
      scheme: http/https # 请求的协议，默认为 HTTP
  ```

我们来举例使用一下 PostStart 和 PreStop 事件

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: expodpostpre
spec:
  containers:
  - name: postprecontainer
    image: busybox
    imagePullPolicy: IfNotPresent
    command: ['sh', '-c']
    args: ['echo "Hello Kubernetes!"; sleep 3600']
    lifecycle:
      postStart:
        httpGet:
          host: www.baidu.com
          path: /
          port: 80
          scheme: HTTP
      preStop:
        exec:
          command: ['sh', '-c', 'echo "preStop callback done!"; sleep 60']
```

postStart 中我们执行 HttpGet 回调，访问了百度首页，preStop 则执行命令输出一段文本，然后停留 60s。我们执行创建命令，观察一下 Pod 的状态。

```shell
kubectl apply -f expodpostpre.yml
```

正常情况下是和之前的是一样的，我们做一些测试操作，例如把 postStart 的网址写一个不可访问的网址，比如：

```yaml
host: www.fackbook.com
```

创建后，查看日志信息

```shell
kubectl logs expodpostpre
```

![image-20201006201438280](https://tva1.sinaimg.cn/large/007S8ZIlly1gjfvkfxqj9j30ob019wei.jpg)

还可以通过 kubectl describe 来查看详细情况

![image-20201006201620120](https://tva1.sinaimg.cn/large/007S8ZIlly1gjfvm7kxyfj30su03ft9f.jpg)

# Pod 的健康检查

​        在容器运行的过程中，我们可以通过探针来持续检查容器的状况，kubernetes 为我们提供了两种探针：存活探针、就绪探针。

- 存活探针livenessProbe：检测容器是否正在运行，如果返回 Failure，kubelet 会终止容器，然后容器会按照重启策略执行。如果没有提供存活探针，默认状态就是 Success。
- 就绪探针readlinessProbe：检测容器是否已经可以启动了应用服务，如果返回 Failure，Endpoint 控制器就会从所有 Service 的 Endpoint 中移除此 Pod 的 IP 地址。从容器启动到第一次探测之前，默认的就绪状态是 Failure。如果没有提供就绪探针，默认状态就是 Success。

容器配置当中有 3 种方法来执行探针检测：exec、tcpSocket、httpGet。

- exec：在容器内部执行指定的命令，如果命令以状态码“0”退出，则表示诊断成功。配置如下：

  ```yaml
  livenessProbe/readlinessProbe:
    exec:
      command: [xxxx] # 命令列表
  ```

- tcpSocket：对容器的指定端口执行 TCP 检测。如果端口是打开的，则诊断成功。配置如下：

  ```yaml
  livenessProbe/readlinessProbe:
    tcpSocket:
      port: Number # 指定端口号
  ```

- httpGet：对容器内的 HTTP 服务进行检测，如果响应的状态码范围为 200~400，则诊断成功。配置如下：

  ```yaml
  livenessProbe/readlinessProbe:
    httpGet:
      port: Number # 指定的端口号
      path: String # 指定路径
  ```

下面举几个例子来展示一下探针的使用。

## 存活探针的使用

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: expodlive
spec:
  containers:
  - name: livecontainer
    image: busybox
    imagePullPolicy: IfNotPresent
    command: ['sh', '-c']
    args: ['mkdir /files_dir; echo "Hello Kubernetes!" > /files_dir/newfile; sleep 3600']
    livenessProbe:
      exec:
        command: ['cat', '/files_dir/newfile']
```

先创建一个文件夹 files_dir，再新建一个文件 newfile，写入 Hello Kubernetes! 内容，然后在存活探针中使用 cat 查看文件内容。执行创建 Pod 命令：

```shell
kubectl apply -f expodlive.yml
```

![image-20201006205138102](https://tva1.sinaimg.cn/large/007S8ZIlly1gjfwmxvt7fj309401t3yh.jpg)

目前 Pod 运行一切正常，现在我们可以做一些破坏性的操作，进入 Pod 内部，删除 newfile 文件。

```shell
[root@k8s-master]# kubectl exec -it expodlive -- /bin/sh
/# rm -f /files_dir/newfile
/# exit
```

由于探针定期检查 /files_dir/newfile 文件是否存在，而我们的 Pod 默认是异常后重启，因此可以通过以下命令查看 Pod 详细信息：

```shell
kubectl describe pods expodlive
```

![image-20201006210634390](https://tva1.sinaimg.cn/large/007S8ZIlly1gjfx2hd5qhj30ss04tq3r.jpg)

可以看到 Event 中打印了存活探针的运行情况：存活探针失败，并且准备重启。

## 就绪探针的使用

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: expodread
spec:
  containers:
  - name: readcontainer
    image: nginx
    imagePullPolicy: IfNotPresent
    ports:
    - name: portnginx
      containerPort: 80
    readinessProbe:
      httpGet:
        port: 80
        path: /
```

我们创建了一个 Nginx 容器，通过 containerPort 属性，将 80 端口暴露出来，然后设置一个就绪探针定期向 80 端口发送 HttpGet 请求，检测响应范围是否为 200~400。使用 apply 命令创建成功后，我们同样要进行一些测试性的操作。

```shell
[root@k8s-master]# kubectl apply -f expodread.yml
[root@k8s-master]# kubectl exec -it expodread -- /bin/sh
/# nginx -s stop
```

![image-20201006212736392](https://tva1.sinaimg.cn/large/007S8ZIlly1gjfxodhgvrj30cb02cmxa.jpg)

执行 nginx -s stop 命令后，容器就会退出，因为 Nginx 容器是持续提供服务的，服务停止后，容器就异常了，然后 kubernetes 又会重新拉起一个容器，在这个过程当中，我们使用 kubectl get pod 命令就会发现，容器的状态先变为 Completed，然后变为 CrashLoopBackOff，最后变为 Running。

![image-20201006212924528](https://tva1.sinaimg.cn/large/007S8ZIlly1gjfxq8v3y1j30b204lmxj.jpg)

对于初学者来讲，总是分不清存活探针和就绪探针的区别，什么情况下该使用存活探针？什么情况下应该使用就绪探针？我给的建议如下：

- 如果容器中的进程能够在遇到问题或异常的情况下自行崩溃，就像刚才的 Nginx 容器，那么不一定需要存活探针，kubelet 会根据 Pod 的重启策略自动执行正确的操作。
- 如果想在探测失败时终止并重启容器，则可以指定存活探针，并将重启策略设置为 Always 或 OnFailure。
- 如果只想在探针成功时才对 Pod 发送网络请求，则可以指定就绪探针，例如 HttpGet。
- 如果容器需要在启动期间处理大型数据、配置文件或迁移，就使用就绪探针。

对于每种探针，还可以设置 5 个参数：

- initialDelaySeconds：启动容器后首次监控检测的等待时间，单位为秒。
- timeoutSeconds：发送监控检测请求后等待响应的超时时间，单位为秒，如果超时就认为探测失败，默认值为 10s。
- periodSeconds：探针的执行周期，默认 10s 执行一次。
- successThreshold：如果出现失败，则需要连续探测成功多次才能确认诊断成功。默认值为1.
- failureThreshold：如果出现失败，则要连续失败多次才重启 Pod（对于存活探针）或标记为 Unready（对于就绪探针）。默认值为3。

具体设置方法如下：

```yaml
livenessProbe/readinessProbe:
  exec/tcpSocket/httpGet:
    initialDelaySeconds: Number
    timeoutSeconds: Number
    periodSeconds: Number
    successThreshold: Number
    failureThreshold: Number
```

# 总结

这篇文章我们主要讲了 Pod 的增删改查，Pod 与容器的关系，如何组织容器，Pod 的生命周期及对应事件，以及 Pod 的健康检查机制。





