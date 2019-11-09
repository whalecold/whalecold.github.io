---
title: kubernetes 搭建
subtitle: Making a Note
date: 2018-09-26
categories: [
    "kubernetes",
]
tags: ["golang", "kubernetes"]
---

## kubernetes v1.10.6 搭建手记

这里主要是跟着这篇 [博客](https://blog.qikqiak.com/post/manual-install-high-available-kubernetes-cluster/) 做的，但是这篇博客是基于 1.8.2 的版本来的，和 1.10.6 有不少区别，所以再做个记录。

* [Build Enviroment](#build-enviroment)
* [Build Etcd Cluster](#build-etcd-cluster)
    * [Install cfssl](#install-cfssl)
    * [build ca](#build-ca)
    * [install etcd](#install-etcd)
* [Build flannel network](#build-flannel-network)
* [build master components](#build-master-components)
    * [build kube-apiserver](#build-kube-apiserver)
    * [build kube-controller-manager](#build-kube-controller-manager)
    * [build kube-scheduler](#build-kube-scheduler)
* [build kubectl](#build-kubectl)
* [build node components](#build-node-components)
    * [install kubelet](#install-kubelet)
    * [install kube-proxy](#install-kube-proxy)
* [build plug-in](#build-plug-in)
    * [install coredns](#install-coredns)
    * [install heapster](#install-heapster)



### Build Enviroment
为了方便搭建目前只是用了两台虚拟机测试，ip 分别是 192.168.21.8 和 192.168.21.9， os 是 centos 7.4

#### 组件版本

+ Kubernetes 1.10.6
+ Docker 17.03.1-ce (最新的对 container-selinux 版本有要求)
+ Etcd 3.3.9
+ Flanneld
+ TLS 认证通信（所有组件，如 etcd、kubernetes master 和 node）
+ kubedns、dashboard、heapster 等插件

#### 自己先设置好环境变量

后面的署将会使用到下面的变量，定义如下（根据自己的机器、网络修改）：
```
 #TLS Bootstrapping 使用的 Token，可以使用命令 head -c 16 /dev/urandom | od -An -t x  > tr -d ' ' 生成
 BOOTSTRAP_TOKEN="8981b594122ebed7596f1d3b69c78223"

 #建议使用未用的网段来定义服务网段和 Pod 网段
 #服务网段 (Service CIDR)，部署前路由不可达，部署后集群内部使用 IP:Port 可达
 SERVICE_CIDR="10.254.0.0/16"

 #Pod 网段 (Cluster CIDR)，部署前路由不可达，部署后路由可达 (flanneld 保证)
 CLUSTER_CIDR="172.30.0.0/16"

 #服务端口范围 (NodePort Range)
 NODE_PORT_RANGE="30000-32766"

 #etcd 集群服务地址列表 这里暂时先只有一台
 ETCD_ENDPOINTS="https://192.168.31.8:2379"

 #flanneld 网络配置前缀
 FLANNEL_ETCD_PREFIX="/kubernetes/network"

 #kubernetes 服务 IP(预先分配，一般为 SERVICE_CIDR 中的第一个 IP)
 CLUSTER_KUBERNETES_SVC_IP="10.254.0.1"

 #集群 DNS 服务 IP(从 SERVICE_CIDR 中预先分配)
 CLUSTER_DNS_SVC_IP="10.254.0.2"

 #集群 DNS 域名
 CLUSTER_DNS_DOMAIN="cluster.local."

 #MASTER API Server 地址
 MASTER_URL="k8s-api.virtual.local"
```
保存为 `env.sh`, 赋加可执行权限 `chmod +x env.sh`, 执行 `mkdir -p /usr/k8s/bin`, 将这个目录添加到系统可执行目录里面，`export PATH=/usr/k8s/bin:$PATH`, 为了方便可以把这个命令添加到 `～/.bashrc` 里面， 把脚本添加到上面的目录中。

### Build Etcd Cluster
#### Install Cfssl
kubernetes 系统需要使用 `TLS` 证书对通信加密， 这里使用 `cfssl` 生成证书。

```
 $ wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
 $ chmod +x cfssl_linux-amd64
 $ mv cfssl_linux-amd64 /usr/k8s/bin/cfssl

 $ wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
 $ chmod +x cfssljson_linux-amd64
 $ mv cfssljson_linux-amd64 /usr/k8s/bin/cfssljson

 $ wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
 $ chmod +x cfssl-certinfo_linux-amd64
 $ mv cfssl-certinfo_linux-amd64 /usr/k8s/bin/cfssl-certinfo

 $ mkdir ssl && cd ssl
 $ cfssl print-defaults config > config.json
 $ cfssl print-defaults csr > csr.json
```

#### Build Ca
修改上面创建的 `config.json` 文件为 `ca-config.json`：
```
> {
>     "signing": {
>         "default": {
>             "expiry": "87600h"
>         },
>         "profiles": {
>             "kubernetes": {
>                 "expiry": "87600h",
>                 "usages": [
>                     "signing",
>                     "key encipherment",
>                     "server auth",
>                     "client auth"
>                 ]
>             }
>         }
>     }
> }
```
+ `config.json`：可以定义多个 profiles，分别指定不同的过期时间、使用场景等参数；后续在签名证书时使用某个 profile。
+ `signing`: 表示该证书可用于签名其它证书；生成的 ca.pem 证书中 CA=TRUE。
+ `server auth`: 表示 client 可以用该 CA 对 server 提供的证书进行校验。
+ `client auth`: 表示 server 可以用该 CA 对 client 提供的证书进行验证。

修改 CA 证书签名请求为 `ca-csr.json`：
```
{
    "CN": "kubernetes",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "BeiJing",
            "ST": "BeiJing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
```
生成 CA 证书和私钥：
```
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
ls ca*
ca-config.json  ca.csr  ca-csr.json  ca-key.pem  ca.pem
```
将生成的 CA 证书、密钥文件、配置文件拷贝到所有机器的 / etc/kubernetes/ssl 目录下面：
```
mkdir -p /etc/kubernetes/ssl
cp ca* /etc/kubernetes/ssl
```
#### Install Etcd
只是测试用 所以只部署一个节点 `192.168.21.8`，命名是 `etcd01`：

##### 定义环境变量
```
$ export NODE_NAME=etcd01 # 当前部署的机器名称 (随便定义，只要能区分不同机器即可)
$ export NODE_IP=192.168.21.8 # 当前部署的机器 IP
$ export NODE_IPS="192.168.21.8" # etcd 集群所有机器 IP
$ # etcd 集群间通信的 IP 和端口
$ export ETCD_NODES=etcd01=https://192.168.21.8:2380
$ # 导入用到的其它全局变量：ETCD_ENDPOINTS、FLANNEL_ETCD_PREFIX、CLUSTER_CIDR
$ source /usr/k8s/bin/env.sh
```
##### 下载 etcd 二进制文件
自己去 [github](https://github.com/coreos/etcd/releases) 去下载找到对应版本就好了

##### 创建 TLS 密钥和证书
创建 etcd 证书签名请求：
```
$ cat > etcd-csr.json <<EOF
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "${NODE_IP}"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
```
+ `NODE_IP` 即上面的全局变量

生成 etcd 证书和私钥：
```
$ cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
  -config=/etc/kubernetes/ssl/ca-config.json \
  -profile=kubernetes etcd-csr.json | cfssljson -bare etcd
$ ls etcd*
etcd.csr  etcd-csr.json  etcd-key.pem  etcd.pem
$  mkdir -p /etc/etcd/ssl
$  mv etcd*.pem /etc/etcd/ssl/
```

创建 etcd 的 systemd unit 文件

```
$ mkdir -p /var/lib/etcd  # 必须要先创建工作目录
$ cat > etcd.service <<EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
ExecStart=/usr/k8s/bin/etcd \\
  --name=${NODE_NAME} \\
  --cert-file=/etc/etcd/ssl/etcd.pem \\
  --key-file=/etc/etcd/ssl/etcd-key.pem \\
  --peer-cert-file=/etc/etcd/ssl/etcd.pem \\
  --peer-key-file=/etc/etcd/ssl/etcd-key.pem \\
  --trusted-ca-file=/etc/kubernetes/ssl/ca.pem \\
  --peer-trusted-ca-file=/etc/kubernetes/ssl/ca.pem \\
  --initial-advertise-peer-urls=https://${NODE_IP}:2380 \\
  --listen-peer-urls=https://${NODE_IP}:2380 \\
  --listen-client-urls=https://${NODE_IP}:2379,http://127.0.0.1:2379 \\
  --advertise-client-urls=https://${NODE_IP}:2379 \\
  --initial-cluster-token=etcd-cluster-0 \\
  --initial-cluster=${ETCD_NODES} \\
  --initial-cluster-state=new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

启动 etcd 服务

```
mv etcd.service /etc/systemd/system/
systemctl daemon-reload
systemctl enable etcd
systemctl start etcd
systemctl status etcd
```
验证
```
for ip in ${NODE_IPS}; do
  ETCDCTL_API=3 /usr/k8s/bin/etcdctl \
  --endpoints=https://${ip}:2379  \
  --cacert=/etc/kubernetes/ssl/ca.pem \
  --cert=/etc/etcd/ssl/etcd.pem \
  --key=/etc/etcd/ssl/etcd-key.pem \
  endpoint health; done
```
结果
```
https://192.168.21.8:2379 is healthy: successfully committed proposal: took = 2.132456ms
```
etcd 到这里就已经搭建好了，下面开始搭建 flanneld 网络。

### Build Flannel Network

> 需要在所有的 node 节点安装

##### 环境变量
```
$ export NODE_IP=192.168.21.8  # 当前部署节点的 IP
# 导入全局变量
$ source /usr/k8s/bin/env.sh
```
##### 创建 TLS 密钥和证书
etcd 集群启用了双向 TLS 认证，所以需要为 flanneld 指定与 etcd 集群通信的 CA 和密钥。

创建 flanneld 证书签名请求：
```
$ cat > flanneld-csr.json <<EOF
{
  "CN": "flanneld",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
```
生成 flanneld 证书和私钥：
```
$ cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
  -config=/etc/kubernetes/ssl/ca-config.json \
  -profile=kubernetes flanneld-csr.json | cfssljson -bare flanneld
$ ls flanneld*
flanneld.csr  flanneld-csr.json  flanneld-key.pem flanneld.pem
$ sudo mkdir -p /etc/flanneld/ssl
$ sudo mv flanneld*.pem /etc/flanneld/ssl
```

##### 向 etcd 写入集群 Pod 网段信息
> 该步骤只需在第一次部署 Flannel 网络时执行，后续在其他节点上部署 Flanneld 时无需再写入该信息
```
$ etcdctl \
  --endpoints=${ETCD_ENDPOINTS} \
  --ca-file=/etc/kubernetes/ssl/ca.pem \
  --cert-file=/etc/flanneld/ssl/flanneld.pem \
  --key-file=/etc/flanneld/ssl/flanneld-key.pem \
  set ${FLANNEL_ETCD_PREFIX}/config '{"Network":"'${CLUSTER_CIDR}'","SubnetLen": 24,"Backend": {"Type":"vxlan"}}'
# 得到如下反馈信息
{"Network":"172.30.0.0/16", "SubnetLen": 24, "Backend": {"Type": "vxlan"}}
```
+ 写入的 Pod 网段 (${CLUSTER_CIDR}，172.30.0.0/16) 必须与 kube-controller-manager 的 --cluster-cidr 选项值一致；

##### 安装和配置 flanneld

先去 [flanneld release](https://github.com/coreos/flannel/releases) 页面下载最新版的 flanneld 二进制文件。
```
$ mkdir flannel
$ wget https://github.com/coreos/flannel/releases/download/v0.9.0/flannel-v0.9.0-linux-amd64.tar.gz
$ tar -xzvf flannel-v0.9.0-linux-amd64.tar.gz -C flannel
$ cp flannel/{flanneld,mk-docker-opts.sh} /usr/k8s/bin
```

创建 flanneld 的 systemd unit 文件
```
$ cat > flanneld.service << EOF
[Unit]
Description=Flanneld overlay address etcd agent
After=network.target
After=network-online.target
Wants=network-online.target
After=etcd.service
Before=docker.service

[Service]
Type=notify
ExecStart=/usr/k8s/bin/flanneld \\
  -etcd-cafile=/etc/kubernetes/ssl/ca.pem \\
  -etcd-certfile=/etc/flanneld/ssl/flanneld.pem \\
  -etcd-keyfile=/etc/flanneld/ssl/flanneld-key.pem \\
  -etcd-endpoints=${ETCD_ENDPOINTS} \\
  -etcd-prefix=${FLANNEL_ETCD_PREFIX}
ExecStartPost=/usr/k8s/bin/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/docker
Restart=on-failure

[Install]
WantedBy=multi-user.target
RequiredBy=docker.service
EOF
```
启动 flanneld
```
$ cp flanneld.service /etc/systemd/system/
$ systemctl daemon-reload
$ systemctl enable flanneld
$ systemctl start flanneld
$ systemctl status flanneld
```

检查服务命令：`ifconfig flannel.1`

检查分配给各 flanneld 的 Pod 网段信息
```
$ # 查看集群 Pod 网段 (/16)
$ etcdctl \
  --endpoints=${ETCD_ENDPOINTS} \
  --ca-file=/etc/kubernetes/ssl/ca.pem \
  --cert-file=/etc/flanneld/ssl/flanneld.pem \
  --key-file=/etc/flanneld/ssl/flanneld-key.pem \
  get ${FLANNEL_ETCD_PREFIX}/config
{"Network": "172.30.0.0/16", "SubnetLen": 24, "Backend": { "Type": "vxlan"} }
$ # 查看已分配的 Pod 子网段列表 (/24)
$ etcdctl \
  --endpoints=${ETCD_ENDPOINTS} \
  --ca-file=/etc/kubernetes/ssl/ca.pem \
  --cert-file=/etc/flanneld/ssl/flanneld.pem \
  --key-file=/etc/flanneld/ssl/flanneld-key.pem \
  ls ${FLANNEL_ETCD_PREFIX}/subnets
/kubernetes/network/subnets/172.30.77.0-24
$ # 查看某一 Pod 网段对应的 flanneld 进程监听的 IP 和网络参数
$ etcdctl \
  --endpoints=${ETCD_ENDPOINTS} \
  --ca-file=/etc/kubernetes/ssl/ca.pem \
  --cert-file=/etc/flanneld/ssl/flanneld.pem \
  --key-file=/etc/flanneld/ssl/flanneld-key.pem \
  get ${FLANNEL_ETCD_PREFIX}/subnets/172.30.77.0-24
{"PublicIP":"192.168.1.137","BackendType":"vxlan","BackendData":{"VtepMAC":"62:fc:03:83:1b:2b"}}
```
确保各节点间 Pod 网段能互联互通
在各个节点部署完 Flanneld 后，查看已分配的 Pod 子网段列表：

```
$ etcdctl \
  --endpoints=${ETCD_ENDPOINTS} \
  --ca-file=/etc/kubernetes/ssl/ca.pem \
  --cert-file=/etc/flanneld/ssl/flanneld.pem \
  --key-file=/etc/flanneld/ssl/flanneld-key.pem \
  ls ${FLANNEL_ETCD_PREFIX}/subnets

/kubernetes/network/subnets/172.30.54.0-24
/kubernetes/network/subnets/172.30.74.0-24
```

### Build Master Components
kubernetes master 节点包含的组件有：
+ kube-apiserver
+ kube-scheduler
+ kube-controller-manager

这三个组件需要在一台机器上面。</br>
master 节点与 node 节点上的 Pods 通过 Pod 网络通信，所以需要在 master 节点上部署 Flannel 网络。
#### 环境变量
```
$ export NODE_IP=192.168.21.8  # 当前部署的 master 机器 IP
$ source /usr/k8s/bin/env.sh
```
#### 下载对应版本的二进制文件：
在 [kubernetes changelog](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG-1.10.md#server-binaries) 页面下载最新版本的文件:
```
$ wget https://storage.googleapis.com/kubernetes-release/release/v1.10.6/kubernetes-server-linux-amd64.tar.gz
$ tar -xzvf kubernetes-server-linux-amd64.tar.gz
```
然后将里面的二进制文件拷贝到 `/usr/k8s/bin` 目录里面
#### 创建 kubernetes 证书
创建 kubernetes 证书签名请求：
```
$ cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "hosts": [
    "127.0.0.1",
    "${NODE_IP}",
    "${MASTER_URL}",
    "${CLUSTER_KUBERNETES_SVC_IP}",
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
```
生成 kubernetes 证书和私钥：
```
$ cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
  -config=/etc/kubernetes/ssl/ca-config.json \
  -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes
$ ls kubernetes*
kubernetes.csr  kubernetes-csr.json  kubernetes-key.pem  kubernetes.pem
$ mkdir -p /etc/kubernetes/ssl/
$ mv kubernetes*.pem /etc/kubernetes/ssl/
```
#### Build Kube-apiserver
##### 创建 kube-apiserver 使用的客户端 token 文件
kubelet 首次启动时向 kube-apiserver 发送 TLS Bootstrapping 请求，kube-apiserver 验证请求中的 token 是否与它配置的 token.csv 一致，如果一致则自动为 kubelet 生成证书和密钥。
```
$ # 导入的 environment.sh 文件定义了 BOOTSTRAP_TOKEN 变量
$ cat > token.csv <<EOF
${BOOTSTRAP_TOKEN},kubelet-bootstrap,10001,"system:kubelet-bootstrap"
EOF
$ mv token.csv /etc/kubernetes/
```
##### 创建 kube-apiserver 的 systemd unit 文件
在创建之前需要先生成一个日志策略文件 (`/etc/kubernetes/audit-policy.yaml`):
```
apiVersion: audit.k8s.io/v1beta1 # This is required.
kind: Policy
# Don't generate audit events for all requests in RequestReceived stage.
omitStages:
  - "RequestReceived"
rules:
  # Log pod changes at RequestResponse level
  - level: RequestResponse
    resources:
    - group: ""
      # Resource "pods" doesn't match requests to any subresource of pods,
      # which is consistent with the RBAC policy.
      resources: ["pods"]
  # Log "pods/log", "pods/status" at Metadata level
  - level: Metadata
    resources:
    - group: ""
      resources: ["pods/log", "pods/status"]

  # Don't log requests to a configmap called"controller-leader"
  - level: None
    resources:
    - group: ""
      resources: ["configmaps"]
      resourceNames: ["controller-leader"]

  # Don't log watch requests by the"system:kube-proxy" on endpoints or services
  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
    resources:
    - group: "" # core API group
      resources: ["endpoints", "services"]

  # Don't log authenticated requests to certain non-resource URL paths.
  - level: None
    userGroups: ["system:authenticated"]
    nonResourceURLs:
    - "/api*" # Wildcard matching.
    - "/version"

  # Log the request body of configmap changes in kube-system.
  - level: Request
    resources:
    - group: "" # core API group
      resources: ["configmaps"]
    # This rule only applies to resources in the "kube-system" namespace.
    # The empty string "" can be used to select non-namespaced resources.
    namespaces: ["kube-system"]

  # Log configmap and secret changes in all other namespaces at the Metadata level.
  - level: Metadata
    resources:
    - group: "" # core API group
      resources: ["secrets", "configmaps"]

  # Log all other resources in core and extensions at the Request level.
  - level: Request
    resources:
    - group: "" # core API group
    - group: "extensions" # Version of group should NOT be included.

  # A catch-all rule to log all other requests at the Metadata level.
  - level: Metadata
    # Long-running requests like watches that fall under this rule will not
    # generate an audit event in RequestReceived.
    omitStages:
      - "RequestReceived"
```
审查日志的相关配置可以查看 [文档了解](https://kubernetes.io/docs/tasks/debug-application-cluster/audit/)

下面是 unit 文件:
```
$ cat  > kube-apiserver.service <<EOF
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
ExecStart=/usr/k8s/bin/kube-apiserver \\
  --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota,NodeRestriction \\
  --advertise-address=${NODE_IP} \\
  --bind-address=0.0.0.0 \\
  --insecure-bind-address=${NODE_IP} \\
  --authorization-mode=Node,RBAC \\
  --runtime-config=rbac.authorization.k8s.io/v1alpha1 \\
  --kubelet-https=true \\
  --token-auth-file=/etc/kubernetes/token.csv \\
  --service-cluster-ip-range=${SERVICE_CIDR} \\
  --service-node-port-range=${NODE_PORT_RANGE} \\
  --tls-cert-file=/etc/kubernetes/ssl/kubernetes.pem \\
  --tls-private-key-file=/etc/kubernetes/ssl/kubernetes-key.pem \\
  --client-ca-file=/etc/kubernetes/ssl/ca.pem \\
  --service-account-key-file=/etc/kubernetes/ssl/ca-key.pem \\
  --etcd-cafile=/etc/kubernetes/ssl/ca.pem \\
  --etcd-certfile=/etc/kubernetes/ssl/kubernetes.pem \\
  --etcd-keyfile=/etc/kubernetes/ssl/kubernetes-key.pem \\
  --etcd-servers=${ETCD_ENDPOINTS} \\
  --enable-swagger-ui=true \\
  --allow-privileged=true \\
  --apiserver-count=2 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/lib/audit.log \\
  --audit-policy-file=/etc/kubernetes/audit-policy.yaml \\
  --event-ttl=1h \\
  --logtostderr=true \\
  --v=6
Restart=on-failure
RestartSec=5
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

##### 启动 kube-apiserver
```
$ cp kube-apiserver.service /etc/systemd/system/
$ systemctl daemon-reload
$ systemctl enable kube-apiserver
$ systemctl start kube-apiserver
$ systemctl status kube-apiserver
```
#### Build Kube-controller-manager
##### 创建 kube-controller-manager 的 systemd unit 文件
```
$ cat > kube-controller-manager.service <<EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/k8s/bin/kube-controller-manager \\
  --address=127.0.0.1 \\
  --master=http://${MASTER_URL}:8080 \\
  --allocate-node-cidrs=true \\
  --service-cluster-ip-range=${SERVICE_CIDR} \\
  --cluster-cidr=${CLUSTER_CIDR} \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/etc/kubernetes/ssl/ca.pem \\
  --cluster-signing-key-file=/etc/kubernetes/ssl/ca-key.pem \\
  --service-account-private-key-file=/etc/kubernetes/ssl/ca-key.pem \\
  --root-ca-file=/etc/kubernetes/ssl/ca.pem \\
  --leader-elect=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
##### 启动 kube-controller-manager
```
$ cp kube-controller-manager.service /etc/systemd/system/
$ systemctl daemon-reload
$ systemctl enable kube-controller-manager
$ systemctl start kube-controller-manager
$ systemctl status kube-controller-manager
```
#### Build Kube-scheduler
##### 创建 kube-scheduler 的 systemd unit 文件
```
$ cat > kube-scheduler.service <<EOF
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/k8s/bin/kube-scheduler \\
  --address=127.0.0.1 \\
  --master=http://${MASTER_URL}:8080 \\
  --leader-elect=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
##### 启动 kube-scheduler
```
$ cp kube-scheduler.service /etc/systemd/system/
$ systemctl daemon-reload
$ systemctl enable kube-scheduler
$ systemctl start kube-scheduler
$ systemctl status kube-scheduler
```
### Build Kubectl
kubectl 默认从 `~/.kube/config` 配置文件中获取访问 kube-apiserver 地址、证书、用户名等信息，需要正确配置该文件才能正常使用 kubectl 命令。

#### 环境变量
```
$ source /usr/k8s/bin/env.sh
$ export KUBE_APISERVER="https://${MASTER_URL}:6443"
```
#### 下载 kubectl
```
$ wget https://dl.k8s.io/v1.10.6/kubernetes-client-linux-amd64.tar.gz # 如果服务器上下载不下来，可以想办法下载到本地，然后 scp 上去即可
$ tar -xzvf kubernetes-client-linux-amd64.tar.gz
$ cp kubernetes/client/bin/kube* /usr/k8s/bin/
$ chmod +x /usr/k8s/bin/kube*
```
#### 创建 admin 证书
kubectl 与 kube-apiserver 的安全端口通信，需要为安全通信提供 TLS 证书和密钥。创建 admin 证书签名请求：
```
$ cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
EOF
```
生成 admin 证书和私钥：
```
$ cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
  -config=/etc/kubernetes/ssl/ca-config.json \
  -profile=kubernetes admin-csr.json | cfssljson -bare admin
$ ls admin
admin.csr  admin-csr.json  admin-key.pem  admin.pem
$ sudo mv admin*.pem /etc/kubernetes/ssl/
```
#### 创建 kubectl kubeconfig 文件
```
# 设置集群参数
$ kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER}
# 设置客户端认证参数
$ kubectl config set-credentials admin \
  --client-certificate=/etc/kubernetes/ssl/admin.pem \
  --embed-certs=true \
  --client-key=/etc/kubernetes/ssl/admin-key.pem \
  --token=${BOOTSTRAP_TOKEN}
# 设置上下文参数
$ kubectl config set-context kubernetes \
  --cluster=kubernetes \
  --user=admin
# 设置默认上下文
$ kubectl config use-context kubernetes
```
将 `~/.kube/config` 文件拷贝到运行 kubectl 命令的机器的 `~/.kube/` 目录下去。

完成后验证下 `master` 节点：
```
# kubectl get componentstatuses
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-0               Healthy   {"health":"true"}
```
### Build Node Components
kubernetes Node 节点包含如下组件：
+ flanneld
+ docker
+ kubelet
+ kube-proxy

#### 环境变量
```
# source /usr/k8s/bin/env.sh
# export KUBE_APISERVER="https://${MASTER_URL}"  // 如果你没有安装 `haproxy` 的话，还是需要使用 6443 端口的哦
# export NODE_IP=192.168.21.9  # 当前部署的节点 IP
```
按照上面的步骤安装配置好 flanneld

#### 开启路由转发
修改 `/etc/sysctl.conf` 文件，添加下面的规则：
```
net.ipv4.ip_forward=1
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
```
执行下面的命令立即生效：
```
$ sysctl -p
```

#### 配置 docker
你可以用二进制或 yum install 的方式来安装 docker，然后修改 docker 的 systemd unit 文件：

```
$ cat /usr/lib/systemd/system/docker.service  # 用 systemctl status docker 命令可查看 unit 文件路径
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target

[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
EnvironmentFile=-/run/flannel/docker
ExecStart=/usr/bin/dockerd --log-level=info $DOCKER_NETWORK_OPTIONS
ExecReload=/bin/kill -s HUP $MAINPID
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
# Uncomment TasksMax if your systemd version supports it.
# Only systemd 226 and above support this version.
#TasksMax=infinity
TimeoutStartSec=0
# set delegate yes so that systemd does not reset the cgroups of docker containers
Delegate=yes
# kill only the docker process, not all processes in the cgroup
KillMode=process
# restart the docker process if it exits prematurely
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s

[Install]
WantedBy=multi-user.target
```
` 注意关闭防火墙 使用方便点。。然后可以配置国内的 docker 镜像源 `
#### 启动 docker
```
$ sudo systemctl daemon-reload
$ sudo systemctl stop firewalld
$ sudo systemctl disable firewalld
$ sudo iptables -F && sudo iptables -X && sudo iptables -F -t nat && sudo iptables -X -t nat
$ sudo systemctl enable docker
$ sudo systemctl start docker
```

#### Install Kubelet

kubelet 启动时向 kube-apiserver 发送 TLS bootstrapping 请求，需要先将 bootstrap token 文件中的 kubelet-bootstrap 用户赋予 system:node-bootstrapper 角色，然后 kubelet 才有权限创建认证请求 (certificatesigningrequests)：

>kubelet 就是运行在 Node 节点上的，所以这一步安装是在所有的 Node 节点上，如果你想把你的 Master 也当做 Node 节点的话，当然也可以在 Master 节点上安装的。

```
kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap
```
+ `--user=kubelet-bootstrap` 是文件 `/etc/kubernetes/token.csv` 中指定的用户名，同时也写入了文件 `/etc/kubernetes/bootstrap.kubeconfig`

##### 创建 kubelet bootstapping kubeconfig 文件
```
$ # 设置集群参数
$ kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=bootstrap.kubeconfig
$ # 设置客户端认证参数
$ kubectl config set-credentials kubelet-bootstrap \
  --token=${BOOTSTRAP_TOKEN} \
  --kubeconfig=bootstrap.kubeconfig
$ # 设置上下文参数
$ kubectl config set-context default \
  --cluster=kubernetes \
  --user=kubelet-bootstrap \
  --kubeconfig=bootstrap.kubeconfig
$ # 设置默认上下文
$ kubectl config use-context default --kubeconfig=bootstrap.kubeconfig
$ mv bootstrap.kubeconfig /etc/kubernetes/
```
##### 创建 kubelet 的 systemd unit 文件
` 注意 `：pod-infra-container-image 这个配置项后面加镜像源 主要是考虑到 k8s 官方源国内访问不了, 需要单独去配置 我这里用的是私有源
```
$ sudo mkdir /var/lib/kubelet # 必须先创建工作目录
$ cat > kubelet.service <<EOF
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
ExecStart=/usr/k8s/bin/kubelet \\
  --fail-swap-on=false \\
  --cgroup-driver=cgroupfs \\
  --address=${NODE_IP} \\
  --hostname-override=${NODE_IP} \\
  --experimental-bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig \\
  --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \\
  --pod-infra-container-image=gcr.io/google_containers/pause-amd64:3.1 \
  --cert-dir=/etc/kubernetes/ssl \\
  --cluster-dns=${CLUSTER_DNS_SVC_IP} \\
  --cluster-domain=${CLUSTER_DNS_DOMAIN} \\
  --hairpin-mode promiscuous-bridge \\
  --allow-privileged=true \\
  --serialize-image-pulls=false \\
  --logtostderr=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

##### 启动 kubelet
```
$ sudo cp kubelet.service /etc/systemd/system/kubelet.service
$ sudo systemctl daemon-reload
$ sudo systemctl enable kubelet
$ sudo systemctl start kubelet
$ systemctl status kubelet
```

##### 通过 kubelet 的 TLS 证书请求
```
$ kubectl get csr
NAME                                                   AGE       REQUESTOR           CONDITION
node-csr--k3G2G1EoM4h9w1FuJRjJjfbIPNxa551A8TZfW9dG-g   2m        kubelet-bootstrap   Pending
$ kubectl get nodes
No resources found.
```
通过 CSR 请求：
```
$ kubectl certificate approve node-csr--k3G2G1EoM4h9w1FuJRjJjfbIPNxa551A8TZfW9dG-g
certificatesigningrequest "node-csr--k3G2G1EoM4h9w1FuJRjJjfbIPNxa551A8TZfW9dG-g" approved
$ kubectl get nodes
NAME            STATUS    ROLES     AGE       VERSION
192.168.21.9   Ready     <none>    48s       v1.10.6
```
自动生成了 kubelet kubeconfig 文件和公私钥：
```
$ ls -l /etc/kubernetes/kubelet.kubeconfig
-rw------- 1 root root 2280 Nov  7 10:26 /etc/kubernetes/kubelet.kubeconfig
$ ls -l /etc/kubernetes/ssl/kubelet*
-rw-r--r-- 1 root root 1046 Nov  7 10:26 /etc/kubernetes/ssl/kubelet-client.crt
-rw------- 1 root root  227 Nov  7 10:22 /etc/kubernetes/ssl/kubelet-client.key
-rw-r--r-- 1 root root 1115 Nov  7 10:16 /etc/kubernetes/ssl/kubelet.crt
-rw------- 1 root root 1675 Nov  7 10:16 /etc/kubernetes/ssl/kubelet.key
```
#### Install Kube-proxy
##### 创建 kube-proxy 证书签名请求：
```
$ cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
```
##### 生成 kube-proxy 客户端证书和私钥
```
$ cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
  -config=/etc/kubernetes/ssl/ca-config.json \
  -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy
$ ls kube-proxy*
kube-proxy.csr  kube-proxy-csr.json  kube-proxy-key.pem  kube-proxy.pem
$ mv kube-proxy*.pem /etc/kubernetes/ssl/
```
##### 创建 kube-proxy kubeconfig 文件
```
$ # 设置集群参数
$ kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=kube-proxy.kubeconfig
$ # 设置客户端认证参数
$ kubectl config set-credentials kube-proxy \
  --client-certificate=/etc/kubernetes/ssl/kube-proxy.pem \
  --client-key=/etc/kubernetes/ssl/kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig
$ # 设置上下文参数
$ kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig
$ # 设置默认上下文
$ kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
$ mv kube-proxy.kubeconfig /etc/kubernetes/
```

##### 创建 kube-proxy 的 systemd unit 文件
```
$ sudo mkdir -p /var/lib/kube-proxy # 必须先创建工作目录
$ cat > kube-proxy.service <<EOF
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
WorkingDirectory=/var/lib/kube-proxy
ExecStart=/usr/k8s/bin/kube-proxy \\
  --bind-address=${NODE_IP} \\
  --hostname-override=${NODE_IP} \\
  --cluster-cidr=${SERVICE_CIDR} \\
  --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig \\
  --logtostderr=true \\
  --v=2
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

##### 启动 kube-proxy
```
$ sudo cp kube-proxy.service /etc/systemd/system/
$ sudo systemctl daemon-reload
$ sudo systemctl enable kube-proxy
$ sudo systemctl start kube-proxy
$ systemctl status kube-proxy
```

#### 验证集群功能
定义 yaml 文件：（将下面内容保存为：nginx-ds.yaml）

```
apiVersion: v1
kind: Service
metadata:
  name: nginx-ds
  labels:
    app: nginx-ds
spec:
  type: NodePort
  selector:
    app: nginx-ds
  ports:
  - name: http
    port: 80
    targetPort: 80
---
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: nginx-ds
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  template:
    metadata:
      labels:
        app: nginx-ds
    spec:
      containers:
      - name: my-nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```
创建 Pod 和服务：
```
$ kubectl create -f nginx-ds.yml
service "nginx-ds" created
daemonset "nginx-ds" created
```
执行下面的命令查看 Pod 和 SVC：
```
$ kubectl get pods -o wide
NAME             READY     STATUS    RESTARTS   AGE       IP           NODE
nginx-ds-f29zt   1/1       Running   0          23m       172.30.54.2   192.168.21.9
$ kubectl get svc
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
nginx-ds     NodePort    10.0.0.169   <none>        80:30265/TCP   24m
```

### Build Plug-in
#### Install Coredns
CoreDNS 给出了标准的 deployment 配置，如下
+ coredns.yaml.sed

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: coredns
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:coredns
rules:
- apiGroups:
  - ""
  resources:
  - endpoints
  - services
  - pods
  - namespaces
  verbs:
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:coredns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:coredns
subjects:
- kind: ServiceAccount
  name: coredns
  namespace: kube-system
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        kubernetes CLUSTER_DOMAIN REVERSE_CIDRS {
          pods insecure
          upstream
          fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        proxy . /etc/resolv.conf
        cache 30
    }
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/name: "CoreDNS"
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  selector:
    matchLabels:
      k8s-app: kube-dns
  template:
    metadata:
      labels:
        k8s-app: kube-dns
    spec:
      serviceAccountName: coredns
      tolerations:
        - key: "CriticalAddonsOnly"
          operator: "Exists"
      containers:
      - name: coredns
        image: coredns/coredns:1.1.1
        imagePullPolicy: IfNotPresent
        args: ["-conf", "/etc/coredns/Corefile"]
        volumeMounts:
        - name: config-volume
          mountPath: /etc/coredns
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        - containerPort: 9153
          name: metrics
          protocol: TCP
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
      dnsPolicy: Default
      volumes:
        - name: config-volume
          configMap:
            name: coredns
            items:
            - key: Corefile
              path: Corefile
---
apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
  annotations:
    prometheus.io/scrape: "true"
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: "CoreDNS"
spec:
  selector:
    k8s-app: kube-dns
  clusterIP: CLUSTER_DNS_IP
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
```

这个文件在所在 github 地址：`https://github.com/kubernetes/kubernetes/tree/release-1.10/cluster/addons/dns`

需要对这个文件做一些改动：
>+ 61 行 kubernetes $DNS_DOMAIN in-addr.arpa ip6.arpa 中的 $DNS_DOMAIN 改为 cluster.local
>+ 153 行 clusterIP: $DNS_SERVER_IP 中 $DNS_SERVER_IP 改为全局变量 CLUSTER_DNS_SVC_IP 的值

##### 创建 dns
```
kubectl create -f coredns.yaml
```
#### Install Heapster
yaml 创建一下就可以了
```
kubectl create -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/grafana.yaml
kubectl create -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/heapster.yaml
kubectl create -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/influxdb.yaml
kubectl create -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/rbac/heapster-rbac.yaml
```
在国内可能会失败，因为访问不到 k8s 镜像源。这样需要把 yaml 下载到本地, 然后修改 image 后面的选项。
