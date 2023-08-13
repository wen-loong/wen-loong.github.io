---
title: kubernetes-pv-pvc-usage
comments: false
aplayer: false
tags:
  - kubernetes
  - k8s
  - pv
  - pvc
  - cloud native
keywords:
  - kubernetes
  - k8s
  - pvc
  - pv
abbrlink: 483d5240
date: 2023-08-13 13:21:44
categories:
description: kubernetes pv pvc 使用
top_img:
cover:
sticky:

---
## 一、基本概念
### 1、PV
PV(PersistentVolumn)是集群中的一块存储，可以由管理员事先制备， 或者使用存储类（Storage Class）来动态制备。 持久卷是集群资源，就像节点也是集群资源一样。[详情](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)

### 2、PVC
PVC(PersistentVolumeClaim)表达的是用户对存储的请求。概念上与 Pod 类似。 Pod 会耗用节点资源，而 PVC 申领会耗用 PV 资源。

### SC
SC(StorageClass) 为管理员提供了描述存储"类"的方法。 不同的类型可能会映射到不同的服务质量等级或备份策略，或是由集群管理员制定的任意策略。 Kubernetes 本身并不清楚各种类代表的什么。这个类的概念在其他存储系统中有时被称为"配置文件"。
每个 StorageClass 都包含 provisioner、parameters 和 reclaimPolicy 字段， 这些字段会在 StorageClass 需要动态制备 PersistentVolume 时会使用到。

StorageClass 对象的命名很重要，用户使用这个命名来请求生成一个特定的类。 当创建 StorageClass 对象时，管理员设置 StorageClass 对象的命名和其他参数， 一旦创建了对象就不能再对其更新。[详情](https://kubernetes.io/docs/concepts/storage/storage-classes/)

### Provisioner
每个 StorageClass 都有一个制备器（Provisioner），用来决定使用哪个卷插件制备 PV。 该字段必须指定。[详情](https://kubernetes.io/docs/concepts/storage/storage-classes/#provisioner)


## 二、环境准备
### 1、NFS Server安装
+ 安装NFS server
```shell
 sudo apt install nfs-kernel-server -y
```
+ 关闭防火墙
```shell
sudo ufw disable
```
+ 建立NFS文件夹，并将其配置为共享文件夹
  + 建立文件夹并授权
```shell
mkdir -p /data/k8s/
sudo chown nobody:nogroup /data/k8s/
sudo chmod 777 /data/k8s/
```
  + 修改`/etc/exports`，添加 `/data/k8s *(insecure,rw,sync,no_root_squash)`
  + 刷新并重启`NFS`服务
```shell
sudo exportfs -ra
systemctl restart rpcbind
sudo systemctl restart rpcbind
sudo systemctl restart nfs
sudo systemctl restart nfs-kernel-server
```
> nfs允许挂载的目录及权限，在文件/etc/exports中进行定义，各字段含义如下：
/home/nfs/：与nfs服务客户端共享的目录，这个路径必须和你前面设置的文件的路径一致！
*：允许所有的网段访问，也可以使用具体的IP
rw：挂接此目录的客户端对该共享目录具有读写权限
async：资料同步写入内存和硬盘
no_root_squash：root用户具有对根目录的完全管理访问权限。
no_subtree_check：不检查父目录的权限。

### 2、安装NFS 客户端并挂载
```shell
   sudo apt-get install nfs-common
   sudo mkdir /data/k8s/
   sudo mkdir /data/k8s
   sudo mkdir -p /data/k8s
   sudo mount -t nfs -o nolock -o tcp 192.168.123.5:/data/k8s/ /data/k8s/
```

## 三、部署相关K8S资源
### 1、 SA(ServiceAccount)
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner

---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: default
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
```

### 2、Deploy Provisioner
```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: nfs-client-provisioner
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs-client-provisioner
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: quay.io/external_storage/nfs-client-provisioner:latest
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: fuseim.pri/ifs
            - name: NFS_SERVER
              value: 192.168.123.5
            - name: NFS_PATH
              value: /data/k8s/
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.123.5
            path: /data/k8s/
```

### 3、SC(StorageClass)
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: course-nfs-storage
  annotations: 
  - storageclass.kubernetes.io/is-default-class: true
provisioner: fuseim.pri/ifs
```

## 四、PV、PVC使用
此处以部署`MySQL`举例

### 1、PV(PersistentVolume)声明
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  capacity:
    storage: 500Mi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: course-nfs-storage
  nfs:
    path: /data/k8s/
    server: 192.168.123.5
```

### 2、PVC声明
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  labels:
    app: mysql-pvc
spec:
  storageClassName: course-nfs-storage
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

### 3、Deployment声明
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  labels:
    app: mysql-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  progressDeadlineSeconds: 300
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - image: mysql
          imagePullPolicy: Always
          name: mysql
          ports:
          - containerPort: 3306
          env:
          - name: POD_IP
            valueFrom:
              fieldRef:
                fieldPath: status.podIP
          - name: MYSQL_ROOT_PASSWORD
            valueFrom:
              secretKeyRef: 
                name: mysql-pass
                key: password
          - name: POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          volumeMounts:
          - name: mysql-config
            mountPath: /etc/mysql/my.cnf
            subPath: my.cnf
          - name: mysql-data
            mountPath: /var/lib/mysql
      volumes:
      - name: mysql-config
        configMap:
          name: mysql-config
      - name: mysql-data
        persistentVolumeClaim:
          claimName: mysql-pvc
```

### 4、ConfigMap
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  annotations:
  labels:
    app: mysql
  name: mysql-config
data:
  my.cnf: |
    [mysqld]
    pid-file        = /var/run/mysqld/mysqld.pid
    socket          = /var/run/mysqld/mysqld.sock
    datadir         = /var/lib/mysql
    lower_case_table_names = 1
    secure-file-priv= NULL
    
    # Custom config should go here
    !includedir /etc/mysql/conf.d/
```

### 5、Secrete
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-pass
type: Opaque
data:
  password: MTIzNDU2
```

### 6、Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
  namespace: dev
  labels:
    app: mysql-service
  annotations:
spec:
  type: NodePort
  selector:
    app: mysql
  ports: 
  - protocol: TCP
    port: 3306
    targetPort: 3306
```
### 7、验证
```shell
kubectl get pv,pvc -o wide
```
```text
NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS         REASON   AGE   VOLUMEMODE
persistentvolume/pvc-7e1f449e-92d2-4d91-b6c6-74fb9e9838d0   500Mi      RWO            Delete           Bound    dev/mysql-pvc       course-nfs-storage            35d   Filesystem
persistentvolume/pvc-d2af51c8-771f-42c7-9672-34887048737a   500Mi      RWO            Delete           Bound    default/mysql-pvc   course-nfs-storage            35d   Filesystem

NAME                              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS         AGE   VOLUMEMODE
persistentvolumeclaim/mysql-pvc   Bound    pvc-d2af51c8-771f-42c7-9672-34887048737a   500Mi      RWO            course-nfs-storage   35d   Filesystem
```
> 可以看到PVC 的状态为`Bound`说明已经成功

再到nfs server查看文件是否落盘
```shell
ubuntu@nfs:~$ tree /data/k8s/
/data/k8s/
├── archived-default-mysql-pvc-pvc-c432d8e3-a5e3-40d9-845e-ade26f9ceb23
├── default-mysql-pvc-pvc-d2af51c8-771f-42c7-9672-34887048737a
└── dev-mysql-pvc-pvc-7e1f449e-92d2-4d91-b6c6-74fb9e9838d0
    ├── #ib_16384_0.dblwr
    ├── #ib_16384_1.dblwr
    ├── #innodb_redo  [error opening dir]
    ├── #innodb_temp  [error opening dir]
    ├── auto.cnf
    ├── binlog.000004
    ├── binlog.000005
    ├── binlog.index
    ├── ca-key.pem
    ├── ca.pem
    ├── client-cert.pem
    ├── client-key.pem
    ├── ib_buffer_pool
    ├── ibdata1
    ├── ibtmp1
    ├── mysql  [error opening dir]
    ├── mysql.ibd
    ├── mysql.sock -> /var/run/mysqld/mysqld.sock
    ├── mysql_upgrade_info
    ├── nacos  [error opening dir]
    ├── performance_schema  [error opening dir]
    ├── private_key.pem
    ├── public_key.pem
    ├── server-cert.pem
    ├── server-key.pem
    ├── sys  [error opening dir]
    ├── undo_001
    └── undo_002

9 directories, 22 files
```