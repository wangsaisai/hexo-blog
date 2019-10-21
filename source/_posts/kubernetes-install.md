---
title: 使用kubeadm创建kubernetes单主集群 - ubuntu
---


# use kubeadmin install k8s in ubuntu


### 准备环境
- 至少2台机器或虚拟机（一个master，多个worker）
- 每台机器 2 GB 内存或更多
- 每台机器 2 CPU 核心或更多
- 集群中的所有机器的网络彼此均能相互连接(公网和内网都可以)
- 禁用 Swap 交换分区

<!--more-->


### 安装软件
```
# 每台机器都需要安装
apt-get update && apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
```


### master node 初始化
```
# 选择一台作为 master
# 在 master 上执行
kubeadm init
```
执行成功后，会输出 token 。
其他worker 使用此 token 加入到 kubernetes 集群。
> Then you can join any number of worker nodes by running the following on each as root:
kubeadm join 10.176.7.9:6443 --token hh6cfg.ijmppt2fpry2dqr9 \
--discovery-token-ca-cert-hash sha256:c862aa8398c66300678371f1276cf6ffe02d589a17dc722c4eedb9ccd1db0282


### worker node 加入 kubernetes 集群
```
# 在worker 上执行
kubeadm join 10.176.7.9:6443 --token hh6cfg.ijmppt2fpry2dqr9 \
--discovery-token-ca-cert-hash sha256:c862aa8398c66300678371f1276cf6ffe02d589a17dc722c4eedb9ccd1db0282
```

*token 24小时后会过期，可重新获取token*
```
# 重新生成token
kubeadm token create
# 获取ca证书sha256编码hash值
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
# join到集群
kubeadm join 10.176.7.9:6443 --token {token} --discovery-token-ca-cert-hash sha256:{hash}
```


### 安装Pod网络插件 - weave
```
# 在 master 上执行
export kubever=$(kubectl version | base64 | tr -d '\n')
wget https://cloud.weave.works/k8s/net?k8s-version=$kubever -O weave.yaml
kubectl apply -f weave.yaml
```


### 验证是否安装成功
```
kubernetes@km:~$ kubectl get nodes
NAME STATUS ROLES AGE VERSION
km Ready master 58m v1.16.2
kw1 Ready <none> 41m v1.16.2
kw2 Ready <none> 39m v1.16.2
```
```
kubernetes@km:~$ kubectl get pods --all-namespaces -o wide
NAMESPACE NAME READY STATUS RESTARTS AGE IP NODE NOMINATED NODE READINESS GATES
kube-system coredns-5644d7b6d9-pfp4r 1/1 Running 0 58m 10.32.0.2 km <none> <none>
kube-system coredns-5644d7b6d9-w98kd 1/1 Running 0 58m 10.32.0.3 km <none> <none>
kube-system etcd-km 1/1 Running 0 57m 10.176.7.9 km <none> <none>
kube-system kube-apiserver-km 1/1 Running 0 57m 10.176.7.9 km <none> <none>
kube-system kube-controller-manager-km 1/1 Running 0 57m 10.176.7.9 km <none> <none>
kube-system kube-proxy-2nnzx 1/1 Running 0 42m 10.176.9.51 kw1 <none> <none>
kube-system kube-proxy-bxhf2 1/1 Running 0 58m 10.176.7.9 km <none> <none>
kube-system kube-proxy-mzqx6 1/1 Running 0 39m 10.176.12.157 kw2 <none> <none>
kube-system kube-scheduler-km 1/1 Running 0 57m 10.176.7.9 km <none> <none>
kube-system weave-net-fjzkw 2/2 Running 0 39m 10.176.12.157 kw2 <none> <none>
kube-system weave-net-gjpnn 2/2 Running 0 42m 10.176.9.51 kw1 <none> <none>
kube-system weave-net-pk682 2/2 Running 0 53m 10.176.7.9 km <none> <none>
```


### 使用普通用户访问kubernetes
```
# 在master节点执行
# 切换到个人用户
mkdir $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
然后将config文件内容拷贝至其他worker节点，或者个人PC里（$HOME/.kube/config），即可在worker以及个人PC上执行kubectl命令


### link
[https://kubernetes.io/zh/docs/setup/independent/install-kubeadm/](https://kubernetes.io/zh/docs/setup/independent/install-kubeadm/)
[https://kubernetes.io/zh/docs/setup/independent/create-cluster-kubeadm/](https://kubernetes.io/zh/docs/setup/independent/create-cluster-kubeadm/)
[https://blog.csdn.net/mailjoin/article/details/79686934](https://blog.csdn.net/mailjoin/article/details/79686934)
