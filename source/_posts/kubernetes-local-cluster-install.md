---
title: kubernetes-install
comments: false
aplayer: false
tags:
  - kubernetes
  - k8s
  - multipass
  - cloud native
keywords:
  - kubernetes
  - k8s
description: kubernetes 本地集群搭建
abbrlink: d6ebbcb
date: 2023-07-06 16:38:48
categories:
top_img:
cover:
sticky:
---

## 一、multipass简介及安装
以下来源于multipass 官网
>Multipass is a lightweight VM manager for Linux, Windows and macOS. It's designed for developers who want a fresh Ubuntu environment with a single command. It uses KVM on Linux, Hyper-V on Windows and QEMU on macOS to run the VM with minimal overhead. It can also use VirtualBox on Windows and macOS. Multipass will fetch images for you and keep them up to date.

它是一个是一个轻量级的VM管理器，适用于希望使用单个命令获得全新 Ubuntu 环境的开发人员。

安装和配置详见[官网](https://multipass.run/)和[GitHub](https://github.com/canonical/multipass)，此处不做过多赘述。

#### 创建虚拟机
本次共创建三台虚拟机，一台master节点，两台worker节点
```shell
multipass launch --name master --cpus 2 --memory 8192M --disk 30G
multipass launch --name node1 --cpus 1 --memory 4096M --disk 30G
multipass launch --name node2 --cpus 1 --memory 4096M --disk 30G
```
进入虚拟机使用如下命令
```shell
multipass shell [VM-name]
```


## kubernetes本地集群搭建

此处使用`kubeadm`和`docker`搭建
##### 环境准备
```shell
# 禁用交换分区(在旧版的 k8s 中 kubelet 都要求关闭 swapoff ，但最新版的 kubelet 其实已经支持 swap ，因此这一步其实可以不做。)
swapoff -a
# 永久禁用，打开/etc/fstab注释掉swap那一行。  
sudo vim /etc/fstab
# 修改内核参数(首先确认你的系统已经加载了 br_netfilter 模块，默认是没有该模块的，需要你先安装 bridge-utils)
apt-get install -y bridge-utils
modprobe br_netfilter
lsmod | grep br_netfilter
# 如果报错找不到包，需要先更新 apt-get update -y
swapoff -a

```
##### `docker`安装及启动
```shell 
apt-get update
apt-get install docker.io -y
# 设置开机启动并启动docker  
sudo systemctl start docker
sudo systemctl enable docker
```

##### `kubelet` `kubeadm``kubectl`的安装

```shell
# 安装基础环境
apt-get install -y ca-certificates curl software-properties-common apt-transport-https curl
curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -
# 执行配置k8s阿里云源  
vim /etc/apt/sources.list.d/kubernetes.list
#加入以下内容
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
# 执行更新
apt-get update -y
# 安装kubeadm、kubectl、kubelet  
apt-get install -y kubelet=1.23.1-00 kubeadm=1.23.1-00 kubectl=1.23.1-00
# 阻止自动更新(apt upgrade时忽略)。所以更新的时候先unhold，更新完再hold。
apt-mark hold kubelet kubeadm kubectl
```

##### 初始化`master`节点
由于网络问题无法访问k8s的repository此处使用aliyun的，可以使用`kubeadm config images pull` 来测试与 `gcr.io` 的连接
```shell
kubeadm init --pod-network-cidr=10.244.0.0/16 --image-repository registry.aliyuncs.com/google_containers
```
也可以将`image`直接导入或者使用aliyun下载镜像之后重新打`tag`
使用`kubeadm config images list`查依赖的images有哪些：
```shell
root@master:~# kubeadm config images list
I0706 15:09:33.039269    4295 version.go:255] remote version is much newer: v1.27.3; falling back to: stable-1.23
k8s.gcr.io/kube-apiserver:v1.23.17
k8s.gcr.io/kube-controller-manager:v1.23.17
k8s.gcr.io/kube-scheduler:v1.23.17
k8s.gcr.io/kube-proxy:v1.23.17
k8s.gcr.io/pause:3.6
k8s.gcr.io/etcd:3.5.1-0
k8s.gcr.io/coredns/coredns:v1.8.6
```
将原来的`k8s.gcr.io`替换为`registry.cn-hangzhou.aliyuncs.com/google_containers/` 手动拉取下来再重新打tag为`k8s.gcr.io`即可
初始化成功之后会有如下提示
```shell
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.28.129.245:6443 --token aenz2j.dthwj7q39fz5pgv0 \
        --discovery-token-ca-cert-hash sha256:60f0c3f424c389a6615aac0186ce9f387053e715142fd9a0f9cd3848c47ef17e
```
可以使用最后的 `kubeadm join`命令将其他worker节点加入进来。

使用 `kubectl get node`检查节点状态
```shell
ubuntu@master:~$ kubectl get nodes
NAME     STATUS     ROLES                  AGE   VERSION
master   NotReady   control-plane,master   16m   v1.23.1
node1    NotReady   <none>                 77s   v1.23.1
node2    NotReady   <none>                 51s   v1.23.1
```
这里显示节点`NotReady`的状态，使用`kubectl get pod -n kube-system` 查看pod是否启动成功
```shell
ubuntu@master:~$ kubectl get pod -n kube-system
NAME                             READY   STATUS    RESTARTS   AGE
coredns-6d8c4cb4d-rtcft          0/1     Pending   0          16m
coredns-6d8c4cb4d-vmlfc          0/1     Pending   0          16m
etcd-master                      1/1     Running   0          17m
kube-apiserver-master            1/1     Running   0          17m
kube-controller-manager-master   1/1     Running   0          17m
kube-proxy-cx7x2                 1/1     Running   0          2m45s
kube-proxy-m5xb9                 1/1     Running   0          16m
kube-proxy-nk4ct                 1/1     Running   0          2m19s
kube-scheduler-master            1/1     Running   0          17m
```
**查看日志**
```shell
ubuntu@master:~$ sudo journalctl -f -u kubelet
Jul 06 15:45:06 master kubelet[10429]: E0706 15:45:06.095625   10429 kubelet.go:2347] "Container runtime network not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized"
Jul 06 15:45:09 master kubelet[10429]: I0706 15:45:09.454481   10429 cni.go:240] "Unable to update cni config" err="no networks found in /etc/cni/net.d"
Jul 06 15:45:11 master kubelet[10429]: E0706 15:45:11.102104   10429 kubelet.go:2347] "Container runtime network not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized"
Jul 06 15:45:14 master kubelet[10429]: I0706 15:45:14.455219   10429 cni.go:240] "Unable to update cni config" err="no networks found in /etc/cni/net.d"
Jul 06 15:45:16 master kubelet[10429]: E0706 15:45:16.110079   10429 kubelet.go:2347] "Container runtime network not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized"
```
原因是没有安装CNI插件
此处解决办法是手动部署一下CNI[参考地址](https://github.com/flannel-io/flannel)
执行`kubectl apply -f kube-flannel.yml` 部署一下flannel，然后使用命令`sudo systemctl restart kubelet` 重启kubelet，重启好之后再查看节点就是`Ready`状态。
```shell
ubuntu@master:~$ kubectl get nodes
NAME     STATUS   ROLES                  AGE   VERSION
master   Ready    control-plane,master   30m   v1.23.1
node1    Ready    <none>                 15m   v1.23.1
node2    Ready    <none>                 15m   v1.23.1
```

以下是`kube-flannel.yml`配置文件
```yaml
---
kind: Namespace
apiVersion: v1
metadata:
  name: kube-flannel
  labels:
    k8s-app: flannel
    pod-security.kubernetes.io/enforce: privileged
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  labels:
    k8s-app: flannel
  name: flannel
rules:
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - nodes/status
  verbs:
  - patch
- apiGroups:
  - networking.k8s.io
  resources:
  - clustercidrs
  verbs:
  - list
  - watch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  labels:
    k8s-app: flannel
  name: flannel
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: flannel
subjects:
- kind: ServiceAccount
  name: flannel
  namespace: kube-flannel
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: flannel
  name: flannel
  namespace: kube-flannel
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: kube-flannel-cfg
  namespace: kube-flannel
  labels:
    tier: node
    k8s-app: flannel
    app: flannel
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "flannel",
          "delegate": {
            "hairpinMode": true,
            "isDefaultGateway": true
          }
        },
        {
          "type": "portmap",
          "capabilities": {
            "portMappings": true
          }
        }
      ]
    }
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds
  namespace: kube-flannel
  labels:
    tier: node
    app: flannel
    k8s-app: flannel
spec:
  selector:
    matchLabels:
      app: flannel
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values:
                - linux
      hostNetwork: true
      priorityClassName: system-node-critical
      tolerations:
      - operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni-plugin
        image: docker.io/flannel/flannel-cni-plugin:v1.1.2
       #image: docker.io/rancher/mirrored-flannelcni-flannel-cni-plugin:v1.1.2
        command:
        - cp
        args:
        - -f
        - /flannel
        - /opt/cni/bin/flannel
        volumeMounts:
        - name: cni-plugin
          mountPath: /opt/cni/bin
      - name: install-cni
        image: docker.io/flannel/flannel:v0.22.0
       #image: docker.io/rancher/mirrored-flannelcni-flannel:v0.22.0
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
        image: docker.io/flannel/flannel:v0.22.0
       #image: docker.io/rancher/mirrored-flannelcni-flannel:v0.22.0
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: false
          capabilities:
            add: ["NET_ADMIN", "NET_RAW"]
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: EVENT_QUEUE_DEPTH
          value: "5000"
        volumeMounts:
        - name: run
          mountPath: /run/flannel
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
        - name: xtables-lock
          mountPath: /run/xtables.lock
      volumes:
      - name: run
        hostPath:
          path: /run/flannel
      - name: cni-plugin
        hostPath:
          path: /opt/cni/bin
      - name: cni
        hostPath:
          path: /etc/cni/net.d
      - name: flannel-cfg
        configMap:
          name: kube-flannel-cfg
      - name: xtables-lock
        hostPath:
          path: /run/xtables.lock
          type: FileOrCreate
```
接下来部署一个`Nginx`测试下
```shell
ubuntu@master:~$ kubectl create deployment nginx --image=nginx
deployment.apps/nginx created
```
查看`Pod`是否已经启动成功
```shell
ubuntu@master:~$ kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
nginx-85b98978db-9495s   1/1     Running   0          109s
```
使用`NodePort`方式将`Nginx` `80`端口暴露出去
```shell
ubuntu@master:~$ kubectl expose deployment nginx --port=80 --type=NodePort
service/nginx exposed
```
查看暴露端口
```shell
ubuntu@master:~$ kubectl get pod,svc
NAME                         READY   STATUS    RESTARTS   AGE
pod/nginx-85b98978db-9495s   1/1     Running   0          2m16s

NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
service/kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP        31m
service/nginx        NodePort    10.104.99.42   <none>        80:31608/TCP   7s
```
使用VM的IP加上端口号就能访问Nginx服务了
获取VM的IP
```shell
PS C:\Users\38189> multipass list
Name                    State             IPv4             Image
master                  Running           172.28.129.245   Ubuntu 22.04 LTS
                                          172.17.0.1
                                          10.244.0.0
                                          10.244.0.1
node1                   Running           172.28.129.127   Ubuntu 22.04 LTS
                                          172.17.0.1
                                          10.244.1.0
                                          10.244.1.1
node2                   Running           172.28.137.244   Ubuntu 22.04 LTS
                                          172.17.0.1
                                          10.244.2.0
                                          10.244.2.1
```
至此，kubernetes 本地集群搭建完毕