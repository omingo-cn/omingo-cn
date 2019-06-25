---
title: k8s安装指南
date: 2019-06-24 15:33:49
categories:
tags: k8s
---

## 准备机器

官方要求:
* ubuntu16.04+
* 内存≥2G
* cpu≥2
* 机器间网络互通
* 每个节点的hostname,mac地址,product_uuid要唯一.
* swap要禁用.

### 规划

ip | 角色 | hostname| 配置| 系统
---------|----------|---------|--------|----
 192.168.1.36 | master | node1 | 2G/2C|ubuntu18.04
 192.168.1.37 | node | node2 | 2G/2C|ubuntu18.04
 192.168.1.38 | node | node3 | 2G/2C|ubuntu18.04


我使用的是vbox,网络模式使用桥接.安装一个完一个虚拟机之后,复制除另外两个,复制的时候选中"重新初始化所有网卡的MAC地址"
{% asset_img 1.png %}

### 修改主机名称(所有节点)
```bash
sudo sed -i '/preserve_hostname: false/c\preserve_hostname: true' /etc/cloud/cloud.cfg && sudo hostnamectl set-hostname {新hostname}
```
退出重新登录后hostname就会改变.
参考:[修改ubuntu18.04hostname](https://askubuntu.com/questions/1028633/host-name-reverts-to-old-name-after-reboot-in-18-04-lts)

### 修改为静态地址(所有节点)
1. 执行命令`sudo vim /etc/netplan/50-cloud-init.yaml`
2. 更改配置
  ```yml
  network:
      ethernets:
          enp0s3:
              addresses: [192.168.1.37/24] # 静态ip
              gateway4: 192.168.1.1 # 网关
              nameservers:
                      addresses: [192.168.1.254] #DNS
      version: 2
  ```
3. 使设置生效`sudo netplan apply`

参考 [ubuntu 18.04 netplan yaml配置固定IP地址](https://ywnz.com/linuxjc/1491.html)
### 关闭swap(所有节点)
`sudo swapoff -a`
`sudo vim /etc/fstab` 注释掉 swap那一行,如下:
```
UUID=f229223e-9634-11e9-9470-080027aa5c7f / ext4 defaults 0 0
#/swap.img	none	swap	sw	0	0  
```

### 检查 mac地址,product_uuid(所有节点)
```bash
ip link # 查看mac
sudo cat /sys/class/dmi/id/product_uuid # 查看product_uuid
```
一般情况下不会冲突.(反正我没遇到这种情况☺)

## 安装docker(所有节点)
国内网络环境,你懂得.
使用阿里云提供的方案快速安装: `curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun`
[Docker CE 镜像源站](https://yq.aliyun.com/articles/110806)

请自行配置docker镜像加速.

## 安装kubeadm, kubelet and kubectl(所有节点)
安装k8s需要安装这三个包:

名称 | 作用 
---------|----------
 kubeadm | 用来引导集群 
 kubelet | 在群集中的所有计算机上运行的组件，并执行诸如启动pod和容器之类的操作
 kubectl | 用来和集群通信的命令行工具

国内网络环境,这回应该懂了吧...

```bash
# 使用root操作
sudo su 
apt-get update && apt-get install -y apt-transport-https
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - 
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF  
apt-get update
apt-get install -y kubelet kubeadm kubectl
```

### 准备镜像(所有节点)

集群初始化的时候需要从k8s.gcr.io拉取镜像,但是国内网络环境,你懂得.
1. `kubeadm config images list` 查看使用到的镜像
```
kubeadm config images list
k8s.gcr.io/kube-apiserver:v1.15.0
k8s.gcr.io/kube-controller-manager:v1.15.0
k8s.gcr.io/kube-scheduler:v1.15.0
k8s.gcr.io/kube-proxy:v1.15.0
k8s.gcr.io/pause:3.1
k8s.gcr.io/etcd:3.3.10
k8s.gcr.io/coredns:1.3.1
```
2. 手动获取一下镜像
```
#上面那一堆复制下来
images=(
    k8s.gcr.io/kube-apiserver:v1.15.0
    k8s.gcr.io/kube-controller-manager:v1.15.0
    k8s.gcr.io/kube-scheduler:v1.15.0
    k8s.gcr.io/kube-proxy:v1.15.0
    k8s.gcr.io/pause:3.1
    k8s.gcr.io/etcd:3.3.10
    k8s.gcr.io/coredns:1.3.1
)
for imageName in ${images[@]} ; do
    imageName=${imageName##*/}
    docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
    docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName
    docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
done 
```
## 初始化集群(master节点)

```
# 因为使用flannel网络插件,需要在kubeadm init 时设置 --pod-network-cidr=10.244.0.0/16
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```
## 配置授权信息(master节点)
集群初始化后有提示
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
## 添加网络插件(master节点)
执行`kubectl get pods -A` 会发现coredns状态为pending.以为还没有安装网络插件,所有和网络相关的pod都会为pending.
集群初始化完成后也有提示`You should now deploy a pod network to the cluster.`
添加网络插件:
```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
稍等片刻,再次执行 `kubectl get pods -A` ,所有POD 都running了.

## 设置master节点也可以运行Pod
kubernetes官方默认策略是worker节点运行Pod，master节点不运行Pod。如果只是为了开发或者其他目的而需要部署单节点集群，可以通过以下的命令设置
`kubectl taint nodes --all node-role.kubernetes.io/master-`

## 加入节点(worker节点)
集群初始化完成后也有提示:
```
Then you can join any number of worker nodes by running the following on each as root:
kubeadm join 192.168.1.36:6443 --token 9wujqe.xfc5msp9l3i9g9s8 \
    --discovery-token-ca-cert-hash sha256:1c67699dbf329cceff693a37a6b3f2c4d706901673343a24c872d802a7ed3433
```

## 测试一下
在master执行:
```
kubectl get nodes 
#此时应该有三个节点
NAME    STATUS   ROLES    AGE    VERSION
node1   Ready    master   36m    v1.15.0
node2   Ready    <none>   8m2s   v1.15.0
node3   Ready    <none>   29s    v1.15.0
```

创建个deployment试试
```
kubectl run my-nginx --image=nginx --replicas=3 --port=80
# 稍等片刻
kubectl get pods -o wide
# 应该这样的
NAME                        READY   STATUS    RESTARTS   AGE   IP           NODE    NOMINATED NODE   READINESS GATES
my-nginx-756fb87568-ssxj5   1/1     Running   0          94s   10.244.0.4   node1   <none>           <none>
my-nginx-756fb87568-x2mtk   1/1     Running   0          94s   10.244.2.2   node3   <none>           <none>
my-nginx-756fb87568-zkjxj   1/1     Running   0          94s   10.244.1.2   node2   <none>           <none>
```
创建个service试试
```
kubectl expose deployment my-nginx --name=my-nginx --port 80 --target-port 80 --external-ip 192.168.1.36
```
打开浏览器访问一下 192.168.1.36,nginx首页.
{% asset_img 2.png %}
可以愉快的玩耍了.