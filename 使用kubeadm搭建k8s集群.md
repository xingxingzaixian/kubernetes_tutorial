## 环境准备

| 作系统   | 主机名     | IP地址          | 配置  |
| -------- | ---------- | --------------- | ----- |
| CentOS 7 | k8s-master | 192.168.143.121 | 2核2G |
| CentOS 7 | k8s-node1  | 192.168.143.131 | 2核2G |
| CentOS 7 | K8s-node2  | 192.168.143.132 | 2核2G |

- 修改hostname

  ```shell
  hostnamectl set-hostname k8s-master
  hostnamectl set-hostname k8s-node1
  hostnamectl set-hostname k8s-node2
  ```

  或者修改vi /etc/hostname

- 修改hosts

  vi /etc/hosts

  ```shell
  192.168.143.130 k8s-master
  192.168.143.131 k8s-node1
  192.168.143.132 k8s-node2
  ```

- 关闭防火墙

  ```shell
  systemctl disable firewalld.service
  systemctl stop firewalld.service
  ```

- 关闭SELinux

  ```shell
  setenforce 0
  sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
  ```

- 关闭swap

  ```shell
  # 临时关闭
  swapoff -a
  
  # 永久关闭
  sed -ri 's/.*swap.*/#$/' /etc/fstab
  ```

## 安装基础组件

- docker

  ```shell
  ~>> yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
  
  ~>> yum makecache fast
  
  ~>> yum -y install docker-ce
  
  ~>> systemctl start docker
  ```

- Kubelet kubeadm kubectl

  ```shell
  ~>> cat <<EOF > /etc/yum.repos.d/kubernetes.repo
  [kubernetes]
  name=Kubernetes
  baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
  enabled=1
  gpgcheck=1
  repo_gpgcheck=1
  gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
  EOF
  
  ~>> setenforce 0
  ~>> sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
  ~>> yum install -y kubelet kubeadm kubectl
  ~>> systemctl enable kubelet && systemctl start kubelet
  ```

# 安装kubernetes集群

## master的安装与配置

- CPU数量至少需要两个，通过虚拟机软件调节

- 内存不要小于2G，否则可能无法关闭交换区

- 不支持交换内存

  基于性能等方面的考虑，kubernetes禁用了交换内存，所以要关闭swap。

  ```shell
  # 临时关闭
  swapoff -a
  
  # 永久关闭
  sed -ri 's/.*swap.*/#$/' /etc/fstab
  ```

- 使用iptables进行流量桥接

  kubernetes的service通过iptables来做后端pod的转发和路由，所以我们必须保证iptables可以进行流量的转发

  ```shell
  ~>> cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
  net.bridge.bridge-nf-call-ip6tables = 1
  net.bridge.bridge-nf-call-iptables = 1
  EOF
  
  ~>> sudo sysctl --system
  ```

### kubeadm init安装

因为kubeadm需要用到容器，这些镜像都是k8s.gcr网站，因为众所周知的原因，国内是访问不到的，所以无法创建成功。一种方式就是科学上网，另外一种就是从其他地方下载。网上通常都是从阿里云下载，但是阿里云的好像不一定能用了，我在docker hub上发现了kubernetes的同步库gotok8s，应该是官方同步过来了，更新比较及时，版本也相互对应，配置好加速器下载也非常快。如果对应版本的库不存在，就找版本相近的（kube开头的几个库版本要相同），在安装的时候指定好对应的版本。

#### 查询所需镜像

```shell
kubeadm config images list
```

通过上面的命令，可以查询我们安装的kubeadm所需要的k8s镜像列表

![image-20201001162626436](https://tva1.sinaimg.cn/large/007S8ZIlly1gj9wviauv1j30rp05ydgu.jpg)

简单的策略就是通过docker pull下载镜像，然后使用docker tag修改为kubeadm所需要的的镜像名字，v1.18.9这个版本我没找到对应的，所以使用v1.18.8。命令如下：

```shell
docker pull gotok8s/kube-apiserver:v1.18.8
docker tag gotok8s/kube-apiserver:v1.18.8 k8s.gcr.io/kube-apiserver:v1.18.8
docker rmi gotok8s/kube-apiserver:v1.18.8

docker pull gotok8s/kube-controller-manager:v1.18.8
docker tag gotok8s/kube-controller-manager:v1.18.8 k8s.gcr.io/kube-controller-manager:v1.18.8
docker rmi gotok8s/kube-controller-manager:v1.18.8

docker pull gotok8s/kube-scheduler:v1.18.8
docker tag gotok8s/kube-scheduler:v1.18.8 k8s.gcr.io/kube-scheduler:v1.18.8
docker rmi gotok8s/kube-scheduler:v1.18.8

docker pull gotok8s/kube-proxy:v1.18.8
docker tag gotok8s/kube-proxy:v1.18.8 k8s.gcr.io/kube-proxy:v1.18.8
docker rmi gotok8s/kube-proxy:v1.18.8

docker pull gotok8s/pause:3.2
docker tag gotok8s/pause:3.2 k8s.gcr.io/pause:3.2
docker rmi gotok8s/pause:3.2

docker pull gotok8s/etcd:3.4.3-0
docker tag gotok8s/etcd:3.4.3-0 k8s.gcr.io/etcd:3.4.3-0
docker rmi gotok8s/etcd:3.4.3-0

docker pull gotok8s/coredns:1.6.7
docker tag gotok8s/coredns:1.6.7 k8s.gcr.io/coredns:1.6.7
docker rmi gotok8s/coredns:1.6.7 
```

#### 安装master

```shell
kubeadm init --kubernetes-version=1.18.8 --pod-network-cidr 10.244.0.0/16 
```

![image-20201001171859620](https://tva1.sinaimg.cn/large/007S8ZIlly1gj9ye5zmsij30lu0atdh8.jpg)

如果有错误会在[Error]行打印出来，根据错误查询解决即可，如果出现不可逆的异常，可以使用kubeadm reset命令来重置状态。

根据执行完的界面提示，需要执行以下命令，来创建集群

```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

此时就可以通过kubectl get nodes命令查看集群状态

![image-20201001172342520](https://tva1.sinaimg.cn/large/007S8ZIlly1gj9yj204y3j309601pjrd.jpg)

这时候只有一个master节点，而且是NotReady状态，不用着急，master就安装到这里，我们根据安装界面的最后一个提示，在Node上执行加入集群的命令即可。

## Node的安装与配置

登录node后，执行安装master后的kubeadm join命令

```shell
kubeadm join 192.168.143.130:6443 --token o3cc3t.4ikrxmog4wxiijrt \
    --discovery-token-ca-cert-hash sha256:684bd6c7f33db9e4c28f5006d896f0fd02015fe2bfc5b186202a6ab00507ba68
```

如果忘记了这个命令，可以在Master节点中执行以下命令查询

```shell
kubeadm token create --print-join-command
```

![image-20201001173147481](https://tva1.sinaimg.cn/large/007S8ZIlly1gj9yrgy2b1j30r602qt97.jpg)

### 可能出现的问题

我这里执行kubeadm join的时候报错了，错误信息如下：

```shell
W1002 01:53:33.649117    9708 join.go:346] [preflight] WARNING: JoinControlPane.controlPlane settings will be ignored when control-plane flag is not set.
[preflight] Running pre-flight checks
        [WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
error execution phase preflight: couldn't validate the identity of the API Server: Get https://192.168.143.130:6443/api/v1/namespaces/kube-public/configmaps/cluster-info?timeout=10s: dial tcp 192.168.143.130:6443: connect: no route to host
```

通过网上查找，可能是iptables规则异常了，重新刷新一下就可以了，执行以下命令修复：

```shell
# 在master机器上执行
[root@k8s-master ~]# systemctl stop kubelet
[root@k8s-master ~]# systemctl stop docker
[root@k8s-master ~]# iptables --flush    #有可能执行这两句就可以了
[root@k8s-master ~]# iptables -tnat --flush
[root@k8s-master ~]# systemctl start kubelet
[root@k8s-master ~]# systemctl start docker

# 在master上重新生成token
[root@k8s-master ~]# kubeadm token create
[root@k8s-master ~]# openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'

# 在node节点中执行kubeadm join，使用上面两条命令创建的的token进行拼接
[root@k8s-node1 ~]# kubeadm join 192.168.143.130:6443 --token m8or6s.j5rm351b5hzyk8el --discovery-token-ca-cert-hash sha256:684bd6c7f33db9e4c28f5006d896f0fd02015fe2bfc5b186202a6ab00507ba68
```

![image-20201001185314433](https://tva1.sinaimg.cn/large/007S8ZIlly1gja147hdbsj30g0032dg0.jpg)

上面的信息显示node已经加入到集群中。

> 配置kubectl命令自动补全
>
> ```shell
> [root@k8s-master ~]# yum install -y bash-completion 
> [root@k8s-master ~]# source /usr/share/bash-completion/bash_completion
> [root@k8s-master ~]# source <(kubectl completion bash)
> [root@k8s-master ~]# echo "source <(kubectl completion bash)" >> ~/.bashrc
> ```

在另外的机器上出现如下错误：

![image-20201002082439488](https://tva1.sinaimg.cn/large/007S8ZIlly1gjaokhqpvwj30rd05l0tq.jpg)

前两个错误提示文件内容没有设置为1，只要设置一下就行了，可以用一下命令修改：

```shell
[root@k8s-node2 ~]# echo "1">/proc/sys/net/ipv4/ip_forward
[root@k8s-node2 ~]# echo "1">/proc/sys/net/bridge/bridge-nf-call-iptables
```

第三个错误是关闭内存交换，前面有设置过，参考一下就行了。

我们在master上查询一下nodes的状态。

![image-20201001190053999](https://tva1.sinaimg.cn/large/007S8ZIlly1gja1c6hdlzj309h02b3yj.jpg)

可以看到一个master和一个node，但是状态都是NotReady，表示还没有准备好。这时候离安装成功还差几步。此时切换到master机器，查询kube-system命名空间下各pod的状态。

```shell
kubectl get pod -n kube-system
```

![image-20201001192633839](https://tva1.sinaimg.cn/large/007S8ZIlly1gja22vtgsjj30gh05ot9j.jpg)

我们发现kube-proxy处于ContainerCreating状态，coredns处于Pending状态，只要把这些问题解决了，所有pod都Running状态，集群就正常了。

我们通过命令查询以下kube-proxy的详细信息情况：

```shell
kubectl describe pod kube-proxy-pm5f6 -n kube-system
```

查看最下面的Event信息

```shell
Events:
  Type     Reason                  Age                    From                Message
  ----     ------                  ----                   ----                -------
  Warning  FailedCreatePodSandBox  3m45s (x139 over 84m)  kubelet, k8s-node1  Failed to create pod sandbox: rpc error: code = Unknown desc = failed pulling image "k8s.gcr.io/pause:3.2": Error response from daemon: Get https://k8s.gcr.io/v2/: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
```

从Event信息中可以看到在节点k8s-node1上拉取镜像k8s.gcr.io/pause:3.2失败，很明显这又是从Google官网下载镜像的，我们依然使用之前的方式，但是要在k8s-node1机器上执行：

```shell
[root@k8s-node1 ~]# docker pull gotok8s/pause:3.2
[root@k8s-node1 ~]# docker tag gotok8s/pause:3.2 k8s.gcr.io/pause:3.2
[root@k8s-node1 ~]# docker rmi gotok8s/pause:3.2
```

执行完后，稍等以下，在k8s-node1机器上会看到pause容器已经被拉起了，这时候在master上查看pod状态，会发现kube-proxy的pod还没有正常，状态变为ImagePullBackOff

![image-20201001202357237](https://tva1.sinaimg.cn/large/007S8ZIlly1gja3qlg1t7j30fi05pwfb.jpg)

继续使用kubectl describe查询pod详情

![image-20201001202526230](https://tva1.sinaimg.cn/large/007S8ZIlly1gja3s55ynxj30rf04r0ti.jpg)

Event里面没有具体的错误，但是我们可以往上面再看一看，会发现node上面还需要一个镜像：k8s.gcr.io/kube-proxy:v1.18.8

![image-20201001202838334](https://tva1.sinaimg.cn/large/007S8ZIlly1gja3vh1wdqj30hy0hytaq.jpg)

以同样的方式在node上下载镜像

```shell
[root@k8s-node1 ~]# docker pull gotok8s/kube-proxy:v1.18.8
[root@k8s-node1 ~]# docker tag gotok8s/kube-proxy:v1.18.8 k8s.gcr.io/kube-proxy:v1.18.8
[root@k8s-node1 ~]# docker rmi gotok8s/kube-proxy:v1.18.8
```

执行完后，在master机器上再查询pod状态，会发现kube-proxy已经Running了。

![image-20201001203132459](https://tva1.sinaimg.cn/large/007S8ZIlly1gja3yhpb9tj30e505mwfa.jpg)

接下来就是处理coredns的问题了，这是因为没有安装网络组件而造成的，我们选择Flannel网络组件，安装命令如下：

```shell
# 访问GitHub上的flannel仓库文件，将内容保存在本地/home/kube-flannel.yml
https://github.com/coreos/flannel/blob/master/Documentation/kube-flannel.yml

# 执行安装命令
[root@k8s-master ~]# cd /home
[root@k8s-master ~]# kubectl apply -f kube-flannel.yml
```

![image-20201001204506858](https://tva1.sinaimg.cn/large/007S8ZIlly1gja4cmd458j30ce040t94.jpg)

然后查看pod信息

![image-20201001204620815](https://tva1.sinaimg.cn/large/007S8ZIlly1gja4dwjf3wj30ef06pt9q.jpg)

可以发现多处了两个名为kube-flannel-ds，而且状态也不是Running，同样可以使用kubectl describe pod kube-flannel-ds-bjvjx -n kube-system

![image-20201001210818914](https://tva1.sinaimg.cn/large/007S8ZIlly1gja50r95hvj30qr05o75d.jpg)

这里是k8s-node1机器缺少镜像quay.io/coreos/flannel:v0.13.0-rc2，还是需要手动下载处理。但是这里有些问题，外网下载不了，国内网站也没有这个版本的，所以只能退而求其次，找第一个版本的，直接在GitHub上下载coreos/flannel的版本。

![image-20201001225833376](https://tva1.sinaimg.cn/large/007S8ZIlly1gja87gjz35j30gk0i20uj.jpg)

下载后通过docker load -i flanneld-v0.12.0-adm64.docker导入镜像

![image-20201001230353666](https://tva1.sinaimg.cn/large/007S8ZIlly1gja8d0hpjpj30dl048wf3.jpg)

然后先删除掉kube-flannel.yml的创建

```shell
[root@k8s-master home]# kubectl delete -f kube-flannel.yml
```

![image-20201001230613807](https://tva1.sinaimg.cn/large/007S8ZIlly1gja8fg05x0j30cl03uaag.jpg)

修改kube-flannel.yml中需要的镜像版本v0.13.0-rc2替换为v0.12.0-amd64，然后重新apply。

> 当然还有另外一种方式，如果你有云服务器的话，可以在云服务器上下载quay.io/coreos/flannel:v0.13.0-rc2，再通过docker save -o flannel.tar quay.io/coreos/flannel:v0.13.0-rc2将镜像保存为flannel.tar包，下载到本地，通过docker load -i flannel.tar加载镜像即可，就不用修改kube-flannel.yml了

Flannel安装完成后（要在每个节点上都安装，看pod数量就知道了），查看一下pod的状态

```shell
[root@k8s-master home]# kubectl get pod -n kube-system
```

![image-20201001231329393](https://tva1.sinaimg.cn/large/007S8ZIlly1gja8n0dzbpj30ef06s3zh.jpg)

所有的pod都是Running状态，然后查看以下所有node的状态

```shell
[root@k8s-master home]# kubectl get nodes
```

![image-20201001231448330](https://tva1.sinaimg.cn/large/007S8ZIlly1gja8od8hmuj3092025wei.jpg)

可以发现节点都已处于Ready状态。

## 总结

这一篇讲了kubernetes的安装与配置方法，即使网络受限，并且不使用代理的情况下也可以安装，再扩展一下，如果把所需的镜像都打包好，也可以完全离线安装。写的比较啰嗦一点，但是最重要的事试错、差错、找问题，然后解决问题的方法，在这篇中我们要学会一个以后会长期使用到的kubectl命令：kubectl describe。



---

### 新增一个Node节点

- 修改hostname

  ```shell
  vi /etc/hostname
  
  # 内容
  k8s-node5
  
  # 再执行命令
  hostnamectl set-hostname k8s-node5
  ```

- 修改hosts（在集群所有节点上执行）

  ```shell
  vi /etc/hosts
  
  # 增加一行
  192.168.143.135 k8s-node5
  ```

- 关闭防火墙

  ```shell
  systemctl disable firewalld.service
  systemctl stop firewalld.service
  ```

- 关闭SELinux

  ```shell
  setenforce 0
  sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
  ```

- 关闭swap

  ```shell
  # 临时关闭
  swapoff -a
  
  # 永久关闭
  sed -ri 's/.*swap.*/#$/' /etc/fstab
  ```

- iptables流量转发设置

  ```shell
  echo "1">/proc/sys/net/ipv4/ip_forward
  echo "1">/proc/sys/net/bridge/bridge-nf-call-iptables
  ```

- 在master上创建token

  ```shell
  # 1. 查询是否存在token，默认24小时过期，如果没有则需要创建
  kubeadm token list
  ```

  ![image-20201018151742886](https://tva1.sinaimg.cn/large/007S8ZIlly1gjtif8prnvj30k803w0t9.jpg)

  如果有token，直接在下面添加节点的命令中使用就行，如果没有token，则在master上执行下面的两条命令创建

  ```shell
  kubeadm token create
  openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
  ```

- 添加节点到集群中

  ```shell
  kubeadm join k8s-master:6443 --token ofjbby.fsg22ziviq2gl9lr --discovery-token-ca-cert-hash sha256:684bd6c7f33db9e4c28f5006d896f0fd02015fe2bfc5b186202a6ab00507ba68 -v=10
  ```

- 修改docker加速

  ```shell
  mv /etc/docker/daemon.json.rpmsave /etc/docker/daemon.json
  
  systemctl restart docker
  ```

- 下载必要的容器

  ```shell
  docker pull gotok8s/pause:3.2
  docker tag gotok8s/pause:3.2 k8s.gcr.io/pause:3.2
  docker rmi gotok8s/pause:3.2
  docker pull gotok8s/kube-proxy:v1.18.8
  docker tag gotok8s/kube-proxy:v1.18.8 k8s.gcr.io/kube-proxy:v1.18.8
  docker rmi gotok8s/kube-proxy:v1.18.8
  ```

### 重新挂载节点

- 在master上删除节点

  ```shell
  kubectl delete node node_name
  ```

- 在节点上清理

  ```shell
  rm -f /etc/kubernetes/pki/ca.crt
  rm -f /var/lib/kubelet/pki/kubelet-client-current.pem
  rm -f /etc/kubernetes/kubelet.conf
  
  systemctl stop kubelet
  ```

- 添加节点到集群中

  ```shell
  kubeadm join k8s-master:6443 --token ofjbby.fsg22ziviq2gl9lr --discovery-token-ca-cert-hash sha256:684bd6c7f33db9e4c28f5006d896f0fd02015fe2bfc5b186202a6ab00507ba68
  ```

  