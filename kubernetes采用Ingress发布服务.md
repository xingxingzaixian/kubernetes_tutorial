​      我们要将kubernetes集群内的服务发布到集群外使用，之前使用的方法都是 NodePort、LoadBalancer的 Service，或者是给 Service 配置 ExternalIP，也可以通过 Pod 的 HostPort 进行配置。但是这些方式都存在一些问题，几乎都是通过节点端口的形式向外暴露服务的，了解 Service 的人应该知道，通过 Service 向外暴露端口，实际是在集群中的所有节点上监听同一个端口，如果 Service 非常多，那每个节点上开启的端口就会变得很多，这样维护起来很复杂，而且也非常不安全。  

​      使用 Ingress 可以解决这个问题，除了 Ingress 自身的服务向外发布以外，其他服务都不需要直接向外发布。用 Ingress 接收所有的外部请求，然后按照域名配置转发给对应的服务。

# Ingress 简介

​      Ingress 包含 3 个组件

- 反向代理负载均衡器

  这个类似 Nginx、Apache ，在集群中可以使用 Deployment、DaemonSet 等控制器来部署反向代理负载均衡器。

- Ingress 控制器

  作为一个监控器不停的与 API Server 进行交互，实时的感知后端 Service、Pod 等的变化情况，例如新增或者减少，得到这些变化信息后，Ingress 控制器再结合 Ingress 服务自动生成配置，然后更新反向代理负载均衡器并且刷新其配置，达到服务发现的作用。

- Ingress 服务

  定义访问规则，加入某个域名对应某个 Service，或者某个域名下的子路径对应某个 Service，那么当这个域名的请求进来时，就把请求转发给对应的 Service。根据这个规则，Ingress 控制器会将访问的规则动态写入负载均衡器的配置中，从而实现整体的服务发现和负载均衡。

Ingress 控制器的种类有很多种，但是在基本使用层面几乎没有差别，下面我们使用 Ingress-Nginx 控制器来展示一下 Ingress 的基本使用。

# 创建 Ingress-Nginx 控制器

下载官方部署文件 https://github.com/kubernetes/ingress-nginx

![image-20201101133555001](https://tva1.sinaimg.cn/large/0081Kckwly1gk9m5mt03fj30xx0fndio.jpg)

选择Tags对应的版本，我这里选择最新的，这个要看 kubernetes 集群的版本，如果版本比较低，有些 API 是不存在的，所以不一定要最新的版本，根据情况下载即可。下载好部署文件https://github.com/kubernetes/ingress-nginx/blob/nginx-0.30.0/deploy/static/mandatory.yaml，需要做一些简单的修改，在 Deployment 服务中指定网络方式为 hostNetwork: true，这样就可以在集群外部访问 Ingress 服务。

![image-20201101140918761](https://tva1.sinaimg.cn/large/0081Kckwly1gk9n4c5d0xj30gj05t3z0.jpg)

执行部署：

```shell
kubectl apply -f mandatory.yaml
```

![image-20201101140243137](https://tva1.sinaimg.cn/large/0081Kckwly1gk9mxheas9j30i406nt9n.jpg)

# 测试 Ingress 服务

- 创建两个服务 Nginx/Httpd

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: nginx
  spec:
    replicas: 1
    selector:
      matchLabels:
        example: nginx
    template:
      metadata:
        labels:
          example: nginx
      spec:
        containers:
        - name: nginx
          image: nginx:latest
          ports:
          - containerPort: 80
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: nginx
  spec:
    selector:
      example: nginx
    ports:
      - protocol: TCP
        port: 80
        targetPort: 80
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: httpd
  spec:
    replicas: 1
    selector:
      matchLabels:
        example: httpd
    template:
      metadata:
        labels:
          example: httpd
      spec:
        containers:
        - name: httpd
  			  image: httpd:latest
          ports:
          - containerPort: 80
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: httpd
  spec:
    selector:
      example: httpd
    ports:
      - protocol: TCP
        port: 80
        targetPort: 80
  ```

  执行创建命令：`kubectl apply -f service.yml`

  测试一下服务是否可以正常访问

  ```shell
  kubect get all
  ```

  ![image-20201101150346562](https://tva1.sinaimg.cn/large/0081Kckwly1gk9op08enwj30gs09l75m.jpg)

  在解决节点的任意机器上访问服务：

  ```shell
  curl 10.111.194.232
  ```

  ![image-20201101150510428](https://tva1.sinaimg.cn/large/0081Kckwly1gk9oqgnef8j3094019glj.jpg)

  ```shell
  curl 10.109.202.249
  ```

  ![image-20201101150550190](https://tva1.sinaimg.cn/large/0081Kckwly1gk9or5gyjrj30cf098750.jpg)

### 创建 Ingress

#### 基于多域名配置

  ```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: webservice
  annotations:
    Kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: www.httpd.com
    http:
      paths:
      - path: /
        backend:
          serviceName: httpd
          servicePort: 80
  - host: www.nginx.com
    http:
      paths:
      - path: /
        backend:
          serviceName: nginx
          servicePort: 80
  ```

  执行创建命令：`kubectl apply -f ingress.yml`

  查询创建的 Ingress 信息：`kubectl get ingress`

  ![image-20201101151537114](https://tva1.sinaimg.cn/large/0081Kckwly1gk9p1c3q7gj30ek01uq2z.jpg)

  可以看到 HOSTS 中显示的域名信息

- 查看 Ingress-Nginx pod 负载均衡信息

  ```shell
  kubectl get po -n ingress-nginx
  ```

   ![image-20201101155047368](https://tva1.sinaimg.cn/large/0081Kckwly1gk9q1x8e1xj30f101n3yk.jpg)

  ```shell
  # 进入 nginx-ingress-controller Pod 容器
  kubectl exec -it nginx-ingress-controller-77db54fc46-q8vp7 -n ingress-nginx -- /bin/bash
  
  # 查看运行的进程
  ps -ef
  ```

  ![image-20201101155217313](https://tva1.sinaimg.cn/large/0081Kckwly1gk9q3hfb7ej30sl051q3t.jpg)

  可以看到 nginx-ingress-controller Pod 容器中运行的是 nginx 程序，我们再查看以下 nginx.conf 配置文件

  ![image-20201101155817677](https://tva1.sinaimg.cn/large/0081Kckwly1gk9q9qnt7hj30aw094wet.jpg)

  ![image-20201101155847381](https://tva1.sinaimg.cn/large/0081Kckwly1gk9qa9b1gij308x08z3yt.jpg)

  配置文件中自动生成了 www.httpd.com 和 www.nginx.com 的服务配置


#### 访问测试

- 查询 Ingress-Nginx 部署节点

  ```shell
  kubectl get po -n ingress-nginx -o wide
  ```

  ![image-20201101151606467](https://tva1.sinaimg.cn/large/0081Kckwly1gk9p1ukgfdj30rn01rt8w.jpg)

- 修改本机的 hosts 文件

  ```shell
  vi /etc/host
  
  192.168.143.132 www.nginx.com
  192.168.143.132 www.httpd.com
  ```

- 访问网站，查看内容

  ![image-20201101161429981](https://tva1.sinaimg.cn/large/0081Kckwly1gk9qqlgq8xj30mx03nwem.jpg)

  ![image-20201101161456654](https://tva1.sinaimg.cn/large/0081Kckwly1gk9qr2fh7sj30g30730ti.jpg)

  ### 基于多个子域名配置

  ```yaml
  apiVersion: extensions/v1beta1
  kind: Ingress
  metadata:
    name: web
    annotations:
      Kubernetes.io/ingress.class: "nginx"
  spec:
    rules:
    - host: webservice.com
      http:
        paths:
        - path: /httpd
          backend:
            serviceName: httpd
            servicePort: 80
        - path: /nginx
          backend:
            serviceName: nginx
            servicePort: 80
  ```
  
  其他的配置测试与上面一致。
  
  