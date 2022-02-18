---
title: "甲骨文云安装K3S"
last_modified_at: 2022-01-09T16:20:02-05:00
categories:
  - Blog
tags:
  - OracleCloud
  - 甲骨文云
  - k3s
toc: true
toc_sticky: true
# classes: wide
---

前段时间申请到了甲骨文云的账号，可以白嫖 3 台永久免费的 VM 实例，2 台 1 核心 1G 的 AMD64 和 1 台 4 核 24G 内存的 ARM64 的主机（ARM 架构的主机常年缺货，不定期会有），让我忍不住直呼真香，简直太良心了，Azure、AWS、GCP、阿里云你们学着点啊，不仅如此还有 oracle 的数据库和负载均衡等，具体可以看[这里](https://www.oracle.com/cn/cloud/free/?source=CloudFree_CTA1_Default&intcmp=CloudFree_CTA1_Default)。如果只是用来魔法上网就有点太浪费了，本着把它的价值榨干的原则，打算搭建一个 Kubernetes 集群来玩一玩，所有的服务都跑在 k8s 中岂不美哉，我这里使用 `K3S` 来搭建，使用 K3S 原因很简单，相较 K8S 对性能要求较低低，甚至可以跑在树莓派中，且具备 K8S 中的绝大多数功能，个人认为用来学习把玩非常适合我这个甲骨文云的环境。

可以结合这个视频看 😄

{% include video id="E-Vuw0ob-pc" provider="youtube" %}

{% include video id="BV1hM4y1F7fV" provider="bilibili" danmaku="1" %}

## 规划

| CPU | 内存 | 角色   | 内网 IP    | 外网 IP |
| --- | ---- | ------ | ---------- | ------- |
| 4   | 24   | server | 10.0.0.154 |         |
| 1   | 1    | agent  | 10.0.0.130 |         |
| 1   | 1    | agent  | 10.0.0.23  |         |

## 前置条件

### Oracle VM 实例

- 系统选择 Ubuntu 20.04.3 LTS
- VCN(虚拟云网络)入站规则需开放 22（SSH）、80（HTTP）、443（HTTPS）端口

### 主机防火墙配置

| 协议 | 端口      | 源                       | 描述                         |
| ---- | --------- | ------------------------ | ---------------------------- |
| TCP  | 22        | K3s server 和 agent 节点 | ssh 连接，22 系统默认已开启  |
| TCP  | 80        | k3s server 节点          | http 访问端口                |
| TCP  | 443       | k3s server 节点          | https 安全访问               |
| TCP  | 6443      | K3s agent 节点           | Kubernetes API Server        |
| UDP  | 8472      | K3s server 和 agent 节点 | 仅对 Flannel VXLAN 需要      |
| TCP  | 10250     | K3s server 和 agent 节点 | Kubelet metrics              |
| TCP  | 2379-2380 | K3s server 节点          | 只有嵌入式 etcd 高可用才需要 |

通常情况下，所有出站流量都是允许的。

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 6443/tcp
sudo ufw allow 8472/udp
sudo ufw allow 10250/tcp
sudo ufw allow 2379/tcp
sudo ufw allow 2380/tcp
```

### 域名（可选）

申请好域名，可以到[freenom](https://www.freenom.com)申请免费域名，配置 DNS 指向 Master 节点的公网 IP，我这里把 DNS 转到[cloudflare](https://www.cloudflare.com)了，用起来比较方便，功能也很多。

### K3S 安装要求

K3s 非常轻巧，但有一些最低要求，两个节点不能有相同的主机名。如果所有节点都有相同的主机名，使用--with-node-id 选项为每个节点添加一个随机后缀，或者添加到集群的每个节点设计一个独特的名称，用--node-name 或$K3S_NODE_NAME 传递。具体安装要求看[官方文档](https://docs.rancher.cn/docs/k3s/installation/installation-requirements/_index/)

## 安装

### 安装 k3s server 节点

主节点这里修改了 k3s 的版本号

```bash
curl -fsSL https://get.k3s.io | INSTALL_K3S_VERSION=v1.21.8+k3s1 \
sh -s - server \
--write-kubeconfig-mode 644
```

查看 server 节点的 k3s_token，复制出来安装 agent 节点会用到

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

### 安装 agent 节点

k3s_url 是 server 节点的 ip 地址加 6443 端口，k3s_token 用 server 节点的 node-token，两个 agent 节点都执行以下操作

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.21.8+k3s1 \
K3S_URL=https://10.0.0.154:6443 \
K3S_TOKEN=<k3s_token> sh -
```

### 验证 K3S 集群

```bash
# 主节点执行以下命令查看节点情况，节点状态均为ready则安装成功
kubectl get nodes -owide
```

### 安装 NFS

nfs 主要是用来在集群中做持久存储，支持多节点读写，配合 PV、PVC 工作，本文主要用来存放静态网站文件，这里是在 server 节点上安装

```bash
# 安装nfs server
sudo apt install nfs-kernel-server
# 检查nfs-server是否启动
sudo systemctl status nfs-server

# 创建共享目录
sudo mkdir -p /nfs/k3s
# 修改权限，这里开放最高权限
sudo chown nobody:nogroup /nfs/k3s
sudo chmod -R 777 /nfs/k3s

# 修改/etc/exports 文件，末尾添加 /nfs/k3s 10.0.0.*(rw,sync,no_subtree_check,no_root_squash)

sudo echo "/nfs/k3s *(rw,sync,no_subtree_check,no_root_squash)" >>  /etc/exports
# 使目录生效
sudo exportfs -arv

# 查看共享目录
showmount -e 10.0.0.154
```

可参考这篇[文章](https://blog.51cto.com/u_13510521/2811925)

### 创建 PV

在 NFS 服务的基础上在 k3s 集群中创建一个 PV 存储卷，可以看到下面的文件中指定了 nfs 的路径、IP 以及访问模式，用余存放静态网站文件，这里我放一个 html 文件，提一下，这里也可以用 configmap 来做

```bash
# 定义PV资源文件
cat <<EOF > nfs-pv1.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv1
  labels:
    type: nfs-pv

spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  storageClassName: nfs
  nfs:
    path: "/nfs/k3s/pv1"
    server: 10.0.0.154
    readOnly: false
EOF

#执行创建PV1
kubectl create -f nfs-pv1.yaml

#查看PV 状态
kubectl get pv
```

### 创建 PVC

PVC 可以理解为对一个存储卷的使用申请

```bash
cat <<EOF > nfs-pvc1.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc1
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 200Mi
  storageClassName: nfs
EOF

# 创建pvc1
kubectl create -f nfs-pvc1.yaml

# 查看pvc1状态
kubectl get pvc
```

### 安装 cert-manager（可选）

主要用来自动签发及管理 TLS 证书，如果不需要 https 安全访问可以跳过，我这里是在主节点安装，因为是 ARM 架构，故需要替换版本，注意：cert-manager 如果使用 let‘s encrypt 签发证书，需要域至主机节点 80、443 端口通畅

```bash
# 下载脚本并替换成arm对应版本
curl -sL https://github.com/jetstack/cert-manager/releases/download/v1.0.4/cert-manager.yaml |sed -r 's/(image:.*):(v.*)$/\1-arm:\2/g' > cert-manager-arm.yaml

# 安装cert-manager 默认会安装在cert-manager名称空间下
kubectl create -f cert-manager-arm.yaml

# 查看pod 状态 均为running则成功
kubectl get pods -n cert-manager

# 定义issuer文件
cat <<EOF > letsencrypt-prod-issuer.yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    # 填写你的email
    email: user@example.com
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-prod
    # Enable the HTTP-01 challenge provider
    solvers:
    # An empty 'selector' means that this solver matches all domains
    - selector: {}
      http01:
        ingress: {}
EOF

# 部署issuer
kubectl create -f letsencrypt-prod-issuer.yaml
```

### 签发证书

以下<your_domain> 包括括号需要替换为对应的域名

```bash
cat <<EOF > test-cert.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: <your_domain>
  namespace: default
spec:
  secretName: <your_domain>-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  commonName: <your_domain>
  dnsNames:
  - <your_domain>
EOF

# 创建证书
kubectl create -f test.cert.yaml

# 查看证书状态 对应的ready 为true则签发成功，如下
xxx.xx.com        True    xxx.xx.com-tls    15s
```

## 部署网站

到这里就可以正式部署起来一个静态网站了，结合前面创建好的 pvc 及证书就可以跑起来一个 https 的网站了。用 k8s 还是老规矩，先上资源文件，主要用了一个 nginx 来提供 http 服务，我的网页文件放在 nfs 里面，nginx 挂载了一个 pvc 的 nfs 目录，对外的服务请求到的文件就是 nfs 里面的文件

```bash
# 创建html文件   这里也可以去下载一个模版网页
echo "hello k8s" > index.html
# 复制网站文件到nfs中
cp index.html /nfs/k3s/pv1/
# 或者用下载的模版
wget https://www.free-css.com/assets/files/free-css-templates/download/page270/univers.zip
unzip univers.zip
mv univers/* /nfs/k3s/pv1/
# 注意一定要替换<your_domain>为对应的域名
cat <<EOF > demosite.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demoside-nginx
  labels:
    app: demoside-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demoside-nginx
  template:
    metadata:
      labels:
        app: demoside-nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html-volume
          mountPath: /usr/share/nginx/html
      volumes:
      - name: html-volume
        persistentVolumeClaim:
          claimName: nfs-pvc1
---
apiVersion: v1
kind: Service
metadata:
  name: demoside-nginx-service
spec:
  selector:
    app: demoside-nginx
  ports:
    - protocol: TCP
      port: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demoside-traefik-ingress
  namespace: default
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    kubernetes.io/ingress.class: traefik
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
spec:
  rules:
  - host: <your_domain>
    http:
      paths:
      - backend:
          service:
            name: demoside-nginx-service
            port:
              number: 80
        path: /
        pathType: Prefix
  tls:
  - hosts:
    - <your_domain>
    secretName: <your_domain>-tls
---
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: redirect

spec:
  redirectScheme:
    scheme: https
    permanent: true

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demoside-redirect
  namespace: default
  annotations:
    kubernetes.io/ingress.class: traefik
    traefik.ingress.kubernetes.io/router.entrypoints: web
    traefik.ingress.kubernetes.io/router.middlewares: default-redirect@kubernetescrd
spec:
  rules:
  - host: <your_domain>
    http:
      paths:
      - backend:
          service:
            name: demoside-nginx-service
            port:
              number: 80
        path: /
        pathType: Prefix
EOF

#执行创建
kubectl create -f demosite.yaml

# 查看pod状态 running 可以打开浏览器访问对应的域名不出意外的话就可以看到hellow k8s
kubectl get pods -o wide

```

文中提到的所有文件可以在[这个仓库](https://github.com/justfuny/oc-k3s)找到。

## EOF
