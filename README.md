# ceph-csi-linux-riscv

版本说明

ceph version 17.2.6 (d7ff0d10654d2280e08f1ab989c7cdf3064446a5) quincy (stable)

K8s version v1.26.5

quay.io/cephcsi/cephcsi                                       v3.11.0        72f63832a2da  4 days ago      1.07 GB
registry.k8s.io/sig-storage/csi-snapshotter                   v7.0.0         541b5d3f1bb6  4 days ago      269 MB
quay.io/ceph/ceph                                             v18            16b93f4b88e0  4 days ago      963 MB
registry.k8s.io/sig-storage/csi-resizer                       v1.10.0        fc4018e64f22  4 days ago      166 MB
registry.k8s.io/sig-storage/csi-provisioner                   v4.0.0         fbd6c208d56c  4 days ago      169 MB
registry.k8s.io/sig-storage/csi-node-driver-registrar         v2.10.0        b719a1335f82  4 days ago      124 MB

# 流程

1. 获取指定的源码，然后进行编译
2. 编译后，进行docker tag

# 注意事项

1. Ceph-csi 镜像需要用到ceph 镜像，这个镜像没有riscv64 架构的，本人是将x64 的ceph:v18 镜像，进行读取dockerfile 操作，然后更改成ubuntu:22.04 riscv64 架构
2. 在部署ceph csi 时候好多忽略了一点就是执行 kubectl apply -f ceph-conf.yaml 这个文件
3. 在执行 使用cephfs 时候需要执行 ceph fs subvolumegroup create <cephfs_name> csi

# 目录说明

- Bin 存放 编译后的二进制
- dockerfile 是本次编译的Dockerfile 文件
- docs 下放了一些 ceph csi 的操作文档
