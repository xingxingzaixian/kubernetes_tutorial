# 五分钟极速搭建kubernetes集群

kubernetes的集群搭建有多种方式：二进制、kubeadm、ansible自动化、minikube。minikube方式比较简单，但是只是单节点，适合学习kubernetes基础的时候使用。其他的方式安装都会出各种问题。我花了一个星期，尝试了这几种方式，除了minikube，其他都没有成功。从centos到Ubuntu，心力交瘁。

前几天在跟同事聊天的时候，我对他说，kubernetes的学习终止于环境搭建。同事神秘的一笑，对我说，我有一个秘籍，五分钟搭建、百分百成功，看你骨骼惊奇，一包辣条卖给你吧。

![image-20200827231534216](https://tva1.sinaimg.cn/large/007S8ZIlly1gi5s0g2gtzj309408yju9.jpg)

我以不相信的语气说，憋™吹牛逼了，你倒是说呀。同事伸出两根手指

- 拉取rancher容器
- 在rancher界面安装kubernetes

超过五分钟，说明你网速不好。我一听果然是我不知道的一种方式，下班后立马回家尝试，于是有了以下的记录。

## 主机准备

| 操作系统 | 主机名     | IP地址          | 配置  |
| -------- | ---------- | --------------- | ----- |
| CentOS 7 | k8s-master | 192.168.143.130 | 2核2G |
| CentOS 7 | k8s-node1  | 192.168.143.131 | 2核2G |
| CentOS 7 | K8s-node2  | 192.168.143.132 | 2核2G |

我们一定要保证以上环境的初始化信息正常，主要是IP和主机名，IP一定要配置为静态IP，主机名一定要修改为不同的，具体可参考上一篇文章

## k8s-master上执行初始化

- 安装rancher

  ```shell
  # 拉取rancher/rancher:v2.0.0镜像，这个是在docker hub上，不推荐使用最新的rancher镜像
  docker run -d --restart=always -p 80:80 -p 443:443 rancher/rancher:v2.0.0
  ```

- 配置时间同步

  ```shell
  # 在所有机器上安装
  yum install -y ntp ntpdate
  
  # ************在192.168.143.130上修改***********
  # 修改/etc/ntp.conf，注释掉server开头的几行，增加如下
  server ntp3.aliyun.com iburst
  
  # 192.168.143.2是网关地址
  restrict 192.168.143.2 mask 255.255.255.0 nomodify notrap
  
  # 注释掉这一行
  # restrict 127.0.0.1
  ```
  
- 开放主机端口
  
  主要是方便各主机之间的通信，这些命令要在所有主机上都执行
  
  ```shell
  firewall-cmd --zone=public --add-port=2379/tcp --permanent
  firewall-cmd --zone=public --add-port=2380/tcp --permanent
  firewall-cmd --zone=public --add-port=6443/tcp --permanent
  firewall-cmd --zone=public --add-port=10250/tcp --permanent
  firewall-cmd --reload
  ```
  
  

## 其他主机初始化

```shell
yum install -y ntp ntpdate

# 修改/etc/ntp.conf，注释掉server开头的几行，增加如下
server 192.168.143.130 prefer

# 注释掉这一行
# restrict 127.0.0.1

# 保存以上文件后，在控制台执行以下命令同步一次
ntpq -p
```



## 在rancher上执行初始化操作

上面我们已经在主机上安装好了rancher，通过浏览器打开https://192.168.143.130，先设置admin用户的密码，然后下一步就行了

![image-20200826005115021](https://tva1.sinaimg.cn/large/007S8ZIlly1gi3jjd34x6j317v0pi75s.jpg)

  修改语言为简体中文，然后点击左上角的集群，再点击右边的添加集群

![image-20200826005215097](https://tva1.sinaimg.cn/large/007S8ZIlly1gi3jkd71cvj311p067gm6.jpg)

选择Custom，表示自建kubernetes集群，如果已经有了集群，可以点击import，导入现有集群

![image-20200826005332196](https://tva1.sinaimg.cn/large/007S8ZIlly1gi3jlphx0ij31360j1dhk.jpg)

名称随便填，其他的先不用管，点击下一步

![image-20200826005527218](https://tva1.sinaimg.cn/large/007S8ZIlly1gi3jnp3nwcj313w0l3q5u.jpg)

首先把三个都选中，我们这里是要先创建kubernetes集群的master节点，所以必须都选中，否则创建不成功，然后点击生成命令旁边的复制，复制需要执行的命令后，在我们的k8s-master主机执行此命令，执行成功后会在以上界面的左下角显示一台主机已注册，然后点击Done

![image-20200826005826575](https://tva1.sinaimg.cn/large/007S8ZIlly1gi3jr5uoenj312j0aogmn.jpg)

回到集群界面，点击集群右边的三个小点，选择编辑，有些版本这里显示的升级，就点击升级按钮

![image-20200826010129931](https://tva1.sinaimg.cn/large/007S8ZIlly1gi3jtzq23xj312s0nrq5t.jpg)

在这个界面选中控制和工作节点，然后复制命令，在其他的主机上去执行命令，每个主机执行完后，都会在左下角显示主机已注册，都执行完后点击保存。每个主机加入集群后都会进行一番初始化操作，这个时间视机器的情况而定。

![image-20200826011005275](https://tva1.sinaimg.cn/large/007S8ZIlly1gi3k2xbzp1j30fw066wet.jpg)

主机注册完后，就会在上面这里看到我们已经注册的集群，点击后显示如下

![image-20200826011108662](https://tva1.sinaimg.cn/large/007S8ZIlly1gi3k40q1y1j312i0kracm.jpg)

![image-20200826011141089](https://tva1.sinaimg.cn/large/007S8ZIlly1gi3k4kvq5jj314m0ad75v.jpg)

点击节点就可以看到我们的主机机器，我们的三节点kubernetes集群就搭建完成了。是不是非常简单，而且是界面化操作，非常方便，几乎不会出现错误。

rancher可以安装kubernetes，也提供了很多操作，但是我还是建议通过kubectl命令进行操作。如果你已经非常熟悉了，使用可视界面能提高效率，如果你刚接触kubernetes，命令行方式跟利于学习。

## 异常问题

在搭建过程中可能出现的问题是机器初始化的问题，比如多个机器的主机名相同或者IP地址相同，只要注意这两点，几乎不会出任何问题。