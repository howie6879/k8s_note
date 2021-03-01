# 安装

## 什么是Kubernetes

随着微服务架构被越来越多的公司使用，大部分单体应用正逐步被拆解成小的、独立运行的微服务。微服务的优势这里不做探讨，但是其带来的服务维护问题大大增加，若想要在管理大量微服务的情况下还需要让资源利用率更多且硬件成本相对更低，那么基于容器部署的微服务的一些自动化设施的需求就这样诞生了，于是就有了Kubernetes，其提供的特性有：

- **服务发现和负载均衡**
- **存储编排**
- **自动发布和回滚**
- **自愈**
- **密钥及配置管理**

## 安装

### 单机安装

关于单机安装`k8s`，我使用的相关环境如下：

- macOS：Catalina 10.15.1
- Docker Desktop Vesion：3.0.2
- Kubernetes：1.19.3

由于镜像的下载涉及到网络原因，因此这里使用了开源项目[k8s-docker-desktop-for-mac](https://github.com/gotok8s/k8s-docker-desktop-for-mac)来解决这个问题，需要注意的是要修改`images`的相关镜像的版本，要和此时`Kubernetes`配对上才行，比如我设置的是：

```txt
k8s.gcr.io/kube-proxy:v1.19.3=gotok8s/kube-proxy:v1.19.3
k8s.gcr.io/kube-controller-manager:v1.19.3=gotok8s/kube-controller-manager:v1.19.3
k8s.gcr.io/kube-scheduler:v1.19.3=gotok8s/kube-scheduler:v1.19.3
k8s.gcr.io/kube-apiserver:v1.19.3=gotok8s/kube-apiserver:v1.19.3
k8s.gcr.io/coredns:1.7.0=gotok8s/coredns:1.7.0
k8s.gcr.io/pause:3.2=gotok8s/pause:3.2
k8s.gcr.io/etcd:3.4.13-0=gotok8s/etcd:3.4.13-0
```

随后打开`Docker`，进入设置界面，勾选`Enable Kubernetes`即可：

![image-20201220195307360](https://gitee.com/howie6879/oss/raw/master/uPic/image-20201220195307360.png)

不出意外，界面左下角会出现`Kubernetes running`的提示，这样就安装成功了。

如果状态一直处于`Kubernetes starting`状态，可在终端执行以下命令然后重启`Docker`：

```shell
rm -rf ~/.kube
rm -rf ~/Library/Group\ Containers/group.com.docker/pki/
```

### 集群安装

#### 准备

- 准备三台机器，比如（使用的配置是4核8G，IP换成你自己的）：
  - 192.168.5.91：Master：
    - 执行：
      - `hostnamectl set-hostname master` 
      - `echo "127.0.0.1   $(hostname)" >> /etc/hosts`
  - 192.168.5.92：Node01
    - 执行：
      - `hostnamectl set-hostname node01` 
      - `echo "127.0.0.1   $(hostname)" >> /etc/hosts`
  - 192.168.5.93：Node02
    - 执行：
      - `hostnamectl set-hostname node02` 
      - `echo "127.0.0.1   $(hostname)" >> /etc/hosts`

- Kubernetes版本：v1.19.3
- Docker版本：19.03.12

开始前请检查以下事项：

- **CentOS 版本**：>= 7.6
- **CPU**：>=2
- *IP*：互通
- 关闭**swap**：`swapoff -a`

配置国内kubernetes源：

```shell
cat > /etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

安装相关依赖工具：

```shell
yum install -y kubelet-1.19.3 kubeadm-1.19.3 kubectl-1.19.3
# 设置开机启动
systemctl enable kubelet.service && systemctl start kubelet.service
# 查看状态
systemctl status kubelet.service
```



#### 初始化Master

在主节点（`192.168.5.91`）执行以下命令：

```sh
export MASTER_IP=192.168.5.91
export APISERVER_NAME=apiserver.demo
export POD_SUBNET=10.100.0.1/16
echo "${MASTER_IP}    ${APISERVER_NAME}" >> /etc/hosts
```

新建脚本`init_master.sh`:

```shell
vim init_master.sh
```

添加：

```bash
#!/bin/bash

# 只在 master 节点执行

# 脚本出错时终止执行
set -e

if [ ${#POD_SUBNET} -eq 0 ] || [ ${#APISERVER_NAME} -eq 0 ]; then
  echo -e "\033[31;1m请确保您已经设置了环境变量 POD_SUBNET 和 APISERVER_NAME \033[0m"
  echo 当前POD_SUBNET=$POD_SUBNET
  echo 当前APISERVER_NAME=$APISERVER_NAME
  exit 1
fi


# 查看完整配置选项 https://godoc.org/k8s.io/kubernetes/cmd/kubeadm/app/apis/kubeadm/v1beta2
rm -f ./kubeadm-config.yaml
cat <<EOF > ./kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
# k8s 版本
kubernetesVersion: v1.19.3
imageRepository: registry.aliyuncs.com/k8sxio
controlPlaneEndpoint: "${APISERVER_NAME}:6443"
networking:
  serviceSubnet: "10.96.0.0/16"
  podSubnet: "${POD_SUBNET}"
  dnsDomain: "cluster.local"
EOF

# kubeadm init
# 根据您服务器网速的情况，您需要等候 3 - 10 分钟
kubeadm config images pull --config=kubeadm-config.yaml
kubeadm init --config=kubeadm-config.yaml --upload-certs

# 配置 kubectl
rm -rf /root/.kube/
mkdir /root/.kube/
cp -i /etc/kubernetes/admin.conf /root/.kube/config

# 安装 calico 网络插件
# 参考文档 https://docs.projectcalico.org/v3.13/getting-started/kubernetes/self-managed-onprem/onpremises
echo "安装calico-3.13.1"
rm -f calico-3.13.1.yaml
wget https://kuboard.cn/install-script/calico/calico-3.13.1.yaml
kubectl apply -f calico-3.13.1.yaml
```

如果出错：

```shell
# issue 01
# [ERROR FileContent--proc-sys-net-bridge-bridge-nf-call-iptables]: /proc/sys/net/bridge/bridge-nf-call-iptables contents are not set to 1
# 所有机器执行
echo 1 > /proc/sys/net/bridge/bridge-nf-call-iptables
echo 1 > /proc/sys/net/bridge/bridge-nf-call-ip6tables
```



检查`master`初始化结果：

```shell
# 直到所有的容器组处于 Running 状态
watch kubectl get pod -n kube-system -o wide
# 查看 master 节点初始化结果
kubectl get nodes -o wide
```

如下图：

![image-20201221165311482](https://gitee.com/howie6879/oss/raw/master/uPic/image-20201221165311482.png)

#### 获得join命令参数

直接在`master`执行：

```shell
kubeadm token create --print-join-command
```

比如此时输出：

```shell
# 有效期两小时
kubeadm join apiserver.demo:6443 --token vh5hl9.9fccw1mzfsmsp4gh     --discovery-token-ca-cert-hash sha256:6970397fdc6de5020df76de950c9df96349ca119f127551d109430c114b06f40
```

#### 初始化Node

在所有`node`执行：

```shell
export MASTER_IP=192.168.5.91
export APISERVER_NAME=apiserver.demo
echo "${MASTER_IP}    ${APISERVER_NAME}" >> /etc/hosts

# 替换为 master 节点上 kubeadm token create 命令的输出
kubeadm join apiserver.demo:6443 --token vh5hl9.9fccw1mzfsmsp4gh     --discovery-token-ca-cert-hash sha256:6970397fdc6de5020df76de950c9df96349ca119f127551d109430c114b06f40
```

#### 检查初始化结果

在`master`节点执行：

```shell
kubectl get nodes -o wide
```

输出结果如下：

![image-20201221181016264](https://gitee.com/howie6879/oss/raw/master/uPic/image-20201221181016264.png)

### Kubernetes Dashboard

[Dashboard](https://github.com/kubernetes/dashboard)可以将容器化应用程序部署到`Kubernetes`集群，对容器化应用程序进行故障排除，以及管理集群资源。

#### 安装

安装命令如下：

```shell
wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.1.0/aio/deploy/recommended.yaml -O recommended.yaml

kubectl apply -f recommended.yaml
kubectl get pods --namespace=kubernetes-dashboard -o wide
```

此处执行完会发现`STATUS`都是`ContainerCreating`：

```shell
NAME                                         READY   STATUS    RESTARTS   AGE    IP          NODE             NOMINATED NODE   READINESS GATES
dashboard-metrics-scraper-79c5968bdc-bdz9d   1/1     `ContainerCreating`   0          100m   10.1.0.20   docker-desktop   <none>           <none>
kubernetes-dashboard-7448ffc97b-z8222        1/1     `ContainerCreating`   0          100m   10.1.0.19   docker-desktop   <none>           <none>
```

查看日志找找原因（注意NAME）：

```shell
kubectl describe pod dashboard-metrics-scraper-79c5968bdc-bdz9d --namespace=kubernetes-dashboard
```

发现是因为`metrics-scraper:v1.0.6`镜像下载不下来，手动执行：

```shell
docker pull kubernetesui/metrics-scraper:v1.0.1
```

拉下来之后就妥了。

最后我们需要将访问形式改为`NodePort`访问：

```shell
kubectl --namespace=kubernetes-dashboard edit service kubernetes-dashboard
# 将里面的 type: ClusterIP 改为 type: NodePort
```

保存后，执行：

```shell
kubectl --namespace=kubernetes-dashboard get service kubernetes-dashboard
```

终端输出：

```shell
NAME                   TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
kubernetes-dashboard   NodePort   10.98.194.124   <none>        443:31213/TCP   107m
```

#### Token

在浏览器访问：`https://0.0.0.0:31213/`:

![image-20201220201125802](https://gitee.com/howie6879/oss/raw/master/uPic/image-20201220201125802.png)

看界面需要生成`Token`：

```shell
vim admin-user.yaml
# 输入
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
# 保存退出
vim admin-user-role-binding.yaml
# 输入
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
# 保存退出

# 执行命令加载配置
kubectl create -f admin-user.yaml
kubectl create -f admin-user-role-binding.yaml

# 若出现已存在
# 执行：kubectl delete -f xxx.yaml 即可
```

获取令牌：

```shell
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
```

复制`token`到刚才的界面登录即可，登录后界面如下：

![image-20201220201512486](https://gitee.com/howie6879/oss/raw/master/uPic/image-20201220201512486.png)

如果想延长`Token`的有效时间：

![image-20201221204143993](https://gitee.com/howie6879/oss/raw/master/uPic/image-20201221204143993.png)

然后在`containners->args`加上`--token-ttl=43200`。

### 部署镜像

下拉一个你自己想部署的镜像，具体命令如下（主节点执行）：

```shell
# 部署
kubectl run hello --image=xxx/hello --port=5000
# 列出 pod
kubectl get pods
# 创建一个服务对象
# NodePort 在所有节点（虚拟机）上开放一个特定端口，任何发送到该端口的流量都被转发到对应服务
kubectl expose po hello --port=5000 --target-port=5000 --type=NodePort  --name hello-http
# 列出服务
kubectl get services
```

## 参考

本部分内容有参考如下文章：

- [使用kubeadm安装kubernetes_v1.19.x](https://kuboard.cn/install/install-k8s.html#%E6%A3%80%E6%9F%A5-centos-hostname)
- [Web基础配置篇（十六）: Kubernetes集群的安装使用](https://www.pomit.cn/p/2366402025269761#1010602)
- Kubernetes in Action中文版：第1、2章

