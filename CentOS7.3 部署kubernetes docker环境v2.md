# **CentOS7.3 部署kubernetes docker环境**
## **1. node节点**
###  **1.1 服务器准备**
```shell
10.2.1.236  master236
10.2.1.231  node231
10.2.1.232  node232
10.2.1.237  node237
```
### **1.2 所有节点均执行的基本配置**
* 关闭防火墙
```shell
systemctl stop firewalld
systemctl disable firewalld
```
* 关闭selinux
```shell
setenforce 0
sed -i 's/enforcing/disabled/' /etc/selinux/config
```
* 关闭swap
```shell
swapoff -a
```
* 进入vim /etc/fstab文件夹
```shell
注释掉 /dev/mapper/system-swap swap 这一整行
```
* 修改hosts
```shell
vim /etc/hosts
  添加以下几行
  10.2.1.236  master236 centos-236.shared
  10.2.1.231  node231 centos-231.shared
  10.2.1.232  node232 centos-232.shared
```
* *注：后续如有新的node添加进来，用相同格式将IP及主机名添加进来即可，master结点hosts也需要同步更新*

* 修改主机名
```shell
hostnamectl set-hostname master236（改成hosts中本机IP对应的主机名）
```
* 卸载旧版本docker
```shell
yum remove docker*
```
* 重启
```shell
reboot
```

### **1.3 安装docker**
我们根据kubernetes 1.17.3，选择安装docker 18.06.3
* 安装基础环境，两条yum命令分开执行
```shell
yum -y install yum-utils 
yum -y install device-mapper-persistent-data lvm2 xfsprogs
```
* 更新docker ce源
```shell
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```
* 安装docker
```shell
yum -y install docker-ce-18.06.3.ce-3.el7  docker-ce-cli-18.06.3.ce-3.el7
```
* 设置开机自启并启动docker
```shell
systemctl enable docker && systemctl start docker
```
* 查看docker版本
```shell
docker version
```

### **1.4 安装kubernetes**
* 修改系统参数
```shell
cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
```
* 使之生效
```shell
sysctl -p /etc/sysctl.d/k8s.conf
```
* 关闭SWAP
```shell
swapoff -a
```
* 增加kubernetes源
```shell
vim /etc/yum.repos.d/k8s.repo
在里面添加
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
```
* 安装kubernetes
```shell
yum install -y kubelet-1.17.3 kubeadm-1.17.3 kubectl-1.17.3 ipvsadm
```
* 继续修改配置
```shell
vim /etc/sysconfig/kubelet
KUBELET_CGROUP_ARGS="--cgroup-driver=systemd"
KUBELET_EXTRA_ARGS="--fail-swap-on=false"
```
* 使之生效
```shell
sysctl --system
```
* 启动查看kubernetes
```shell
systemctl enable kubelet && systemctl start kubelet
```
* 查看kubernetes版本
```shell
kubeadm version
kubelet --version
```
### **1.5 node节点相关配置**
* node节点网络参数设置
```shell
mkdir /run/flannel
scp root@k8s-master:/run/flannel/subnet.env /run/flannel/
scp root@k8s-master:/etc/sysctl.d/k8s.conf /etc/sysctl.d/

modprobe br_netfilter
sysctl -p /etc/sysctl.d/k8s.conf
```
* 安装从master结点复制来的镜像k8s-node.tar
```shell
docker load --input k8s-node.tar
```
* 在master节点部署完成后，执行来自master结点初始化的命令，加入集群，node节点部署完成
```shell
kubeadm join 10.2.1.236:6443 --token pf0r8h.f4mp7bmwvs2nrx6s     --discovery-token-ca-cert-hash sha256:3d2fde48d29d8c12dee2e27b1a9e712a2d911e3bab667a86ec38016b7b116276
```

### **1.6 node节点重置**
如果有类似Pod节点地址重置这样的情况，就需要重置node节点
* 在node节点执行以下命令
```shell
kubeadm reset
ifconfig cni0 down
ip link delete cni0
ifconfig flannel.1 down
ip link delete flannel.1
rm -rf /var/lib/cni/
```
* 执行后，继续根据master节点生成的加入集群命令，添加到集群

##  **2. master节点**
### **2.1 master部署**
* 执行1.1~1.4的环境部署工作
* 创建脚本文件/etc/sysconfig/modules/ipvs.modules, 写入
```shell
#!/bin/bash
ipvs_mods_dir="/usr/lib/modules/$(uname -r)/kernel/net/netfilter/ipvs"
for i in $(ls $ipvs_mods_dir | grep -o "^[^.]*"); do
    /sbin/modinfo -F filename $i &> /dev/null
    if [ $? -eq 0 ]; then
        /sbin/modprobe $i
    fi
done
```
* 保存运行并检查
```shell
:wq
chmod +x /etc/sysconfig/modules/ipvs.modules
/etc/sysconfig/modules/ipvs.modules
lsmod | grep ip_vs
```
* 查看所需的镜像列表
```shell
kubeadm config images list
```
* 创建脚本vim pull-k8s.sh拉取镜像，脚本写入：
```shell
#!/bin/bash
images=(
    kube-apiserver:v1.17.3
    kube-controller-manager:v1.17.3
    kube-scheduler:v1.17.3
    kube-proxy:v1.17.3
    pause:3.1
    etcd:3.4.3-0
    coredns:1.6.5
)
for imageName in ${images[@]} ; do
    docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
    docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName
    docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
done
```
* 保存退出并执行
```shell
chmod +x pull-k8s.sh
./pull-k8s.sh
```
* 初始化集群，注意此处设置的pod网络(pod-network-cidr)应该与宿主机网段隔离开，否则会出现网络错误
```shell
kubeadm init \
--kubernetes-version=v1.17.3 \
--pod-network-cidr=10.244.0.0/16
```
* 为kubectl准备Kubeconfig文件
```shell
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```
* 部署Flannel网络插件
```shell
git clone --depth 1 https://github.com/coreos/flannel.git
cd flannel/Documentation/
kubectl create -f kube-flannel.yml
```
* 查看Pod状态，等待flannel Pod变为Running后，coredns才会变为running
```shell
coredns-6955765f44-hvkp9            1/1     Running   0          130m
coredns-6955765f44-vl5jd            1/1     Running   0          130m
etcd-master236                      1/1     Running   4          170m
kube-apiserver-master236            1/1     Running   4          170m
kube-controller-manager-master236   1/1     Running   4          170m
kube-flannel-ds-amd64-7vw8x         1/1     Running   0          117m
kube-flannel-ds-amd64-z6n94         1/1     Running   0          60m
kube-proxy-k9f2d                    1/1     Running   0          60m
kube-proxy-xvl4d                    1/1     Running   4          170m
kube-scheduler-master236            1/1     Running   5          170m
```
* 等所有Pod均为running状态，查看node
```shell
kubectl get node
```
* 为node结点初始化一个加入集群的命令
```shell
kubeadm token create --print-join-command
```