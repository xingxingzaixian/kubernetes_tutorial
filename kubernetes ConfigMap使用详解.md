在应用部署时，配置文件几乎是绕不开的，通常我们是把配置文件放置在指定目录下，通过配置文件使应用修改的灵活性更高。但是如果我们把应用打包成容器镜像后，容器内的镜像文件就不容易修改了，一般我们会采用以下方式修改容器中的配置文件：

- 通过环境变量传入
- 外挂文件，在容器启动时引入

但是以上两种方式在进行大规模集群部署时，对多个容器进行不同的配置会变得比较复杂。这种不方便不仅针对容器，对于传统运维来讲，也是存在的，因此配置中心这个概念就应运而生了，例如

**Apollo**，由携程开源的分布式配置中心，就是将配置信息存储在数据库中，然后对外提供 API，这样就能集中化管理不同应用的不同配置，而且修改后实时生效。kubernetes 作为集中化运维管理实施方案，也提供了集中配置管理方案- **ConfigMap**。下面我们就来详细讲解一下使用方式。

## ConfigMap 的创建

- 通过 YAML 配置文件方式

  - 按照环境变量的方式配置

    ```yaml
    # example_env.yml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: example
    data:
      server_name: "www.baidu.com"
      server_port: "8080"
    ```

    执行创建命令：

    ```shell
    kubectl apply -f example_env.yml
    ```

    查看configmap内容

    ![image-20201115171746365](https://tva1.sinaimg.cn/large/0081Kckwly1gkpz8sffayj30dp07sgly.jpg)

  - 按照文件内容方式配置

    ```yaml
    # example_content.yml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: examplecontent
    data:
      redis.conf: |
        daemonize no
        pidfile /var/run/redis.pid
        port 6379
        bind 127.0.0.1
        databases 16
      my.cnf: |
        [client]
        port=3306
        [mysqld]
        default-storage-engine=INNODB
        socket=/usr/local/mysql/mysql.sock
    ```

    执行创建命令：

    ```shell
    kubectl apply -f example_content.yml
    ```

    查看执行内容

    ![image-20201115172037828](https://tva1.sinaimg.cn/large/0081Kckwly1gkpzbse4fsj30ai0aj3yx.jpg)

- 通过 kubectl 命令行方式

  命令的的创建方式主要是不写 YAML 文件，而是通过命令行引入本地文件的内容作为配置

  - 导入文件内容作为配置

    ```yaml
    # my.cnf
    [client]
    port=3306
    
    [mysqld]
    default-storage-engine=INNODB
    socket=/usr/local/mysql/mysql.sock
    ```

    执行创建命令：

    ```shell
    kubectl create configmap examplefile --from-file=my.cnf
    ```

    ![image-20201115172152597](https://tva1.sinaimg.cn/large/0081Kckwly1gkpzd0p0ktj30cl08igm2.jpg)

  - 导入目录下所有文件内容作为配置

    传入目录后，将会把目录下所有文件名和内容作为 key : value 生成配置，目录结构如下：

    ![image-20201115172428135](https://tva1.sinaimg.cn/large/0081Kckwly1gkpzfpkid2j304h01q3yb.jpg)

    包含两个文件 my.cnf 和 redis.conf，执行以下创建命令：

    ```shell
    kubectl create configmap exampledir --from-file=/home/kube_study/configmap/dir
    ```

    查看执行结果，可以发现在 exampledir 中配置了两个键值对，一个的键为 my.cnf，内容是它的文件内容；一个的键为 redis.conf，内容也是它的内容。

## 在 Pod 中使用 ConfigMap

### 通过环境变量的方式使用

我们使用第一个名为 example 的 ConfigMap 作为示例，先采用 **valueFrom** 的方式引入：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: envpod
spec:
  containers:
  - name: test
    image: busybox
    command: ["/bin/sh", "-c", "env | grep SERVER"]
    env:
    - name: SERVERNAME    # 定义环境变量的名称
      valueFrom:          
        configMapKeyRef:  
          name: example    # 环境变量取自 example
          key: server_name # key 为 server_name
    - name: SERVERPORT
      valueFrom:
        configMapKeyRef:
          name: example
          key: server_port
  restartPolicy: Never
```

创建 Pod，并查看输出日志

```shell
kubectl apply -f envpod.yml
kubectl logs pod/envpod
```

![image-20201115174847466](https://tva1.sinaimg.cn/large/0081Kckwly1gkq050sta1j30aq01s0sp.jpg)

还有另外一种引入方式 envFrom，将自动把 ConfigMap 中的 key=value 自动生成环境变量：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: envpod1
spec:
  containers:
  - name: test
    image: busybox
    command: ["/bin/sh", "-c", "env | grep SERVER"]
    envFrom:
      - configMapRef:
          name: example
  restartPolicy: Never
```

请注意配置方式，- configMapRef 比上一行多了两个空格，name: example 比上一行多了**四个空格**，否则这里会报错

```
error: error validating "podenv1.yml": error validating data: ValidationError(Pod.spec.containers[0].envFrom[0]): unknown field "name" in io.k8s.api.core.v1.EnvFromSource; if you choose to ignore these errors, turn validation off with --validate=false
```

查看 Pod 打印的日志内容

![image-20201115180848898](https://tva1.sinaimg.cn/large/0081Kckwly1gkq0puw3lrj30ai01r748.jpg)

**注意**：环境变量的名称要符合 POSIX 命名规范，如果不符合规范，则会跳过该环境变量的创建，但是不会阻止 Pod 的启动，我们可以在 Event 中查看记录的日志。

### 通过 volumeMount 使用 ConfigMap

我们以上面创建的 exampledir 来作为示例

- 将配置挂载为目录

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: filepod
spec:
  containers:
  - name: test
    image: busybox
    command: ["/bin/sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: serverfile  # 引用的 volums 名称
      mountPath: /configfiles  # 容器内挂载的目录
  volumes:
  - name: serverfile  # 定义的 volums 名称
    configMap:
      name: exampledir      # 使用名为 exampledir 的 ConfigMap 
      items:
        - key: redis.conf   # ConfigMap 中的 key 名称
          path: redis.conf  # 挂载到容器后的文件名
        - key: my.cnf       # ConfigMap 中的 key 名称
          path: my.cnf      # 挂载到容器后的文件名
  restartPolicy: Never
```

创建 Pod

```shell
kubectl apply -f filepod.yml
```

登录 Pod，查看目录 /configfiles 下的文件，可以看到两个配置都以文件的方式呈现出来

![image-20201115182802935](https://tva1.sinaimg.cn/large/0081Kckwly1gkq19vi35fj30cq01xgll.jpg)

**注意**：上面我们采用的 items 来讲 ConfigMap 中的 key 和本地生成的文件对应起来，如果我们不指定 items，那么将会以 ConfigMap 中的 key 为文件名，value 为文件内容创建文件。配置方式如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: filepod1
spec:
  containers:
  - name: test
    image: busybox
    command: ["/bin/sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: serverfile  # 引用的 volums 名称
      mountPath: /configfiles  # 容器内挂载的目录
  volumes:
  - name: serverfile  # 定义的 volums 名称
    configMap:
      name: exampledir      # 使用名为 exampledir 的 ConfigMap 
```

创建后可查看容器内目录

![image-20201115183728829](https://tva1.sinaimg.cn/large/0081Kckwly1gkq1jojt4ij30cz01paa1.jpg)

在进行目录挂载时，如果挂载的目录中有文件，该目录下的文件将被清掉，因此我们需要明确的将配置挂载为文件来使用。

- 将配置挂载为文件

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: filepod
  spec:
    containers:
    - name: test
      image: busybox
      command: ["/bin/sh", "-c", "sleep 3600"]
      volumeMounts:
      - name: serverfile  # 引用的 volums 名称
        mountPath: /configfiles/a.txt  # 容器内挂载的目录
        subPath: a.txt
      - name: serverfile  # 引用的 volums 名称
        mountPath: /configfiles/b.txt  # 容器内挂载的目录
        subPath: b.txt
    volumes:
    - name: serverfile  # 定义的 volums 名称
      configMap:
        name: exampledir      # 使用名为 exampledir 的 ConfigMap 
        items:
          - key: redis.conf   # ConfigMap 中的 key 名称
            path: a.txt  # 挂载到容器后的文件名
          - key: my.cnf       # ConfigMap 中的 key 名称
            path: b.txt      # 挂载到容器后的文件名
  ```

  执行创建命令后，登录到 Pod 中去查看以下生成的文件

  ![image-20201115194108716](https://tva1.sinaimg.cn/large/0081Kckwly1gkq3dx9piij30fm08hmxu.jpg)

  文件已生成，而且内容和我们配置的相同，要注意我们在 items 中配置的 path 要和 subPath 名称相同，否则文件将不会被创建，并且 subPath 也会被当做目录创建在容器中，但是它是一个空目录。

  ​      subPath 可以用来解决一个 Pod 挂载多个 ConfigMap，并且置于同一目录下的问题。

## 使用 ConfigMap 的限制条件

- ConfigMap 必须在 Pod 之前创建
- ConfigMap 会收到 Namespace 影响，只有处于相同 Namespace 中的 Pod 才可以引用
- Pod 引用了 ConfigMap 后，即使更新了 ConfigMap 中的值，Pod 中也不会变化，重启后才会变

