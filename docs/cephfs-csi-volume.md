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
        key = AQBs2U1lK+uHKBAAGTEH5fSx5mjBPoH87CxHIQ==
        caps mds = "allow rw"
        caps mon = "allow r"
        caps osd = "allow rw pool=cephfs_data, allow rw pool=cephfs_metadata"
exported keyring for client.cephfs
```

### 本地测试挂载并创建目录

> 注意：ceph-node01 操作

```bash
1.挂载cephfs
root@ceph-node01(192.168.199.44)/root> mount.ceph ceph-node01:6789:/ /mnt/cephfs/ -o name=cephfs,secret=AQBs2U1lK+uHKBAAGTEH5fSx5mjBPoH87CxHIQ==

2.创建目录
root@ceph-node01(192.168.199.44)/root> mkdir -p /mnt/cephfs/html

3.创建首页文件
root@localhost(192.168.199.44)/root>echo "$(date) hello cephfs." > /mnt/cephfs/html/index.html
```

### 将认证的用户key存放在k8s Secret资源中

1. 将key通过 base64 进行加密

```bash
root@ceph-node01(192.168.199.44)~>ceph auth get-key client.cephfs | base64
QVFCczJVMWxLK3VIS0JBQUdURUg1ZlN4NW1qQlBvSDg3Q3hISVE9PQ==
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
  key: QVFCczJVMWxLK3VIS0JBQUdURUg1ZlN4NW1qQlBvSDg3Q3hISVE9PQ==
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

### 创建pod资源挂载cephfs文件存储进行数据持久化



1. 编写编排文件

```bash
root@k8s-master(192.168.199.41)~>cd manifests/

root@k8s-master(192.168.199.41)/root/manifests> cat pod-ngx.yaml
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
    cephfs:
      monitors:
      - 192.168.199.44:6789
      - 192.168.199.45:6789
      - 192.168.199.46:6789
      path: /html
      user: cephfs
      secretRef:
        name: cephfs-secret
```

1. 创建pod资源

```bash
root@k8s-master(192.168.199.41)/root/manifests> kubectl apply  -f pod-ngx.yaml
pod/ngx created

查看创建的pod
root@k8s-master(192.168.199.41)/root/manifests> kubectl get po -o wide
NAME   READY   STATUS    RESTARTS   AGE   IP              NODE         NOMINATED NODE   READINESS GATES
ngx    1/1     Running   0          74s   10.244.85.196   k8s-node01   <none>           <none>




查看创建过程
root@k8s-master(192.168.199.41)~/manifests>kubectl describe po ngx
...
Volumes:
  html:
    Type:        CephFS (a CephFS mount on the host that shares a pod's lifetime)
    Monitors:    [192.168.199.44:6789 192.168.199.45:6789 192.168.199.46:6789]
    Path:        /html
    User:        cephfs
    SecretFile:
    SecretRef:   &LocalObjectReference{Name:cephfs-secret,}
    ReadOnly:    false
...
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  7s    default-scheduler  Successfully assigned default/ngx to k8s-node01
  Normal  Pulled     6s    kubelet            Container image "nginx:alpine" already present on machine
  Normal  Created    6s    kubelet            Created container ngx
  Normal  Started    5s    kubelet            Started container ngx
```

### 查看pod资源挂载的cephfs信息

**1. 直接访问Pod**

```bash
root@k8s-master(192.168.199.41)/root/manifests> kubectl get po -o wide
NAME   READY   STATUS    RESTARTS   AGE    IP              NODE         NOMINATED NODE   READINESS GATES
ngx    1/1     Running   0          108s   10.244.85.197   k8s-node01   <none>           <none>
root@k8s-master(192.168.199.41)/root/manifests> curl 10.244.85.197
Fri Nov 10 16:07:59 CST 2023 hello cephfs.
```

**2. 在宿主机上查看挂载的cephfs信息**

为什么会在Pod中看到挂载的cephfs，其实是宿主机将目录挂载到了容器的某个路径中

首先查看Pod运行在了哪个Node节点上，然后查看cephfs的挂载信息。

```bash
root@k8s-master(192.168.199.41)/root/manifests> kubectl get po -o wide
NAME   READY   STATUS    RESTARTS   AGE     IP              NODE         NOMINATED NODE   READINESS GATES
ngx    1/1     Running   0          3m30s   10.244.85.197   k8s-node01   <none>           <none>

root@k8s-node01(192.168.199.42)~>df -Th | egrep html
192.168.199.44:6789,192.168.199.45:6789,192.168.199.46:6789:/html ceph      8.5G     0  8.5G   0% /var/lib/kubelet/pods/a6df1c7c-9d13-4b8e-a8ed-18e1af363f58/volumes/kubernetes.io~cephfs/html
```