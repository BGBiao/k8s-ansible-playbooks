/*================================================================
*Copyright (C) 2019 BGBiao Ltd. All rights reserved.
*
*FileName:readme.md
*Author:Xuebiao Xu
*Date:2019年09月30日
*Description:
*
================================================================*/

## Kubernetes V1.15管理NVIDIA GPU容器

参考链接:

- [nvidia-k8s-device-plugin](https://github.com/NVIDIA/k8s-device-plugin/)
- [k8s-1.15调度GPU文档](https://v1-15.docs.kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/)
- [nvidia-docker](https://github.com/NVIDIA/nvidia-docker)

**0. GPU主机依赖**

- 1.下载`nvidia-driver`(官方提示要约等于361.93) 
- 2.安装`nvidia-docker2.x`(nvidia-docker1.x和2.x完全不同)
- 3.`docker`配置成`nvidia`的默认运行时
- 4.`kubernetes`版本大于1.10


**1. systemd服务配置文件**

`注意:`在企业级生产环境里通常都会使用`Centos`来运行服务，但由于`GPU`环境下需要安装GPU驱动、cuda、cudnn之类的依赖库，导致操作不方便，因此可能会使用`Ubuntu`来运行GPU相关服务，两种发型版的`systemd`服务启动配置默认不同，因此在自动化安装时需要适配到多个发行版

- 1.centos服务默认目录: /usr/lib/systemd/system/docker.service
- 2.ubuntu服务默认目录: /lib/systemd/system/docker.service

可在手动部署服务时，将服务配置文件都放置到`/etc/systemd/system/`目录

`提示:systemd加载配置文件的顺序和优先级可自行查阅`

**2. kubelet默认配置**

`注意:`k8s官方文档依然标明需要添加`--feature-gates="Accelerators=true"`参数，但其实在`k8s-v1.15`版本`Accelerators`已经废弃，改为使用`"DevicePlugins=true"`参数了。

另外，在`k8s`比较高版本后(至少v1.15)，`kubelet`相关参数建议在`--config`中进行指定，大概内容如下:

```
$ cat kubelet.config
...
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
port: 10250
featureGates:
  DevicePlugins: true
clusterDomain: cluster.local.
...
...
```

**3. docker默认配置**

增加`nvidia`的默认运行时
安装`nvidia-docker 2+`

```
$ cat /etc/docker/daemon.json
{
    "default-runtime": "nvidia",
    "runtimes": {
        "nvidia": {
            "path": "/usr/bin/nvidia-container-runtime",
            "runtimeArgs": []
        }
    }
}

# 重启docker和kubelet
$ systemctl daemon-reload && systemctl restart docker kubelet

```

**4. 给gpu节点打标签**

```
kubectl label nodes 172.16.21.0 gpu=nvidia-tesla-p100

```

**5. 给gpu节点部署nvidia-device-plugin插件**



```
# 给gpu节点创建nvidia-device-plugin插件
$ kubectl  apply -f nvidia-device-plugin-v1.9.yaml
daemonset.extensions/nvidia-device-plugin-daemonset created

$ kubectl  get pods -n kube-system  -o wide | grep nvidia-device
nvidia-device-plugin-daemonset-p9kff   1/1     Running   0          2s     20.0.52.3       172.16.21.0     <none>           <none>

# 查看device-plugin日志详情
$ kubectl  logs nvidia-device-plugin-daemonset-p9kff -n kube-system
2019/09/30 08:05:44 Loading NVML
2019/09/30 08:05:44 Fetching devices.
2019/09/30 08:05:44 Starting FS watcher.
2019/09/30 08:05:44 Starting OS watcher.
2019/09/30 08:05:44 Starting to serve on /var/lib/kubelet/device-plugins/nvidia.sock
2019/09/30 08:05:44 Registered device plugin with Kubelet

# 也可以在docker上测试该驱动
$ docker run --security-opt=no-new-privileges --cap-drop=ALL --network=none -it -v /var/lib/kubelet/device-plugins:/var/lib/kubelet/device-plugins nvidia/k8s-device-plugin:1.11

```

**6. 在k8s集群中调度gpu业务容器**

```
$ kubectl apply -f gpu-deploy-svc.yaml
...
...

# 查看gpu容器的deploy和svc(使用了service，并且使用nodePort类型)
$ kubectl  get pods,svc | grep gpu

pod/gpu-image-cluster-6565586479-x89bk   1/1     Running   0          9m1s
service/gpu-image-cluster   NodePort    10.253.172.218   <none>        8080:38080/TCP   8m21s


```

**7. 测试GPU容器业务**

`注意`:想要测试GPU容器，可以直接使用nvidia/cuda:8.0-runtime-ubuntu14.04镜像，容器运行后执行`nvidia-smi`可以显示GPU卡，即为生产。


```
$ kubectl  get nodes
NAME            STATUS   ROLES    AGE     VERSION
172.16.21.0     Ready    <none>   7h11m   v1.15.0
172.16.21.26    Ready    <none>   24d     v1.15.0
172.16.21.27    Ready    <none>   24d     v1.15.0
172.16.21.28    Ready    <none>   24d     v1.15.0

# 使用service 进行访问
$ curl -H 'Content-Type:application/json' -X POST -d '{"imgUrl": "https://img.bgbiao.cn/image/2019-08-27/f208b338-af03-4557-8d23-9cf308c38ba9-1566921008172.png"}' "http://10.253.172.218:8080/api/predict/class"
{"class_id":958,"class_name":"\u5c0f\u718a\u732b\u5927\u6bb5\u6587\u5b57\uff08\u53d7\u4e0d\u4e86\u7f51\u604b\uff09\u8868\u60c5\u5305","code":200,"message":"OK"}


# 由于是使用的nodePort类型的service，可以直接访问每个node节点的28080
$ curl -H 'Content-Type:application/json' -X POST -d '{"imgUrl": "https://img.bgbiao.cn/image/2019-08-27/f208b338-af03-4557-8d23-9cf308c38ba9-1566921008172.png"}' "http://172.16.21.0:38080/api/predict/class"
{"class_id":958,"class_name":"\u5c0f\u718a\u732b\u5927\u6bb5\u6587\u5b57\uff08\u53d7\u4e0d\u4e86\u7f51\u604b\uff09\u8868\u60c5\u5305","code":200,"message":"OK"}

$ curl -H 'Content-Type:application/json' -X POST -d '{"imgUrl": "https://img.bgbiao.cn/image/2019-08-27/f208b338-af03-4557-8d23-9cf308c38ba9-1566921008172.png"}' "http://172.16.21.26:38080/api/predict/class"
{"class_id":958,"class_name":"\u5c0f\u718a\u732b\u5927\u6bb5\u6587\u5b57\uff08\u53d7\u4e0d\u4e86\u7f51\u604b\uff09\u8868\u60c5\u5305","code":200,"message":"OK"}

$ curl -H 'Content-Type:application/json' -X POST -d '{"imgUrl": "https://img.bgbiao.cn/image/2019-08-27/f208b338-af03-4557-8d23-9cf308c38ba9-1566921008172.png"}' "http://172.16.21.27:38080/api/predict/class"
{"class_id":958,"class_name":"\u5c0f\u718a\u732b\u5927\u6bb5\u6587\u5b57\uff08\u53d7\u4e0d\u4e86\u7f51\u604b\uff09\u8868\u60c5\u5305","code":200,"message":"OK"}

$ curl -H 'Content-Type:application/json' -X POST -d '{"imgUrl": "https://img.bgbiao.cn/image/2019-08-27/f208b338-af03-4557-8d23-9cf308c38ba9-1566921008172.png"}' "http://172.16.21.28:38080/api/predict/class"
{"class_id":958,"class_name":"\u5c0f\u718a\u732b\u5927\u6bb5\u6587\u5b57\uff08\u53d7\u4e0d\u4e86\u7f51\u604b\uff09\u8868\u60c5\u5305","code":200,"message":"OK"}

```

