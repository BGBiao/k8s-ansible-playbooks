## k8s集群中的一些核心manifest

### 1. myapp

`说明:无状态的业务应用(检测K8s集群核心功能)`

```
$ kubectl  apply -f myapp.yml
namespace/myapp configured
secret/mydocker configured
service/myapp configured
deployment.apps/myapp configured

$ kubectl  get deploy,svc -n myapp -o wide
NAME                          READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS   IMAGES                                                           SELECTOR
deployment.extensions/myapp   5/5     5            5           2m59s   myapp        harbor.bgbiao.cn/soul-ops/cmdb-api-python:2019-09-05T1707   name=myapp

NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE     SELECTOR
service/myapp   ClusterIP   10.253.110.56   <none>        8080/TCP   2m59s   name=myapp

$ curl 10.253.110.56:8080
Hello World!

```


### 2. coredns

`注意:kubelet后期由于整个集群内部需要各个节点互相通信，因此coredns是必须存在的一个组件`

```
$ kubectl apply -f coredns.yml

# coredns相关pod
$ kubectl  get pods -n kube-system
NAME                       READY   STATUS    RESTARTS   AGE
coredns-6fb94bcbb4-68sqr   0/1     Running   0          25m
coredns-6fb94bcbb4-hn864   0/1     Running   0          25m


# 使用busybox容器来测试dns是否正常(发现只能解析pod自己本身的hostname，其他hostname无法解析，且无法解析外网域名)

$ kubectl exec busybox -it sh
/ # cat /etc/resolv.conf
nameserver 10.253.0.250
search default.svc.cluster.local. svc.cluster.local. cluster.local.
options ndots:5
/ # hostname
busybox
/ # nslookup busybox
Server:    10.253.0.250
Address 1: 10.253.0.250

Name:      busybox
Address 1: 20.0.14.4 busybox

/ # nslookup www.baidu.com
Server:    10.253.0.250
Address 1: 10.253.0.250
nslookup: can't resolve 'www.baidu.com'

# 查看dns的pod日志发现如下问题(应该是创建master证书的时候没有把10.253.0.1加上)
E0906 06:40:22.712612       1 reflector.go:125] pkg/mod/k8s.io/client-go@v0.0.0-20190620085101-78d2af792bab/tools/cache/reflector.go:98: Failed to list *v1.Endpoints: Get https://10.253.0.1:443/api/v1/endpoints?limit=500&resourceVersion=0: x509: certificate is valid for 172.16.21.23, 172.16.21.24, 172.16.21.25, 172.16.21.26, 172.16.21.27, 172.16.21.29, not 10.253.0.1

# 解决办法:修改k8s-csr.json中的ip地址，增加127.0.0.1和10.253.0.1(svc的地址)重新生成k8s证书，重启整个集群

$ kubectl exec busybox -it sh
/ # cat /etc/resolv.conf
nameserver 10.253.0.250
search default.svc.cluster.local. svc.cluster.local. cluster.local.
options ndots:5
/ # nslookup kubernetes
Server:    10.253.0.250
Address 1: 10.253.0.250

Name:      kubernetes
Address 1: 10.253.0.1
/ # nslookup baidu.com
Server:    10.253.0.250
Address 1: 10.253.0.250

Name:      baidu.com
Address 1: 220.181.38.148
Address 2: 39.156.69.79

```

### 3. traefik-ingress

`注意`:traefik是一种ingress-controller,和官方的`nginx-controller`相似，用来统一对外提供流量入口，不同的是`nginx-controller`默认支持L4和L7层的负载均衡，`traefik-controller`默认仅支持L7层负载和代理，需要借助`Mesh`之类的东西来实现L4层的流量管控.

```
# 创建traefik集群
$ kubectl apply -f traefik/

# 查看集群部署状态
# daemonset方式部署会在每个可调度的node节点上部署一个traefik
$ kubectl  get deploy,daemonset,service,ingress -n kube-system | grep traefik

daemonset.extensions/traefik-ingress-controller   3         3         3       3            3           <none>          3d18h
service/traefik-ingress-service   ClusterIP   10.253.81.132   <none>        80/TCP,8080/TCP          3d18h

service/traefik-web-ui            ClusterIP   10.253.59.218   <none>        80/TCP                   3d18h
ingress.extensions/traefik-web-ui   prod-traefik-ui.bgbiao.cn             80      3d18h

# 测试访问
# 每个node节点都应该可以
$ curl -H 'host:prod-traefik-ui.bgbiao.cn' 172.16.21.28
<a href="/dashboard/">Found</a>.

$ curl -H 'host:prod-traefik-ui.bgbiao.cn' 172.16.21.26
<a href="/dashboard/">Found</a>.

$ curl -H 'host:prod-traefik-ui.bgbiao.cn' 172.16.21.27
<a href="/dashboard/">Found</a>.

```

### 4. dashboard

`注意:dashboard是一个官方的集群管理和操作的web管理系统(默认开启了认证和https)，内部简单查询使用，暂时关闭了https和认证`

```
# 使用官方示例改造成http之后直接创建(1.5.0版本，支持汉化)
$ kubectl -f dashboard/kubernetes-dashboard-http.yaml

# 或者(当前dashboard官方最新版本，不支持汉化)
$ kubectl -f dashboard/kubernetes-dashboard-httpv2.0.0-beat4.yaml
 
# 查看dashboard相关示例
$ kubectl  get svc,ingress -A  | grep dashboard

kubernetes-dashboard   service/dashboard-metrics-scraper   ClusterIP   10.253.61.79    <none>        8000/TCP                       46h
kubernetes-dashboard   service/kubernetes-dashboard        NodePort    10.253.233.70   <none>        443:48382/TCP,9090:43275/TCP   46h
kubernetes-dashboard   ingress.extensions/kubernetes-dashboard   prod-dashboard.bgbiao.cn              80      44h

# 测试dashboard(分别测试traefik暴露出的ingress地址和dashboard本身的service地址)
$ curl -H 'host:prod-dashboard.bgbiao.cn' 172.16.21.27 -I
HTTP/1.1 200 OK
Accept-Ranges: bytes
Cache-Control: no-store
Content-Length: 1262
Content-Type: text/html; charset=utf-8
Date: Tue, 10 Sep 2019 02:17:16 GMT
Last-Modified: Thu, 29 Aug 2019 09:14:59 GMT

$ curl 10.253.233.70:9090 -I
HTTP/1.1 200 OK
Accept-Ranges: bytes
Cache-Control: no-store
Content-Length: 1262
Content-Type: text/html; charset=utf-8
Last-Modified: Thu, 29 Aug 2019 09:14:59 GMT
Date: Tue, 10 Sep 2019 02:17:19 GMT


```

### 5. prometheus

`注意:prometheus是一个云原生的分布式监控系统，可以实时对K8s集群资源和服务进行监控和报警` 


### 6. elk

`注意:集群的业务日志需要统一的进行分析和处理，采用开源成熟的elk方案`

- log-polit: 日志采集到指定的存储[es,kafka]
- es|kafka: 集群存储
- kibana: 日志统一查询
