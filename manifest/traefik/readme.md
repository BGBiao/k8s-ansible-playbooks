## Treafik集群创建

**traefikv-1.X集群部署**

`注意:当前采用traefikv1.17.6版本`

```
# 创建traefik容器
$ kubectl apply -f traefik/

# 查看traefik容器信息
$ kubectl  get pods -n kube-system -o wide
NAME                               READY   STATUS    RESTARTS   AGE     IP          NODE           NOMINATED NODE   READINESS GATES
coredns-6fb94bcbb4-h8pzv           1/1     Running   0          40m     20.0.38.2   172.29.202.159   <none>           <none>
traefik-ingress-controller-g65rf   1/1     Running   0          8m55s   20.0.87.8   172.29.202.159   <none>           <none>


# 注意:traefil-ingress-controller可以使用daemonset和deploy部署，前者会主动调度到node节点，实际过程中会专门隔离几个节点作为网关节点进行统一入口管理(会在每个node节点上启动一个类似nginx的代理暴露到主机的80端口)
$ curl -H 'host:prod-traefik-ui.bgbiao.cn' 172.29.202.159
<a href="/dashboard/">Found</a>.

```

查看traefikv1.x版本的dashboard:

![](https://tva1.sinaimg.cn/large/006y8mN6ly1g7ec1e2ufij31i30u010k.jpg)

**traefik-v2.X集群部署**

`注意:当前采用traefikv2.0.1版本`

```
# 创建traefik2.0.1版本
# 注意:由于traefik本身需要serviceaccount和rbac权限，在上面v1.17.6中我们已经创建，这里仅创建和traefik相关的控制器
$ kubectl apply -f traefik-ds-v2.0.1.yaml 

# 查看traefik的daemonset信息(可以看到两个集群都部署成功了)
$ kubectl get daemonset  -n kube-system
NAME                            DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
traefik-ingress-controller      4         4         4       4            4           <none>          16m
traefik-ingress-controller-v2   4         4         4       4            4           <none>          11m


# 测试访问下
# 注意:当前环境中traefikv1.17.6和traefikv2.0.1并存，因此后者使用了81和8081端口来提供服务
$ curl -H 'host:prod-traefik-ui.bgbiao.cn' 172.29.202.159
<a href="/dashboard/">Found</a>.

$ curl -H 'host:prod-traefik-ui.bgbiao.cn' 172.29.202.159:81
<a href="/dashboard/">Found</a>.
$ 
```

查看traefikv2.0.1的dashboard:

![](https://tva1.sinaimg.cn/large/006y8mN6ly1g7ec1gi9q8j32700pk44p.jpg)
