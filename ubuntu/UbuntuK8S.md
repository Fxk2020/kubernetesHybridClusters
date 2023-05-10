# 在Ubuntu安装K8S集群(docker版本：19.03  K8S版本：1.17.3)
## 一.安装docker（参考：https://blog.51cto.com/u_64214/6166141）
1.先卸载docker 防止已经安装过docker
```text
sudo apt-get remove docker docker-engine docker.io containerd runc
```
2.安装依赖
```
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
```
3.导入阿里云证书
```text
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
```
4.设置阿里云稳定仓库
```text
sudo add-apt-repository "deb [arch=amd64] https://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
```
5.更新仓库 （其实就是在 /etc/apt/sources.list 加了docker源）
```text
apt update
```
6.查看当前仓库中docker都有那些版本
```text
apt-cache madison docker-ce
```
7.安装指定版本docker版本，需要根据仓库中已有的版本及名称进行
```text
apt install docker-ce=5:19.03.12~3-0~ubuntu-bionic docker-ce-cli=5:19.03.12~3-0~ubuntu-bionic
```
8.检查是否安装成功
```text
docker --version
```
9.启动docker命令
```text
sudo systemctl start docker
```

**********
## 二.基于docker安装K8S（参考：https://zhuanlan.zhihu.com/p/111422247）
0.关闭swap
```text
sudo swapoff -a
```
1.docker 加速器及 cgroup 配置
```text
sudo mkdir -p /etc/docker
```
```text
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "registry-mirrors": [
    "https://dockerhub.azk8s.cn",
    "https://hub-mirror.c.163.com"
  ]
}
EOF
```
```text
sudo systemctl daemon-reload
sudo systemctl restart docker
```
2.关闭 dnsmasq 服务
```text
# 打开配置文件
sudo vi /etc/NetworkManager/NetworkManager.conf

[main]
plugins=ifupdown,keyfile,ofono
#dns=dnsmasq  # 这行注释

# 重启服务
sudo systemctl restart network-manager
```
3.安装 kubeadm, kubelet and kubectl三大组件
3.1.安装依赖，加载key，添加源
```text
# 安装依赖
sudo apt-get update && apt-get install -y apt-transport-https
# 加载key
sudo curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - 
# 添加源
sudo cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
```
3.2.更新仓库
```text
sudo apt-get update
```
3.3查看仓库kubelet，kubeadm，kubectl版本
```text
apt-cache madison kubelet
apt-cache madison kubeadm
apt-cache madison kubectl
```
3.4.正式安装三大组件
```text
sudo apt-get install kubelet=1.17.3-00 kubeadm=1.17.3-00 kubectl=1.17.3-00
```
4.检测是否安装成功
```text
kubelet --version
```
5.进行初始化主节点的操作/作为子节点进行加入到其他主节点的操作

**********
## 注：
一.如果机器A是某个K8S集群的子节点，若想将该节点脱离该集群，加入到其他K8S集群中怎样做：
1.在master节点查看所有的node节点
```text
kubectl get node
```
2.在master节点上操作删除node02
```text
kubectl delete node node02
```
3.在子节点上运行，在该节点上清空集群信息
```text
kubeadm reset
```
4.再返回master节点生成token值并且输出加入集群的命令
```text
[root@master ~]# kubeadm token create --print-join-command
W0206 11:04:15.278475   29635 validation.go:28] Cannot validate kubelet config - no validator is available
W0206 11:04:15.358432   29635 validation.go:28] Cannot validate kube-proxy config - no validator is available
kubeadm join 192.168.124.146:6443 --token beugsu.9qaksadu3lz5mo7t     --discovery-token-ca-cert-hash sha256:3394bd74680568f0d97ad67b42411b7b49eb6773f19de9efcd5a0e2d92d3447f 
```
5.复制上面命令输出的第三行在node02节点上操作加入集群即可。