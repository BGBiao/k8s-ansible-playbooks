### k8s-dashboard相关文档

`注意:dashboard默认开启了https并且开启了token和tls认证，一般而言内部系统加上token和tls认证之后反倒比较复杂和麻烦,因此可以想办法将http暴露出来`

[dashboard-http-all](./kubernetes-dashboard-http.yaml)

[dashboard-http-v2.0.0-beat4](./kubernetes-dashboard-httpv2.0.0-beat4.yaml)

```
# yaml配置下载

$ wget https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml

# 修改镜像:k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1 为registry.cn-hangzhou.aliyuncs.com/google_containers/kubernetes-dashboard-amd64:v1.10.1

$ kubectl apply -f kubernetes-dashboard.yaml


$ kubectl  get pods,svc -n kube-system | grep dash

pod/kubernetes-dashboard-86844cc55f-rfj4d   1/1     Running   0          4m25s
service/kubernetes-dashboard      ClusterIP   10.253.255.27   <none>        443/TCP                  4m25s



```

**http支持**

修改几个地方:

- 1. container部分将容器的9090暴露出来(默认暴露了8443)
- 2. 修改启动参数(已经变成http，就没必要自动生成证书了)
- 2. http探活部分将https改为http探活
- 3. 修改service部分的端口

```
# container部分增加
        ports:
        - containerPort: 9090
          protocol: TCP

# 修改容器启动参数(取消自动生成证书)
        args:
          #- --auto-generate-certificates
# liveness探活检测
        livenessProbe:
          httpGet:
            scheme: HTTP
            path: /
            port: 9090

# service对外暴露端口(service之前是将容器的8443负载到service的443,我们这里直接映射到容器的9090,后面可以使用ingress)
spec:
  ports:
    - port: 9090
      targetPort: 9090

# 重新创建dashboard
$ kubectl create -f kubernetes-dashboard-http.yaml 

$ kubectl  get pods,svc  -n kube-system -o wide | grep dashboard
pod/kubernetes-dashboard-769598dcb7-b9mfc   1/1     Running   0          15m     20.0.97.2   172.16.21.23   <none>           <none>
service/kubernetes-dashboard      ClusterIP   10.253.172.6    <none>        9090/TCP                 15m     k8s-app=kubernetes-dashboard

# 访问pod容器的http端口
$ curl 20.0.97.2:9090 -I
HTTP/1.1 200 OK
Accept-Ranges: bytes
Cache-Control: no-store
Content-Length: 990
Content-Type: text/html; charset=utf-8
Last-Modified: Mon, 17 Dec 2018 09:04:43 GMT
Date: Fri, 06 Sep 2019 10:46:58 GMT

# 访问svc的http端口
$ curl 10.253.172.6:9090 -I
HTTP/1.1 200 OK
Accept-Ranges: bytes
Cache-Control: no-store
Content-Length: 990
Content-Type: text/html; charset=utf-8
Last-Modified: Mon, 17 Dec 2018 09:04:43 GMT
Date: Fri, 06 Sep 2019 10:47:04 GMT

```
`注意:将dashboard切换为http之后，就可以使用同一的traefik或者nginx进行代理了`

**问题列表**

- 访问dashboard之后发现各种没有权限

```
# 大概报错信息
configmaps is forbidden: User "system:serviceaccount:kube-system:kubernetes-dashboard" cannot list deploy.
....

# 给kubernetes-dashboard用户创建serviceaccount以及相关规则

$ cat clusterrole-binding.yml
# 给dashboard配置全部权限
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: kubernetes-dashboard
subjects:
  - kind: ServiceAccount
    name: kubernetes-dashboard
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io

$ kubectl create -f clusterrole-binding.yml 

# 至此用户可以直接访问http服务来使用dashboard观察集群详细信息
```

- 增加一个访问pod的权限(exec)

```
pods "myapp-6548f6cd69-h8dxq" is forbidden: User "system:serviceaccount:kubernetes-dashboard:kubernetes-dashboard" cannot create resource "pods/exec" in API group "" in the namespace "myapp"

```
