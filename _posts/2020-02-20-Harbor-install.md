---
layout: post
title:  "two ways to install Harbor"
categories: Linux
tags:  linux harbor 
author: Utachi
---

* content
{:toc}

# Harbor的部署 

Harbor是Docker Images Registry Service系统，用于管理容器镜像，由VMVare中国团队开发。
Harbor项目源码，[https://github.com/goharbor/harbor](https://github.com/goharbor/harbor)

## 1. 直接安装

参考[https://github.com/goharbor/harbor/tree/master/docs/1.10](https://github.com/goharbor/harbor/tree/master/docs/1.10)。

###  1.1 环境准备
```bash
#下载harbor离线包
wget  https://github.com/goharbor/harbor/releases/download/v1.10.1/harbor-offline-installer-v1.10.1.tgz 
tar zxvf harbor-offline-installer-v1.10.1.tgz 

#安装Docker
wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
yum -y install docker-ce-18.06.1.ce-3.el7
mkdir -p /etc/docker
tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://w9jw321y.mirror.aliyuncs.com"]
}
EOF
systemctl daemon-reload && systemctl restart docker && systemctl enable docker
docker info 

#载入harbor需要的离线images
cd harbor && docker load -i harbor.v1.10.1.tar.gz

#安装docker-compose
yum install -y python-devel python-pip 
pip install docker-compose -i https://pypi.doubanio.com/simple
```





###  1.2 自定义CA&SSL

* 生成证书颁发机构证书

```bash

#创建并切入CA操作目录
mkdir CA && cd CA    

#Generate a CA certificate private key
openssl genrsa -out ca.key 4096

#Generate the CA certificate
openssl req -x509 -new -nodes -sha512 -days 3650 \
 -subj "/C=CN/ST=Beijing/L=Beijing/O=utachi/OU=Personal/CN=harbor.utachi.cn" \
 -key ca.key \
 -out ca.crt
```
* 生成服务器证书

```bash
#Generate a CA certificate private key
openssl genrsa -out harbor.utachi.cn.key 4096

#Generate the CA certificate
openssl req -sha512 -new \
    -subj "/C=CN/ST=Beijing/L=Beijing/O=utachi/OU=Personal/CN=harbor.utachi.cn" \
    -key harbor.utachi.cn.key \
    -out harbor.utachi.cn.csr

#Generate an x509 v3 extension file.
#可以为Harbor主机生成符合主题备用名称（SAN）和x509 v3的证书扩展要求。替换DNS条目以反映该域
cat > v3.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1=harbor.utachi.cn
DNS.2=utachi
DNS.3=harbor    #主机名
EOF
#Use the v3.ext file to generate a certificate for Harbor host.
openssl x509 -req -sha512 -days 3650 \
    -extfile v3.ext \
    -CA ca.crt -CAkey ca.key -CAcreateserial \
    -in harbor.utachi.cn.csr \
    -out harbor.utachi.cn.crt
```

* 提供证书给Harbor和Docker

```bash
mkdir -p /data/cert/
cp harbor.utachi.cn.crt /data/cert/
cp harbor.utachi.cn.key /data/cert/


#Docker daemon 识别 .crt 文件为 CA certificates；识别 .cert 文件为 client certificates.
openssl x509 -inform PEM -in harbor.utachi.cn.crt -out harbor.utachi.cn.cert

mkdir -p /etc/docker/certs.d/harbor.utachi.cn/
cp harbor.utachi.cn.cert /etc/docker/certs.d/harbor.utachi.cn/
cp harbor.utachi.cn.key /etc/docker/certs.d/harbor.utachi.cn/
cp ca.crt /etc/docker/certs.d/harbor.utachi.cn/

#重启docker
systemctl restart docker 
```


###  1.3 部署harbor

* 返回harbor目录，配置yaml文件

```yaml
hostname: harbor.utachi.cn
...
  certificate: /etc/docker/certs.d/harbor.utachi.cn/harbor.utachi.cn.cert
  private_key: /etc/docker/certs.d/harbor.utachi.cn/harbor.utachi.cn.key
...
#默认的管理员账号密码也在这里配置
```
* 执行安装脚本

```bash
./prepare    #预安装scripts部署了一个nginx实例用于应用https
docker-compose down -v
docker-compose up -d
```


## 2. 通过Helm安装部署harbor 

###  2.1 安装helm


* 下载helm二进制包

 [https://storage.googleapis.com/kubernetes-helm/helm-v2.16.3-linux-amd64.tar.gz](https://storage.googleapis.com/kubernetes-helm/helm-v2.16.3-linux-amd64.tar.gz)


#### 2.1.1 安装helm客户端

```bash
tar -zxvf helm-v2.16.3-linux-amd64.tar.gz
mv linux-amd64/helm  /usr/local/bin/helm
helm help
```

#### 2.1.2 安装helm服务端

* RBAC创建tiller角色

```bash
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
```
* 初始化helm
 
```bash
helm init \
--service-account tiller \
--tiller-namespace=kube-system  \
--tiller-image=registry.cn-hangzhou.aliyuncs.com/google_containers/tiller:v2.16.3  \
--stable-repo-url=http://mirror.azure.cn/kubernetes/charts/  \
--upgrade

#初始化的yaml文件位置：~/.helm/repository/repositories.yaml
#若已经在k8s部署了tiller则只需要执行
helm init  --client-only
```
* 检查tiller部署结果
```bash
kubectl get pod -n kube-system -l app=helm
#显示：
#tiller-deploy-866f98f75d-rc9br          0/1     Pending   0          13m
#查看：kubectl  describe pod/tiller-deploy-866f98f75d-rc9br
#Events:
#Type     Reason            Age                 From               Message
#----     ------            ----                ----               -------
#Warning  FailedScheduling  53s (x13 over 17m)  default-scheduler  #0/1 nodes are available: 1 node(s) had taints that the pod didn't #tolerate.
#原因：默认master不参与计算pod的负载而标记为污点
#解除：kubectl taint nodes --all node-role.kubernetes.io/master-

```
* 检查helm 调用tille的结果
```bash
helm version
```
*  测试helm示例
```bash
#创建一个chart示例
helm create helm-chart
#检查chart语法
helm lint ./helm-chart
#使用默认chart部署到k8s
helm install --name helm-test1 ./helm-chart --set service.type=NodePort
#kubectl get pod 查看是否部署成功
```

### 2.3 部署Harbor-helm

* 下载helm部署代码并进入harbor helm目录

```bash
git clone https://github.com/goharbor/harbor-helm.git
cd harbor-helm

#更新helm dependency。 harbor的helm部署依赖postgresql
helm dependency update
```
#### 2.3.1 安装前准备

* 编辑values.yaml
* 
```bash
 ingress:
    hosts:
      core: harbo.utachi.io            #修改ingress里面的域名
      notary: harbo.utachi.io
……
externalURL: https://harbo.utachi.io   #修改访问的URL
……
  persistentVolumeClaim:               #persistentVolumeClaim:需要手动创建5个pv，官方默认为两个5G，3个1G，可根据需要修改size

……
```

* 初始化存储

```bash
#使用local-storage作为持久化存储，使用k8s-master机器的 /data/disks 目录作为本地持久化存储
mkdir /data/disks/{disk1-5G,disk2-5G,disk3-1G,disk4-1G,disk5-1G}

tee disk1-5G.yaml <<-'EOF'
apiVersion: v1
kind: PersistentVolume
metadata:
  name: disk1
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  local:
    path: "/data/disks/disk1-5G"
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - k8s-master       #存储所在节点主机名
EOF
#其他4个文件同理
kubectl apply -f disk*.yaml
kubectl get pv
```
*  helm install harbor

```bash
helm install . --debug --name harbo.utachi.io --set externalDomain=harbo.utachi.io
#externalDomain为外部能访问到harbor的域名，若无，此处可添加域名用于解析。
```

#### 2.3.2  helm-harbor自定义配置

* 如果有创建好的storageClass可以直接在values.yaml配置，如果没有或暂时不能使用storageClass的，可以使用NFS。
* 修改templates默认的namespace。注意需要修改依赖的chart文件夹下的postgresql-0.9-1.tgz压缩文件中的部署yaml模板，否则harbor的ui、register、mysql和postgresql不在一个namespace下，不能正常安装。
* 注意修改postgresql的持久化数据卷，可参考 [https://github.com/kubernetes/charts/tree/master/stable/postgresql](https://github.com/kubernetes/charts/tree/master/stable/postgresql)