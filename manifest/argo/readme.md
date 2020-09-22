## 云原生领域的工作流和CD工具-Argo

Argo 相关项目: 

- [Argo 工作流引擎](https://github.com/argoproj/argo)
- [Argo-CD GitOps引擎](https://github.com/argoproj/argo-cd)
- - [Argo-CD 示例](https://github.com/argoproj/argocd-example-apps)
- - [Argo-CD 通知](https://github.com/argoproj-labs/argocd-notifications)

### Argo 工作流引擎

Argo Workflows是一个开源的容器本地工作流引擎，用于在Kubernetes上编排并行作业。Argo工作流实现为Kubernetes CRD(自定义资源定义)。

- 定义工作流，其中工作流中的每一步都是一个容器。
- 将多步骤工作流建模为任务序列，或者使用有向无环图(DAG)获取任务之间的依赖关系。
- 使用Kubernetes上的Argo工作流，只需花费一小部分时间就可以轻松运行用于机器学习或数据处理的计算密集型作业。
- 在Kubernetes上本地运行CI / CD管道而无需配置复杂的软件开发产品。

[Argo 工作流引擎](https://argoproj.github.io/argo/quick-start/)

```

# 快速开始
$ kubectl create ns argo
$ kubectl apply -n argo -f https://raw.githubusercontent.com/argoproj/argo/stable/manifests/quick-start-postgres.yaml


# 注意: 快速构建版本，使用了postgresql 存储后端数据；且集群多组件需要容器内部的dns进行通信，因此coredns必须可以使用
# 可以看到argo 工作流引擎，使用了minio 作为对象存储，使用postgres 作为关系数据存储，然后通过argo-server 来对外暴露工作流引擎，内部调用workflow的CRD来实现整个工作流的编排和运行。

$ kubectl get pods -n argo | grep Running
argo-server-6fbf8d9bf-rxnjd            1/1     Running     4          18h
minio                                  1/1     Running     0          4d15h
postgres-6b5c55f477-l99mw              1/1     Running     0          18h
workflow-controller-76f8dbb879-zk96h   1/1     Running     0          18h


# 查看并修改argo 的service
# 注意: 默认的argo-service 是没有开启NodePort 的，我们可以开启或者使用Ingress方案将服务对外
$ kubectl get svc -n argo
NAME                          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
argo-server                   NodePort    10.253.48.71     <none>        2746:35627/TCP   4d15h
minio                         ClusterIP   10.253.227.137   <none>        9000/TCP         4d15h
postgres                      ClusterIP   10.253.181.132   <none>        5432/TCP         4d15h
workflow-controller-metrics   ClusterIP   10.253.161.62    <none>        9090/TCP         4d15h


```

接下来，访问http://NodeIP:35627 即可以访问 Argo 工作流引擎。

官方提供了几种工作流模板，可以尝试去做一些事情。

### Argo-CD GitOps 解决方案

Argo CD是一款面向Kubernetes的GitOps持续交付工具。

[Argo-cd 官方示例](https://cd.apps.argoproj.io/applications/argo-cd)

[Argo-cd 官方文档](https://argoproj.github.io/argo-cd/)

```
# 快速开始

$ kubectl create namespace argocd
$ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

$ kubectl get all -n argocd
NAME                                                 READY   STATUS    RESTARTS   AGE
pod/argocd-application-controller-6766567bf4-f8c48   1/1     Running   0          4d19h
pod/argocd-dex-server-7965ffd4c4-7bnn7               1/1     Running   0          18h
pod/argocd-redis-cccbb8f7-tnr4z                      1/1     Running   0          18h
pod/argocd-repo-server-57c54dd4d8-rc96d              1/1     Running   0          4d19h
pod/argocd-server-55c7767565-flf67                   1/1     Running   0          4d18h

NAME                            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/argocd-dex-server       ClusterIP   10.253.148.219   <none>        5556/TCP,5557/TCP,5558/TCP   4d19h
service/argocd-metrics          ClusterIP   10.253.119.35    <none>        8082/TCP                     4d19h
service/argocd-redis            ClusterIP   10.253.177.0     <none>        6379/TCP                     4d19h
service/argocd-repo-server      ClusterIP   10.253.240.241   <none>        8081/TCP,8084/TCP            4d19h
service/argocd-server           NodePort    10.253.142.245   <none>        80:42790/TCP,443:31738/TCP   4d19h
service/argocd-server-metrics   ClusterIP   10.253.178.177   <none>        8083/TCP                     4d19h

NAME                                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/argocd-application-controller   1/1     1            1           4d19h
deployment.apps/argocd-dex-server               1/1     1            1           4d19h
deployment.apps/argocd-redis                    1/1     1            1           4d19h
deployment.apps/argocd-repo-server              1/1     1            1           4d19h
deployment.apps/argocd-server                   1/1     1            1           4d19h

NAME                                                       DESIRED   CURRENT   READY   AGE
replicaset.apps/argocd-application-controller-6766567bf4   1         1         1       4d19h
replicaset.apps/argocd-dex-server-7965ffd4c4               1         1         1       4d19h
replicaset.apps/argocd-redis-cccbb8f7                      1         1         1       4d19h
replicaset.apps/argocd-repo-server-57c54dd4d8              1         1         1       4d19h
replicaset.apps/argocd-server-55c7767565                   1         1         1       4d18h
replicaset.apps/argocd-server-795cf55b85                   0         0         0       4d19h


```

可以看到Argo-CD 有几个核心的组件，以此来负责整个基于Git的持续交付流程(将代码从git中拉取下来，并实时发布到k8s集群中，并且能检测到相关的配置变动)

![Argo-CD 架构图](https://tva1.sinaimg.cn/large/007S8ZIlgy1gizh7gwbogj30kn0jojtj.jpg)

部署成功后，即可访问Argo-CD。

![argo-cd登录](https://tva1.sinaimg.cn/large/007S8ZIlly1gityu89rk2j31cy0u0tfr.jpg)

![argo-cd主页面](https://tva1.sinaimg.cn/large/007S8ZIlly1gitywi6y1kj31im0u0aew.jpg)


