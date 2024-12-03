### 创建k8s-volume使用的文件存储

> 注意：ceph-node01 操作

创建存储池

一个 ceph 文件系统需要至少两个 RADOS 存储池，一个用于存储数据，一个用于存储元数据

```bash
1.创建cephfs_pool存储池
root@ceph-node01(192.168.199.44)/etc/ceph> ceph osd pool create cephfs_data 128
pool 'cephfs_data' created

2.创建cephfs_metadata存储池
root@ceph-node01(192.168.199.44)/etc/ceph> ceph osd pool create cephfs_metadata 64
pool 'cephfs_metadata' created

3.查看存储池
root@ceph-node01(192.168.199.44)/etc/ceph> ceph osd pool ls
cephfs_pool
cephfs_metadata

4.创建cephfs
root@ceph-node01(192.168.199.44)/etc/ceph> ceph fs new cephfs cephfs_metadata cephfs_data
new fs with metadata pool 2 and data pool 1

5.设置最大活动数
root@ceph-node01(192.168.199.44)/etc/ceph> ceph fs set cephfs max_mds 2

6.查看cephfs
root@ceph-node01(192.168.199.44)/etc/ceph> ceph fs ls
name: cephfs, metadata pool: cephfs_metadata, data pools: [cephfs_pool ]
```

### 创建k8s访问cephfs的认证用户

> 注意：ceph-node01 操作

K8S想要访问Ceph中的RBD块设备，必须通过一个认证用户才可以访问，如果没有认证用户则无法访问Ceph集群中的块设备。

命令格式：`ceph auth get-or-create {用户名称} mon '{访问mon的方式}' osd '{访问osd的方式}'`

```bash
ceph auth get-or-create client.cephfs mon 'allow r' mds 'allow rw' osd 'allow rw pool=cephfs_data, allow rw pool=cephfs_metadata'

ceph auth get  client.cephfs
[client.cephfs]
        key = AQDW9k1lDBtfBRAAP7aBGqZMXSXlNeWbmQNAoQ==
        caps mds = "allow rw"
        caps mon = "allow r"
        caps osd = "allow rw pool=cephfs_data, allow rw pool=cephfs_metadata"
exported keyring for client.cephfs
```

### 本地测试挂载并创建目录

> 注意：ceph-node01 操作

```bash
1.挂载cephfs
root@ceph-node01(192.168.199.44)/root> mount.ceph ceph-node01:6789:/ /mnt/cephfs/ -o name=cephfs,secret=AQDW9k1lDBtfBRAAP7aBGqZMXSXlNeWbmQNAoQ==
2.创建首页文件
root@localhost(192.168.199.44)/root>echo "$(date) hello cephfs." > /mnt/cephfs/index.html
```

### 将认证的用户key存放在k8s Secret资源中

1. 将key通过 base64 进行加密

```bash
root@ceph-node01(192.168.199.44)~>ceph auth get-key client.cephfs | base64
QVFEQjcwMWxLU2U3QWhBQXF3My94YS9aU2hPd1FkZWlOQ2EwMXc9PQ==
```

### 在k8s集群所有节点安装ceph-common

ceph-common是 ceph 命令包，需要在每个节点安装，否则将无法执行命令对ceph 进行操作。

```bash
root@k8s-master(192.168.199.41)~>yum install -y ceph-common
root@k8s-node01(192.168.199.42)~>yum install -y ceph-common
root@k8s-node02(192.168.199.43)~>yum install -y ceph-common
```

### 部署 ceph-csi

> ceph-csi 链接：https://github.com/ceph/ceph-csi/tree/devel/deploy

StorageClass资源可以通过客户端根据用户的需求自动创建出PV以及PVC资源。

StorageClass使用Ceph作为底层存储，为用户自动创建出PV以及PVC资源，使用的客户端工具是csi，首先需要在K8S集群中部署csi客户端工具，由csi客户端中驱动去连接Ceph集群。

**注意：在部署 ceph-csi 时，需要特别注意版本，ceph-csi所使用的版本依据ceph和k8s版本而定。**

**本次版本如下：**

```bash
ceph: nautilus
k8s: v1.19.7
ceph-csi: 3.4.0
```

ceph-csi 下载链接： https://github.com/ceph/ceph-csi/archive/refs/tags/v3.4.0.tar.gz

```bash
root@k8s-master(192.168.199.41)~>tar xf ceph-csi-3.4.0.tar.gz
root@k8s-master(192.168.199.41)/root> cd ceph-csi-3.4.0/deploy/cephfs/kubernetes/
```

1. 修改 `csi-config-map.yaml`

```yaml
root@k8s-master(192.168.199.41)/root/ceph-csi-3.4.0/deploy/cephfs/kubernetes> vim csi-config-map.yaml

---
apiVersion: v1
kind: ConfigMap
data:
  config.json: |-
    [
      {
        "clusterID": "212d53e8-9766-44a4-9cfd-d9ca6ae44882",	//通过 ceph mon dump 查看fsid
        "monitors": [
          "192.168.199.44:6789",	//所有ceph mon节点
          "192.168.199.45:6789",
          "192.168.199.46:6789"
        ]
      }
    ]
metadata:
  name: ceph-csi-config
```

1. k8s-master节点参与Pod调度

这里只对三个节点的集群，如果k8s-node节点超过三个不用执行该步骤。因为安装 `csi-cephfs`需要在三个node节点上调度。

```bash
root@k8s-master(192.168.199.41)/root> kubectl taint nodes k8s-master node-role.kubernetes.io/master:NoSchedule-
```

1. 部署 `cephfs csi`

```bash
root@k8s-master(192.168.199.41)/root/ceph-csi-3.4.0/deploy/cephfs/kubernetes> kubectl apply -f csi-config-map.yaml
root@k8s-master(192.168.199.41)/root/ceph-csi-3.4.0/deploy/cephfs/kubernetes> kubectl apply -f csi-provisioner-rbac.yaml
root@k8s-master(192.168.199.41)/root/ceph-csi-3.4.0/deploy/cephfs/kubernetes> kubectl apply -f csi-nodeplugin-rbac.yaml
root@k8s-master(192.168.199.41)/root/ceph-csi-3.4.0/deploy/cephfs/kubernetes> kubectl apply -f csi-cephfsplugin-provisioner.yaml
root@k8s-master(192.168.199.41)/root/ceph-csi-3.4.0/deploy/cephfs/kubernetes> kubectl apply -f csi-cephfsplugin.yaml
```

查看启动的Pods是否正常

```bash
root@k8s-master(192.168.199.41)/root> kubectl get po -o wide
NAME                                            READY   STATUS    RESTARTS   AGE     IP               NODE         NOMINATED NODE   READINESS GATES
csi-cephfsplugin-6p5vt                          3/3     Running   0          3m53s   192.168.199.41   k8s-master   <none>           <none>
csi-cephfsplugin-ds24t                          3/3     Running   0          3m53s   192.168.199.43   k8s-node02   <none>           <none>
csi-cephfsplugin-provisioner-55fb69589f-7kjdq   6/6     Running   0          4m6s    10.244.58.196    k8s-node02   <none>           <none>
csi-cephfsplugin-provisioner-55fb69589f-v5blc   6/6     Running   0          4m6s    10.244.85.203    k8s-node01   <none>           <none>
csi-cephfsplugin-provisioner-55fb69589f-wjw5j   6/6     Running   0          4m6s    10.244.235.196   k8s-master   <none>           <none>
csi-cephfsplugin-q9qmc                          3/3     Running   0          3m53s   192.168.199.42   k8s-node01   <none>           <none>
```

4.创建密钥

```bash
root@k8s-master(192.168.199.41)/root> cd ceph-csi-3.4.0/examples/cephfs/
root@k8s-master(192.168.199.41)/root/ceph-csi-3.4.0/examples/cephfs> vim secret.yaml

---
apiVersion: v1
kind: Secret
metadata:
  name: csi-cephfs-secret
  namespace: default
stringData:
  # Required for statically provisioned volumes
  userID: cephfs	//上面创建 cephfs 的认证用户
  userKey: AQDW9k1lDBtfBRAAP7aBGqZMXSXlNeWbmQNAoQ==	//通过 ceph auth get-key client.cephfs 查看，无需通过 base64 转换

  # Required for dynamically provisioned volumes
  adminID: admin	// ceph管理员认证用户
  adminKey: AQDG0U1lita+ABAAVoqPu5pxe/7dMh2UdXsRKA== //通过 ceph auth get-key client.admin 查看，无需通过 base64 转换
  
  root@k8s-master(192.168.199.41)/root/ceph-csi-3.4.0/examples/cephfs> kubectl apply -f secret.yaml
```

### 创建StorageClass

```yaml
root@k8s-master(192.168.199.41)/root/ceph-csi-3.4.0/examples/cephfs> vim storageclass.yaml

---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: csi-cephfs-sc
provisioner: cephfs.csi.ceph.com
parameters:
  # (required) String representing a Ceph cluster to provision storage from.
  # Should be unique across all Ceph clusters in use for provisioning,
  # cannot be greater than 36 bytes in length, and should remain immutable for
  # the lifetime of the StorageClass in use.
  # Ensure to create an entry in the configmap named ceph-csi-config, based on
  # csi-config-map-sample.yaml, to accompany the string chosen to
  # represent the Ceph cluster in clusterID below
  clusterID: 212d53e8-9766-44a4-9cfd-d9ca6ae44882	//通过 ceph mon dump 查看fsid

  # (required) CephFS filesystem name into which the volume shall be created
  # eg: fsName: myfs
  fsName: cephfs	// 通过 ceph fs ls 查看

  # (optional) Ceph pool into which volume data shall be stored
  # pool: <cephfs-data-pool>

  # (optional) Comma separated string of Ceph-fuse mount options.
  # For eg:
  # fuseMountOptions: debug

  # (optional) Comma separated string of Cephfs kernel mount options.
  # Check man mount.ceph for mount options. For eg:
  # kernelMountOptions: readdir_max_bytes=1048576,norbytes

  # The secrets have to contain user and/or Ceph admin credentials.
  csi.storage.k8s.io/provisioner-secret-name: csi-cephfs-secret
  csi.storage.k8s.io/provisioner-secret-namespace: default
  csi.storage.k8s.io/controller-expand-secret-name: csi-cephfs-secret
  csi.storage.k8s.io/controller-expand-secret-namespace: default
  csi.storage.k8s.io/node-stage-secret-name: csi-cephfs-secret
  csi.storage.k8s.io/node-stage-secret-namespace: default

  # (optional) The driver can use either ceph-fuse (fuse) or
  # ceph kernelclient (kernel).
  # If omitted, default volume mounter will be used - this is
  # determined by probing for ceph-fuse and mount.ceph
  # mounter: kernel

  # (optional) Prefix to use for naming subvolumes.
  # If omitted, defaults to "csi-vol-".
  # volumeNamePrefix: "foo-bar-"

reclaimPolicy: Delete
allowVolumeExpansion: true
mountOptions:
  - discard	//debug修改为discard
  
root@k8s-master(192.168.199.41)/root/ceph-csi-3.4.0/examples/cephfs> kubectl apply -f storageclass.yaml
root@k8s-master(192.168.199.41)/root/ceph-csi-3.4.0/examples/cephfs> kubectl get sc
NAME            PROVISIONER           RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
csi-cephfs-sc   cephfs.csi.ceph.com   Delete          Immediate           true                   2d15h
```

### 创建PVC

```yaml
root@k8s-master(192.168.199.41)/root/ceph-csi-3.4.0/examples/cephfs> vim pvc.yaml

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: csi-cephfs-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: csi-cephfs-sc
  
root@k8s-master(192.168.199.41)/root/ceph-csi-3.4.0/examples/cephfs> kubectl apply -f pvc.yaml
root@k8s-master(192.168.199.41)/root/ceph-csi-3.4.0/examples/cephfs> kubectl get pvc,pv
NAME                                   STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS    AGE
persistentvolumeclaim/csi-cephfs-pvc   Bound    pvc-c7023d0d-f100-49ea-8010-b8752a5b46ee   1Gi        RWX            csi-cephfs-sc   9s

NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                    STORAGECLASS    REASON   AGE
persistentvolume/pvc-c7023d0d-f100-49ea-8010-b8752a5b46ee   1Gi        RWX            Delete           Bound    default/csi-cephfs-pvc   csi-cephfs-sc            8s
```

### 创建Pod挂载pvc

```bash
root@k8s-master(192.168.199.41)/root/ceph-csi-3.4.0/examples/cephfs> cat pod.yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: csi-cephfs-demo-pod
spec:
  containers:
    - name: web-server
      image: docker.io/library/nginx:latest
      volumeMounts:
        - name: mypvc
          mountPath: /usr/share/nginx/html	//这里修改为nginx访问根目录
  volumes:
    - name: mypvc
      persistentVolumeClaim:
        claimName: csi-cephfs-pvc
        readOnly: false


root@k8s-master(192.168.199.41)/root/ceph-csi-3.4.0/examples/cephfs> kubectl apply -f pod.yaml
```

### 测试验证

将 `cephfs` 挂载到本地写入 `index.html` ，然后通过 Pod 访问

1. 挂载到本地

```bash
root@ceph-node01(192.168.199.44)/root> mount.ceph ceph-node01:6789:/ /mnt/cephfs/ -o name=cephfs,secret=AQDW9k1lDBtfBRAAP7aBGqZMXSXlNeWbmQNAoQ==
```

1. 写入文件

```bash
root@ceph-node01(192.168.199.44)/root> cd /mnt/cephfs/
1.pvc生成的目录是在这个 volumes里
root@ceph-node01(192.168.199.44)/mnt/cephfs> ls
html/  index.html  volumes/

2.写入文件
root@ceph-node01(192.168.199.44)~>echo "$(date) hello cephfs." > /mnt/cephfs/volumes/csi/csi-vol-ff2f1bae-81d1-11ee-86dc-6aa525639d6b/b8209261-dbc2-4277-b6b8-ac98e6786215/index.html

3.Pod地址访问
root@k8s-master(192.168.199.41)/root/ceph-csi-3.4.0/examples/cephfs> kubectl get po -o wide
cNAME                                            READY   STATUS    RESTARTS   AGE     IP               NODE         NOMINATED NODE   READINESS GATES
csi-cephfsplugin-provisioner-55fb69589f-gb77z   6/6     Running   0          10m     10.244.58.197    k8s-node02   <none>           <none>
csi-cephfsplugin-provisioner-55fb69589f-m768l   6/6     Running   0          10m     10.244.235.197   k8s-master   <none>           <none>
csi-cephfsplugin-provisioner-55fb69589f-v864s   6/6     Running   0          10m     10.244.85.205    k8s-node01   <none>           <none>
csi-cephfsplugin-rjxl4                          3/3     Running   0          10m     192.168.199.43   k8s-node02   <none>           <none>
csi-cephfsplugin-mlb2j                          3/3     Running   0          10m     192.168.199.41   k8s-master   <none>           <none>
csi-cephfsplugin-n967b                          3/3     Running   0          10m     192.168.199.42   k8s-node01   <none>           <none>
csi-cephfs-demo-pod                             1/1     Running   0          2m53s   10.244.85.206    k8s-node01   <none>           <none>
root@k8s-master(192.168.199.41)/root/ceph-csi-3.4.0/examples/cephfs> curl 10.244.85.206
Mon Nov 13 11:18:21 CST 2023 hello cephfs.
```