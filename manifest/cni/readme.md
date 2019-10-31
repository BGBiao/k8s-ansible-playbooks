## kubernetes CNI网络插件集锦

- [Flannel]()
- [Calico]()
- [Terway](./terway)

`注意:` 

阿里云的`terway`网络插件支持两种模式:

- VPC模式: 该模式下需要使用[podnetwork](terway/podnetwork.yaml)来创建网络插件，vpc模式下课分为一般容器和带eni的容器，一般容器仅为使用了vpc网络，但是无法获取vpc子网的通信能力;通过`limits: aliyun/eni: 1`资源限制给容器pod分配弹性eni，使得其拥有vpc子网通信的能力
- ENI多ip模式:该模式下需要使用[mutil-ip](terway/terway-mutiip.yaml)来创建网络插件，可以在ECS主机上绑定虚拟网卡，让节点上的pod使用到vpc子网(虚拟网卡受ECS规格限制,pod的ip个数受虚拟网卡的限制)
