### 1、为什么要使用本地持久化存储？

这样做的好处很明显，由于这个 Volume 直接使用的是本地磁盘，尤其是 SSD 盘，它的读写性能相比于大多数远程存储来说，要好得多。
Kubernetes 在 v1.10 之后，就逐渐依靠 PV、PVC 体系实现了这个特性。这个特性的名字叫作：Local Persistent Volume。

### 2、Local PV 使用场景
它的适用范围非常固定，比如：高优先级的系统应用，需要在多个不同节点上存储数据，并且对 I/O 较为敏感。
典型的应用包括：分布式数据存储比如 MongoDB、Cassandra 等，分布式文件系统比如 GlusterFS、Ceph等，以及需要在本地磁盘上进行大量数据缓存的分布式应用。
### 2、如何保证 Pod 始终能被正确地调度到它所请求的 Local Persistent Volume 所在的节点上
通常我们先创建PV，然后创建PVC，这时候如果两者匹配那么系统会自动进行绑定，哪怕是动态PV创建，也是先调度POD到任意一个节点，然后根据PVC来进行创建PV然后进行绑定最后挂载到POD中，可是本地持久化存储有一个问题就是这种PV必须要先准备好，而且不一定集群所有节点都有这种PV，如果POD随意调度肯定不行，如何保证POD一定会被调度到有PV的节点上呢？这时候就需要在PV中声明节点亲和，且POD被调度的时候还要考虑卷的分布情况。

定义PV
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-pv
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local: # local类型
    path: /data/vol1  # 节点上的具体路径
  nodeAffinity: # 这里就设置了节点亲和
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node01 # 这里我们使用node01节点，该节点有/data/vol1路径

```
定义存储类
```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```
这里的```volumeBindingMode: WaitForFirstConsumer```很关键，意思就是延迟绑定，当有符合PVC要求的PV不立即绑定。因为POD使用PVC，而绑定之后，POD被调度到其他节点，显然其他节点很有可能没有那个PV所以POD就挂起了，另外就算该节点有合适的PV，而POD被设置成不能运行在该节点，这时候就没法了，延迟绑定的好处是，POD的调度要参考卷的分布。当开始调度POD的时候看看它要求的LPV在哪里，然后就调度到该节点，然后进行PVC的绑定，最后在挂载到POD中，这样就保证了POD所在的节点就一定是LPV所在的节点。所以让PVC延迟绑定，就是等到使用这个PVC的POD出现在调度器上之后（真正被调度之前），然后根据综合评估再来绑定这个PVC。
定义PVC
```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: local-claim
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: local-storage
```
定义POD
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tomcat-deploy
spec:
  replicas: 1
  selector:
    matchLabels:
      appname: myapp
  template:
    metadata:
      name: myapp
      labels:
        appname: myapp
    spec:
      containers:
      - name: myapp
        image: tomcat:8.5.38-jre8
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        volumeMounts:
          - name: tomcatedata
            mountPath : "/data"
      volumes:
        - name: tomcatedata
          persistentVolumeClaim:
            claimName: local-claim
```
这个POD被调度到node01上，因为我们的PV就在node01上，这时候你删除这个POD，然后在重建该POD，那么依然会被调度到node01上。

总结：本地卷也就是LPV不支持动态供给的方式，延迟绑定，就是为了综合考虑所有因素再进行POD调度。其根本原因是动态供给是先调度POD到节点，然后动态创建PV以及绑定PVC最后运行POD；而LPV是先创建与某一节点关联的PV，然后在调度的时候综合考虑各种因素而且要包括PV在哪个节点，然后再进行调度，到达该节点后在进行PVC的绑定。也就说动态供给不考虑节点，LPV必须考虑节点。所以这两种机制有冲突导致无法在动态供给策略下使用LPV。换句话说动态供给是PV跟着POD走，而LPV是POD跟着PV走。

---

参考文档
1、 [PV、PVC体系是不是多此一举？从本地持久化卷谈起](https://blog.csdn.net/qq_34556414/article/details/117755505)
2、[kubernetes的本地持久化存储--Local Persistent Volume解析](https://haojianxun.github.io/2019/01/10/kubernetes%E7%9A%84%E6%9C%AC%E5%9C%B0%E6%8C%81%E4%B9%85%E5%8C%96%E5%AD%98%E5%82%A8--Local%20Persistent%20Volume%E8%A7%A3%E6%9E%90/)
3、[PV、PVC、StorageClass讲解](https://www.cnblogs.com/rexcheny/p/10925464.html)
