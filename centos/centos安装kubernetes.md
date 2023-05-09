# centos安装kubernetes

### 1.版本号

```
centos 7
docker 19.03.9
kubernetes 1.17.3
```

### 2.前期准备

关闭防火墙：

```
systemctl stop firewalld
systemctl disable firewalld
```

关闭selinux:

```
sed -i 's/enforcing/disabled/' /etc/selinux/config
setenforce 0
```

关闭swap:

```
swapoff -a 临时
sed -ri 's/.*swap.*/#&/' /etc/fstab 永久
free -g 验证，swap 必须为0
```

将桥接的IPv4 流量传递到iptables 的链：

```
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sysctl --system
```

### 3.安装docker

卸载系统之前的docker

```
sudo yum remove docker \
docker-client \
docker-client-latest \
docker-common \
docker-latest \
docker-latest-logrotate \
docker-logrotate \
docker-engine
```

安装Docker-CE

```
sudo yum install -y yum-utils device-mapper-persistent-data lvm2

sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

yum install docker-ce-19.03.9-3.el7 -y
systemctl start docker
systemctl enable docker
```

配置docker加速

```
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
"registry-mirrors": ["https://82m9ar63.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

### 4.安装kube核心组件

添加阿里云yum 源

```
cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

安装kubeadm，kubelet和kubectl

```
yum list|grep kube
yum install -y kubelet-1.17.3 kubeadm-1.17.3 kubectl-1.17.3
systemctl enable kubelet
systemctl start kubelet
```

### 5. master节点初始化

master节点初始化

```
kubeadm init --apiserver-advertise-address=10.0.100.122 --image-repository registry.cn-hangzhou.aliyuncs.com/google_containers --kubernetes-version v1.17.3 --service-cidr=10.96.0.0/16 --pod-network-cidr=10.244.0.0/16
```

10.0.100.122改成master节点的IP地址

![image-20230509193835020](https://oss-img-fxk.oss-cn-beijing.aliyuncs.com/markdown/image-20230509193835020.png)

这条指令一定要保存下来根据这个加入节点。



测试kubectl(主节点执行)

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl get nodes //获取所有节点
```

目前master 状态为notready。等待网络加入完成即可。

![image-20230509194011079](https://oss-img-fxk.oss-cn-beijing.aliyuncs.com/markdown/image-20230509194011079.png)

安装Pod 网络插件（CNI）

```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kubeflannel.yml
```

这个文件需要科学上网，可以下载下来后传到主机上进行操作

等待大约3分钟，master的状态即可变为ready

![image-20230509194358363](https://oss-img-fxk.oss-cn-beijing.aliyuncs.com/markdown/image-20230509194358363.png)

在Node 节点执行。

向集群添加新节点，执行在kubeadm init 输出的kubeadm join 命令