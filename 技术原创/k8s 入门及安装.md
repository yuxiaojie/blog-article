### Docker 的能力边界

容器就是一个沙盒技术，将不同的程序单独运行在一个独立的环境沙盒中，而且这个环境沙盒以及内部的程序可以很方便的进行转移，这就是 Docker 容器最大的优点。Docker 并没有使用任何全新的技术，而是使用一些原有的技术来控制这个沙盒的边界

#### 环境隔离

容器本质上也是宿主机系统中的一个进程，为了产生一个隔离的效果，Docker 使用了 `Namespace` 技术，这是 Linux 创建进程的时候的一个可选参数，当填入后，创建的进程所查询的系统所有进程，都是一个全新的进程空间，而自己的 PID 就是 1，但是实际在操作系统中，这个进程仍然是一个真实的 PID，只是这个新创的进程自己看不到。

类似的，Linux 还提供 `Mount`、`UTS`、`IPC`、 `Network` 和 `User` 这些的 `Namespace` 技术，效果也跟进程的 `Namespace` 技术类似

这个技术是大部分容器技术的基础，所以对比虚拟机真实的隔离环境，Docker 的隔离其实就是各种 `Namespace` 配置，本质仍然是在宿主机上运行的一个进程，这也体现了 Docker 的轻量级的优点，资源开销小，性能高，尤其是隔离方面基本没什么开销，但这里的问题就是，隔离的不彻底，因为有部分的资源是不能 `Namespace` 化的，比如时间，如果会跟预期现象不同的是，一个容器修改后，宿主机包括所有的其他容器都会被修改

#### 资源限制

容器本质上也是一个宿主机上真实运行的进程，所以也会跟其他的进程进行竞争，所以也可能会抢占宿主机其他进程的资源，也可能会被抢占，这跟容器隔离的概念相悖。而 `Linux Cgroups` 技术就是用来为进程组设置资源限制的重要功能，通过这个技术可以实现限制某个容器的资源使用情况，但是这个也有一些问题，比如 `/proc` 这个路径下面存放的是宿主机真实的 CPU 或者内存等资源的使用情况，而不是当前容器的，所以这也会给开发者带来困惑

#### 文件系统

容器内部文件系统是独立的文件系统，看起来可以任意修改，这个实现其实也是基于 `Mount Namespace` 这个技术，在创建容器进程之前，Docker 就为用户自动的配置了 `Namespace` 以及 `Cgroups` 参数，实际上就是有一个目录 mount 到容器的根目录，然后一般会在这个根目录下挂载一个完整操作系统的文件系统，然后切换进程的根目录(Change Root)，这个时候运行的容器进程看到的，就是一个完整的独立的文件系统了

这里需要明确，这个挂载的只是一个文件系统，但是操作系统的内核仍然是当前系统的内核，所以所有的容器都是共享宿主机的操作系统内核的，而所有涉及内核参数的修改，都相当于修改了所有容器的全局变量一样，这个区别于虚拟机的完全独立的操作系统内核的

Docker 还有一个创新在于，镜像是分层的，上一层镜像对底层镜像的修改，并不会真实的修改，而是暂时的，这里实际是使用的 Linux 的联合挂载到统一的一个挂载点上，所以实现了每层镜像都是一个增量，当上层镜像对下层镜像做了修改，实际也是在本层镜像以增量的形式做了修改然后覆盖了底层的镜像文件，所以底层镜像文件实际上并没有被修改。而删除文件也是类似的，实际就是在本层镜像创建了一个 `whiteout` 文件，这个文件被联合挂载之后，就会覆盖底层的文件并且不显示，在用户看来就跟被删除了一样

Docker 镜像完整的打包了镜像制作者配置的完整的文件系统，镜像使用者 pull 得到的内容完全一致，可以完全复现这个镜像制作者当初的完整环境，这就是容器非常大的优点  -  **`强一致性`**

####  总结

Docker 在 `开发 - 测试 - 发布` 这个流程中，实现了信息强一致性传递，解决了令人头疼的环境问题，但是 Docker 本质上仍是只是一个工具，在这个链路上传递的是 Docker Image，而这整个链路的服务节点，都因为容器技术带来新的变化，此时为了解决传递过程的一系列问题，就又引入了容器编排

需要理解：

> 容器本身没有价值，有价值的是“容器编排”



### Kubernetes 的优势

在 Docker 兴起之后，容器编排领域也出现了很多的方案，比较有代表性的就是 Docker 公司的 `Compose+Swarm` 组合以及 Google 与 RedHat 公司共同主导的 `Kubernetes` 项目

容器编排能提供路由网关、水平扩展、监控、备份、灾难恢复等 一系列运维能力，除此之外，`Kubernetes` 主要旨在解决各种容器与容器之间的各种关系的处理，为此 `Kubernetes` 对服务类型(依靠关系的总结)进行了抽象实现:

- 需要被部署同一台机器的可以定义为 `Pod` , 单个`Pod` 共享同一个`Network Namespace` 以及 同一组数据卷，从而可以做到高速数据交互
- 数据库服务可以被定义为 `Pod` ，再为这个 `Pod` 绑定一个 `Service` 服务，`Service` 服务可以声明 IP 等不变的信息，所以 `Service` 主要用于作为 `Pod` 的代理入口，从而代替 `Pod` 对外暴露一个固定的网络地址
- 如果需要一次性启动和管理多个应用的实例，可以定义一个 `Deployment` 的多实例管理器
- 如果有一次性的任务，可以将任务定义为 `Job`
- 如果有定时任务，可以将任务定义为 `CronJob`
- 如果有每个宿主机上必须且只能运行一个副本 的守护进程服务，可以定义为 `DaemonSet`

在 `Kubernetes` 的设计中，推荐先通过一个编排对象来描述你的应用，然后定义一些服务对象来提供一些平台级的功能，这就是所谓的 `声明式 API`



### k8s 集群安装

#### 环境要求



##### 主机：

目前使用 3 台虚拟机搭建 k8s 集群，后续测试没问题再部署至物理机上，单机资源 CPU 不少于双核，内存最好大于 8G (影响能调度的pod数量)，30G 及以上的磁盘空间，



###### 关闭防火墙

集群之间的服务是需要有一些开放的端口的，测试部署时直接关闭了防火墙

阿里云主机默认也是关闭防火墙的而使用自带的安全组，如果是自己的机房那根据自己的安全策略修改，而需要开放的端口可以参考 [开启机器上的某些端口](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#check-required-ports)

```shell
firewall-cmd --state									# 查看防火墙状态
systemctl stop firewalld 							# 停止 firewall
systemctl disable firewalld					 	# 禁止 firewall 开机启动
```



###### 关闭 SELinux

```shell
/usr/sbin/sestatus -v      # 如果SELinux status参数为enabled即为开启状态

# 修改/etc/selinux/config 文件, 将 SELINUX=enforcing 改为 SELINUX=disabled , 重启机器即可
```



###### 禁用交换分区

官方文档要求，为保证 kubelet 正常工作，必须要禁用交换分区

```shell
# 删除 swap 区所有内容
swapoff -a

# 注释 swap 行
vim /etc/fstab

# 重启系统，测试
reboot
free -h

# swap 一行应该全部是 0
              total        used        free      shared  buff/cache   available
Mem:           3.1G        192M        2.6G        8.9M        309M        2.7G
Swap:            0B          0B          0B
```



###### 配置内核参数

```shell
cat > /etc/sysctl.d/k8s.conf <<EOF  
net.bridge.bridge-nf-call-ip6tables = 1  
net.bridge.bridge-nf-call-iptables = 1  
net.ipv4.ip_forward = 1  
EOF  
  
sysctl --system
```





##### 操作系统

x86 或者 ARM 架构的都可以，但是因为依赖 Docker，所以操作系统必须是 64位操作系统，3.10及以上的 Linux 内核



##### 网络

部署是需要自动联网拉取组件的镜像的，所以主机都需要能连接外网，然后主机之间网络要能互通，这是搭建集群的前提



##### Docker 

如果没有安装 Docker 的话，需要先安装下 Docker，如果 Docker 版本很低也可以卸载后安装新版本

> 注意，测试的时候我是安装最新版的 Docker 版本，生产环境的 Docker 版本应该选择一个比较稳定的版本，因为往往最新版本的 Docker 跟 k8s 可能会有一定的兼容问题需要需要打磨



###### 卸载旧版本 Docker

```shell
sudo yum remove docker \
                docker-client \
                docker-client-latest \
                docker-common \
                docker-latest \
                docker-latest-logrotate \
                docker-logrotate \
                docker-selinux \
                docker-engine-selinux \
                docker-engine
```



###### 安装新版本 Docker

```shell
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install -y docker-ce
```



###### 配置docker开机自启动并启动docker

```shell
systemctl enable docker
systemctl daemon-reload
systemctl start docker
```



###### 安装完毕可以查询安装的docker版本

```shell
[root@t1 ~]# docker --version
Docker version 19.03.9, build 9d988398e7
```



##### Docker Registry 加速

直接访问 DockerHub 拉取速度比较慢，可以配置一个国内的镜像加速器

```shell
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
    "registry-mirrors": [
        "https://1nj0zren.mirror.aliyuncs.com",
        "https://docker.mirrors.ustc.edu.cn",
        "http://f1361db2.m.daocloud.io",
        "https://registry.docker-cn.com"
    ]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```



#### 安装 kubeadm、kubelet 和 kubectl

官网使用的是 `packages.cloud.google.com` 链接但是很慢，所以替换为国内源

```shell
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

yum install -y kubelet kubeadm kubectl –disableexcludes=kubernetes

systemctl enable kubelet
systemctl daemon-reload
systemctl restart kubelet
```



#### 配置 Master 节点



#####一键配置 Master 节点

`kubeadm` 可以一键配置 Master 节点，但是对于需要每个组件都有很多配置选项可以支持自定义配置，这里是通过 `kubeadm`  参数来进行配置，主要是修改了 api 服务器的地址，也就是 Master 节点的内网ip，还有就是各种组件的镜像地址替换为国内源

```shell
kubeadm init --kubernetes-version=1.18.2 \
  --apiserver-advertise-address=10.211.55.7 \
  --image-repository registry.aliyuncs.com/google_containers \
  --service-cidr=10.1.0.0/16 \
  --pod-network-cidr=10.244.0.0/16 \
  --ignore-preflight-errors=Swap
W0527 16:55:11.365656   12682 configset.go:202] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
[init] Using Kubernetes version: v1.18.2
[preflight] Running pre-flight checks
	[WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [t1 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.1.0.1 10.211.55.7]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [t1 localhost] and IPs [10.211.55.7 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [t1 localhost] and IPs [10.211.55.7 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
W0527 16:57:49.302991   12682 manifests.go:225] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[control-plane] Creating static Pod manifest for "kube-scheduler"
W0527 16:57:49.303915   12682 manifests.go:225] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 21.003692 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.18" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node t1 as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node t1 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: kmsy23.4dlci17ysyp2y6sc
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.211.55.7:6443 --token kmsy23.4dlci17ysyp2y6sc \
    --discovery-token-ca-cert-hash sha256:7330ea011681579494c1640668955f5aeaf1e0f60e9e3cb3455f8a7e7c9b38c7
[root@t1 ~]#
```



完成整个部署之后，`kubeadm` 最后面会打印一行指令：

```shell
kubeadm join 10.211.55.7:6443 --token kmsy23.4dlci17ysyp2y6sc \
    --discovery-token-ca-cert-hash sha256:7330ea011681579494c1640668955f5aeaf1e0f60e9e3cb3455f8a7e7c9b38c7
```

这个命令的主要作用就是，让 worker 节点加入集群使用的



##### 配置授权信息

k8s 集群默认需要加密方式访问，所以需要将刚刚部署生成的 Kubernetes 集群的安全配置文件，保存到当前用户的.kube 目录下，kubectl 默认会使用这个目录下的授权信息访问 Kubernetes 集群

```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```



现在我们可以通过 `kubectl get` 查看节点的状态，可以看到节点的状态是 NotReady，这是因为我们尚未部署任何网络插件

```shell
[root@t1 ~]# kubectl get nodes
NAME   STATUS     ROLES    AGE   VERSION
t1     NotReady   master   17m   v1.18.3
```



##### 安装网络插件

Kubernetes 支持容器网络插件，使用的是一个名叫 CNI 的通用接口，它也是当前容器网络的事实标准，市面上的所有容器网络开源项目都可以通过 CNI 接入 Kubernetes，比如 Flannel、Calico、 Canal、Romana 等等，这里使用的是 Weave 作为网络插件

```shell
[root@t1 ~]# kubectl apply -f
"https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"

# 安装完需要等一会，可以通过以下命令看到 weave 插件已经是 Running 状态
[root@t1 ~]# kubectl get pods --namespace=kube-system
NAME                         READY   STATUS    RESTARTS   AGE
coredns-7ff77c879f-46wls     1/1     Running   0          20h
coredns-7ff77c879f-kjfdg     1/1     Running   0          20h
etcd-t1                      1/1     Running   0          20h
kube-apiserver-t1            1/1     Running   0          20h
kube-controller-manager-t1   1/1     Running   3          20h
kube-proxy-9lxwq             1/1     Running   0          20h
kube-scheduler-t1            1/1     Running   4          20h
weave-net-mdmb8              2/2     Running   0          3m1s

# 再查看每个节点的状态，此时已经变为 Ready
[root@t1 ~]# kubectl get nodes
NAME   STATUS   ROLES    AGE   VERSION
t1     Ready    master   20h   v1.18.3
```



#### 配置 Worker 节点

相比较于配置 Master 节点，Worker 节点配置异常轻松，完成 k8s 需要的环境准备以及安装 kubeadm、kubelet 和 kubectl，然后将配置 Master 节点最后打印的 join 命令执行一下即可，完成后在 Master 节点查看各个节点即可以查看各个节点的状态

```shell
[root@t1 ~]# kubectl get nodes
NAME   STATUS   ROLES    AGE     VERSION
t1     Ready    master   21h     v1.18.3
t2     Ready    <none>   8m14s   v1.18.3
t3     Ready    <none>   8m9s    v1.18.3
```



#### 配置 Dashboard 

```shell
# 安装 Dashboard 
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml

# 创建一个登陆 Dashboard 的账号
[root@t1 ~]# kubectl create serviceaccount dashboard-admin -n kube-system
serviceaccount/dashboard-admin created
[root@t1 ~]# kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin
clusterrolebinding.rbac.authorization.k8s.io/dashboard-admin created

# 获取登陆 Dashboard 需要的 token
[root@t1 ~]# kubectl describe secrets -n kube-system $(kubectl -n kube-system get secret | awk '/dashboard-admin/{print $1}')
Name:         dashboard-admin-token-mb786
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: dashboard-admin
              kubernetes.io/service-account.uid: 2c14417e-83da-4673-816e-eef63d6a5c5c

Type:  kubernetes.io/service-account-token

Data
====
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IkR4d3RGM2x6RTdNLXJfNVJCdVk0REtYeTNSZG55dUctVGJydHQ5VXFfOHMifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtYWRtaW4tdG9rZW4tbWI3ODYiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGFzaGJvYXJkLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiMmMxNDQxN2UtODNkYS00NjczLTgxNmUtZWVmNjNkNmE1YzVjIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOmRhc2hib2FyZC1hZG1pbiJ9.onRmNHu6SODeGaPxYFidXsYvP1NKm9FJ0Ozws_k3Vs4sVLG6i631c5OinsRV_p0ia67lm2H_yy71cnuUrkfuiLAUAFr74Cr9jKKUK_w7lc0YvKGlBv8diy1v3w1wwLpX4SW7bnYDkPENZO8KyifCSrH_YMkNo-CEfmYqQY3zVm89PiSOzTlFNgT455Mv6_AODZlOndD4_VjxExIRQAJKWyul1dsdSABxK2l3cfBXXeBI3fhCyt1LMGXImCUALRXln_k7Lqb9ZrHlnssfA8TbL3-vAcnD0PFwYmg6QY3xFJFN0Pv3Yy4dhykzBjIvEo1Ljt401rjSHB53nACqUFsSKw
ca.crt:     1025 bytes
namespace:  11 bytes

```



对于访问 Dashboard 可以有三种方式

- 通过 kubectl proxy 访问 dashboard
- 通过 API server 访问 dashboard（https 6443端口和http 8080端口方式）
- kubernetes-dashboard 服务暴露了 NodePort，可以使用 `http://NodeIP:nodePort` 地址访问 dashboard



##### 通过 kubectl proxy 访问 dashboard

```shell
[root@t1 tmp]# kubectl proxy --address='0.0.0.0' --port=8086 --accept-hosts='^*$'
Starting to serve on [::]:8086
```

启动后浏览器访问 `http://{master_ip}:8086/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/login` , `{master_ip}` 就是可以 Master 节点可以被本地访问的ip地址，注意开放端口能访问



##### 通过 API server 访问 dashboard

直接浏览器通过访问 `https://{master_ip}:6443/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/login` 来进行访问，这里有个问题就是需要在本地安装证书，在网上查资料，可以由一下命令在 Master 节点生成客户端可用的证书

```shell
grep 'client-certificate-data' ~/.kube/config | head -n 1 | awk '{print $2}' | base64 -d >> kubecfg.crt
grep 'client-key-data' ~/.kube/config | head -n 1 | awk '{print $2}' | base64 -d >> kubecfg.key
openssl pkcs12 -export -clcerts -inkey kubecfg.key -in kubecfg.crt -out kubecfg.p12 -name "kubernetes-client"
```

生成证书需要密码，可以直接一直回车表示不设置，如果设置了，客户端导入证书的时候需要输入

生成证书后，客户端吧证书下载到本地之后，可以双击导入证书，并在证书中进行授信，授信后访问前面的地址时，选择刚安装的证书，然后就可以正常访问了



##### kubernetes-dashboard 服务暴露了 NodePort

在安装 Dashboard 时，配置修改 NodePort 并开放，所以跟  kubectl proxy 有点类似，所以可以直接通过这个端口来访问，但是这个方法我没有测试



配置没问题之后，访问 dashboard 会要求输入 token，将之前获取的 token 输入即可进入主页面

![](https://user-gold-cdn.xitu.io/2020/6/2/1727441fef36dca9?w=2456&h=1610&f=png&s=198079)

![](https://user-gold-cdn.xitu.io/2020/6/2/17274423110c8c87?w=3262&h=1482&f=png&s=289372)

#### 配置容器存储插件

容器存储插件会在容器中插入一个基于网络或者其他机制的数据卷，这使得在这个数据卷中保存的数据，实际是存储在远端服务器中，以达到持久化的目的，这里根据教程，选择安装插件 Rook，推荐的原因在于 Rook 巧妙地依赖了 Kubernetes 提供的编排能力，合理的使用了很多诸如 Operator、CRD 等重要的扩展特性

```shell
git clone https://github.com/rook/rook.git
cd rook/cluster/examples/kubernetes/ceph
kubectl apply -f common.yaml
kubectl apply -f operator.yaml
kubectl apply -f cluster.yaml

[root@t1 ceph]# kubectl -n rook-ceph get pods
NAME                                            READY   STATUS              RESTARTS   AGE
csi-cephfsplugin-provisioner-7459667b67-f9q5z   0/5     ContainerCreating   0          2m44s
csi-cephfsplugin-provisioner-7459667b67-p6nxr   0/5     ContainerCreating   0          2m44s
csi-cephfsplugin-rppth                          3/3     Running             0          2m44s
csi-cephfsplugin-s5rjs                          0/3     ContainerCreating   0          2m44s
csi-rbdplugin-p4mzl                             0/3     ContainerCreating   0          2m45s
csi-rbdplugin-pkdh8                             3/3     Running             0          2m45s
csi-rbdplugin-provisioner-7cbb778994-fs2tl      0/6     ContainerCreating   0          2m45s
csi-rbdplugin-provisioner-7cbb778994-h8zdc      0/6     ContainerCreating   0          2m45s
rook-ceph-mon-a-canary-86c69f8675-tccv5         1/1     Running             0          44s
rook-ceph-mon-b-canary-7894bddc7b-5hd8m         1/1     Running             0          43s
rook-ceph-mon-c-canary-95585b665-8prv2          0/1     Pending             0          43s
rook-ceph-operator-567d7945d6-qrbns             1/1     Running             0          9m54s
rook-discover-7r72h                             1/1     Running             0          6m41s
rook-discover-zsp4l                             1/1     Running             0          6m41s
```








