## kubeconfig

`注意:需要有个集群管理员的权限`

`只有和master节点一样的权限才能够将相关证书给第三方的接口调用，比如spinnaker`

```
# 生成一个客户端可以使用的Kubeconfig文件
kubectl config set-credentials default-auth --client-key=/data/kubernetes/ssl/kubelet-client-current.pem --client-certificate=/data/kubernetes/ssl/kubelet-client-current.pem --kubeconfig=kubelet.kubeconfig --embed-certs=true


# 超级管理员的权限(和master节点一样的权限)
kubectl apply -f role-binding.yaml

```

