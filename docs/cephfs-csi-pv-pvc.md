### 创建k8s-volume使用的文件存储

> 注意：ceph-node01 操作

1. 创建MDS

```bash
root@localhost(192.168.199.44)/etc/ceph>ceph-deploy mds create ceph-node0{1..3}
```

1. 创建存储池

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

5.查看cephfs
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
        key = AQDB701lKSe7AhAAqw3/xa/ZShOwQdeiNCa01w==
        caps mds = "allow rw"
        caps mon = "allow r"
        caps osd = "allow rw pool=cephfs_data, allow rw pool=cephfs_metadata"
exported keyring for client.cephfs
```

### 本地测试挂载并创建目录

> 注意：ceph-node01 操作

```bash
1.挂载cephfs
root@ceph-node01(192.168.199.44)/root> mount.ceph ceph-node01:6789:/ /mnt/cephfs/ -o name=cephfs,secret=AQDB701lKSe7AhAAqw3/xa/ZShOwQdeiNCa01w==

2.创建目录
root@ceph-node01(192.168.199.44)/root> mkdir -p /mnt/cephfs/html

3.创建首页文件
root@localhost(192.168.199.44)/root>echo "$(date) hello cephfs." > /mnt/cephfs/html/index.html
```

### 将认证的用户key存放在k8s Secret资源中

1. 将key通过 base64 进行加密

```bash
root@ceph-node01(192.168.199.44)~>ceph auth get-key client.cephfs | base64
QVFEQjcwMWxLU2U3QWhBQXF3My94YS9aU2hPd1FkZWlOQ2EwMXc9PQ==
```

1. 将加密后的key 存在 secret 中

```yaml
root@k8s-master(192.168.199.41)~>mkdir -pv manifests
mkdir: created directory ‘manifests’
root@k8s-master(192.168.199.41)~>cd manifests/
root@k8s-master(192.168.199.41)~/manifests>vim cephfs-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: cephfs-secret
data:
  key: QVFEQjcwMWxLU2U3QWhBQXF3My94YS9aU2hPd1FkZWlOQ2EwMXc9PQ==
```

1. 创建 secret资源

RBD的Secret要与Pod在同一Namespace下，如果不同的Namespace的Pod都需要使用RBD进行存储，则需要在每个Namespace下都进行创建。

```bash
root@k8s-master(192.168.199.41)/root/manifests> kubectl apply -f cephfs-secret.yaml
secret/cephfs-secret created
root@k8s-master(192.168.199.41)/root/manifests> kubectl get secret
NAME                  TYPE                                  DATA   AGE
cephfs-secret         Opaque                                1      4s
default-token-hp7gb   kubernetes.io/service-account-token   3      48m
```

### 在k8s集群所有节点安装ceph-common

ceph-common是 ceph 命令包，需要在每个节点安装，否则将无法执行命令对ceph 进行操作。

```bash
root@k8s-master(192.168.199.41)~>yum install -y ceph-common
root@k8s-node01(192.168.199.42)~>yum install -y ceph-common
root@k8s-node02(192.168.199.43)~>yum install -y ceph-common
```

### 创建PV及PVC资源使用cephfs作为文件存储

在K8S集群中创建PV及PVC存储资源，主要是对PV进行了一些配置，存储底层采用Ceph集群的cephfs

**1. 编写资源编排文件**

```yaml
root@k8s-master(192.168.199.41)/root/manifests/cephfs-pv> cat cephfs-pv-pvc.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: cephfs-pv
  namespace: default
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteMany
  cephfs:
    monitors:
    - 192.168.199.44:6789
    - 192.168.199.45:6789
    - 192.168.199.46:6789
    path: /
    user: cephfs
    secretRef:
      name: cephfs-secret
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: cephfs
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cephfs-pvc
  namespace: default
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  volumeName: cephfs-pv
  storageClassName: cephfs
```

**2. 在集群中创建pv和pvc**

```bash
1.创建资源
root@k8s-master(192.168.199.41)/root/manifests/cephfs-pv> kubectl apply -f cephfs-pv-pvc.yaml
persistentvolume/cephfs-pv created
persistentvolumeclaim/cephfs-pvc created

2.查看创建的资源
root@k8s-master(192.168.199.41)/root/manifests/cephfs-pv> kubectl get pv,pvc
NAME                         CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                STORAGECLASS   REASON   AGE
persistentvolume/cephfs-pv   1Gi        RWX            Recycle          Bound    default/cephfs-pvc   cephfs                  12s

NAME                               STATUS   VOLUME      CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/cephfs-pvc   Bound    cephfs-pv   1Gi        RWX            cephfs         12s
```

### 创建Pod资源挂载PV存储卷并写入数据

**1. 编写Pod资源编排文件**

```yaml
root@k8s-master(192.168.199.41)/root/manifests/cephfs-pv> cat cephfs-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: ngx
  name: ngx
spec:
  containers:
  - image: nginx:alpine
    name: ngx
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
  restartPolicy: Always
  volumes:
  - name: html
    persistentVolumeClaim:
      claimName: cephfs-pvc
      readOnly: false
```

**2. 在集群中创建Pod资源**

```bash
root@k8s-master(192.168.199.41)/root/manifests/cephfs-pv> kubectl apply -f cephfs-pod.yaml
pod/ngx created
```

**3. 直接访问页面**

```bash
root@k8s-master(192.168.199.41)/root/manifests/cephfs-pv> kubectl get po -o wide
NAME   READY   STATUS    RESTARTS   AGE   IP              NODE         NOMINATED NODE   READINESS GATES
ngx    1/1     Running   0          58s   10.244.85.199   k8s-node01   <none>           <none>
root@k8s-master(192.168.199.41)/root/manifests/cephfs-pv> curl 10.244.85.199
Fri Nov 10 17:12:13 CST 2023 hello cephfs.
```