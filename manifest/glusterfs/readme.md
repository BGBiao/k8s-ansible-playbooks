## GlusterFS在Kubernetes中的使用

`注意:` GlusterFS的endpoint，pv，pvc，pod均需要在一个`namespace`中间，因为pod在挂载的时候其实是需要去读取glusterfs的`service`的endpoint的.

