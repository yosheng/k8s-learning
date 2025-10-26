
# 第一章：基于Containerd的k8s集群部署



#### Kubernetes介绍

![1677739343172](kubeadm部署k8s1.27单节点集群(containerd).accets/1677739343172.png)

kubernetes（k8s）是2015年由Google公司基于Go语言编写的一款开源的容器集群编排系统，用于自动化容器的部署、扩缩容和管理；

kubernetes（k8s）是基于Google内部的Borg系统的特征开发的一个版本，集成了Borg系统大部分优势；

官方地址：https://Kubernetes.io

代码托管平台：https://github.com/Kubernetes



#### kubernetes具备的功能

- 自我修复：k8s可以监控容器的运行状况，并在发现容器出现异常时自动重启故障实例；
- 弹性伸缩：k8s可以根据资源的使用情况自动地调整容器的副本数。例如，在高峰时段，k8s可以自动增加容器的副本数以应对更多的流量；而在低峰时段，k8s可以减少应用的副本数，节省资源；
- 资源限额：k8s允许指定每个容器所需的CPU和内存资源，能够更好的管理容器的资源使用量；
- 滚动升级：k8s可以在不中断服务的情况下滚动升级应用版本，确保在整个过程中仍有足够的实例在提供服务；
- 负载均衡：k8s可以根据应用的负载情况自动分配流量，确保各个实例之间的负载均衡，避免某些实例过载导致的性能下降；
- 服务发现：k8s可以自动发现应用的实例，并为它们分配一个统一的访问地址。这样，用户只需要知道这个统一的地址，就可以访问到应用的任意实例，而无需关心具体的实例信息；
- 存储管理：k8s可以自动管理应用的存储资源，为应用提供持久化的数据存储。这样，在应用实例发生变化时，用户数据仍能保持一致，确保数据的持久性；
- 密钥与配置管理：Kubernetes 允许你存储和管理敏感信息，例如：密码、令牌、证书、ssh密钥等信息进行统一管理，并共享给多个容器复用；



#### kubernetes集群角色

k8s集群需要建⽴在多个节点上，将多个节点组建成一个集群，然后进⾏统⼀管理，但是在k8s集群内部，这些节点⼜被划分成了两类⻆⾊：

- Master（管理节点）：负责集群的所有管理工作，和协调集群中运行的容器应用； 

- Node（工作节点）：负责运行集群中所有用户的容器应用， 执行实际的工作负载 ； 



**Master管理节点组件：**

- API Server：作为集群的控制中心，处理外部和内部通信，接收用户请求并处理集群内部组件之间的通信；
- Scheduler：负责将待部署的 Pods 分配到合适的 Node 节点上，根据资源需求、策略和约束等因素进行调度；
- Controller Manager：管理集群中的各种控制器，例如 Deployment、ReplicaSet、Node 控制器等，管理集群中的各种资源；
- etcd：作为集群的数据存储，保存集群的配置信息和状态信息；



**Node工作节点组件：**

- Kubelet：负责与 Master 节点通信，并根据 Master 节点的调度决策来创建、更新和删除 Pod，同时维护 Node 节点上的容器状态；
- 容器运行时（如 Docker、containerd 等）：负责运行和管理容器，提供容器生命周期管理功能。例如：创建、更新、删除容器等；
- Kube-proxy：负责为集群内的服务实现网络代理和负载均衡，确保服务的访问性；



**非必须的集群插件：**

- DNS服务：严格意义上的必须插件，在k8s中，很多功能都需要用到DNS服务，例如：服务发现、负载均衡、有状态应用的访问等；
- Dashboard： 是k8s集群的Web管理界面；
- 资源监控：例如metrics-server监视器，用于监控集群中资源利用率；

 

#### kubernetes集群类型

- 一主多从集群：由一台Master管理节点和多台Node工作节点组成，生产环境下Master节点存在单点故障的风险，适合学习和测试环境使用；

- 多主多从集群：由多台Master管理节点和多Node工作节点组成，安全性高，适合生产环境使用；



#### 主机硬件配置说明

> 提示：系统尽量别带图形界面，图形比较吃内存。

| 主机名   | IP地址       | 角色     | 系统个环境 | 硬件配置            |
| -------- | ------------ | -------- | ---------- | ------------------- |
| master01 | 192.168.0.10 | 管理节点 | CentOS 7.6 | 2CPU/4G内存/50G存储 |
| node01   | 192.168.0.11 | 管理节点 | CentOS 7.6 | 2CPU/4G内存/50G存储 |
| node02   | 192.168.0.12 | 管理节点 | CentOS 7.6 | 2CPU/4G内存/50G存储 |



#### 集群前期环境准备

修改每个节点主机名

~~~powershell
hostnamectl set-hostname master01
hostnamectl set-hostname node01
hostnamectl set-hostname node02
~~~



 **以下前期环境准备需要在所有节点都执行** 

配置集群之间本地解析，集群在初始化时需要能够解析主机名

```sh
echo "192.168.0.10 master01" >> /etc/hosts
echo "192.168.0.11 node01" >> /etc/hosts
echo "192.168.0.12 node02" >> /etc/hosts
```



**开启bridge网桥过滤功能**

bridge(桥接网络) 是 Linux 系统中的一种虚拟网络设备，它充当一个虚拟的网桥，为集群内的容器提供网络通信功能，容器就可以通过这个 bridge 与其他容器或外部网络通信了

~~~powershell
cat > /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

模块介绍
net.bridge.bridge-nf-call-ip6tables = 1  //对网桥上的IPv6数据包通过iptables处理
net.bridge.bridge-nf-call-iptables = 1   //对网桥上的IPv4数据包通过iptables处理
net.ipv4.ip_forward = 1       //开启IPv4路由转发,来实现集群中的容器与外部网络的通信
~~~

~~~powershell
#由于开启bridge功能，需要加载br_netfilter模块来允许在bridge设备上的数据包经过iptables防火墙处理
modprobe br_netfilter && lsmod | grep br_netfilter

#...会输出以下内容
br_netfilter           22256  0
bridge                151336  1 br_netfilter

命令及模块介绍：
modprobe        //命令可以加载内核模块
br_netfilter    //该模块允许在bridge设备上的数据包经过iptables防火墙处理
~~~

~~~powershell
#加载配置文件，使上述配置生效
sysctl -p /etc/sysctl.d/k8s.conf
~~~



**配置ipvs功能**

在k8s中Service有两种代理模式，一种是基于iptables的，一种是基于ipvs，两者对比ipvs负载均衡算法更加的灵活，且带有健康检查的功能，如果想要使用ipvs代理模式，需要手动载入ipvs模块。

`ipset` 和 `ipvsadm`  是两个与网络管理和负载均衡相关的软件包，在k8s代理模式中，提供多种负载均衡算法，如轮询（Round Robin）、最小连接（Least Connection）和加权最小连接（Weighted Least Connection）等；

~~~powershell
yum -y install ipset ipvsadm
~~~



将需要加载的ipvs相关模块写入到文件中

```powershell
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack
EOF

模块介绍
ip_vs         //提供负载均衡的模块,支持多种负载均衡算法,如轮询、最小连接、加权最小连接等
ip_vs_rr      //轮询算法的模块（默认算法）
ip_vs_wrr     //加权轮询算法的模块,根据后端服务器的权重值转发请求
ip_vs_sh      //哈希算法的模块,同一客户端的请求始终被分发到相同的后端服务器,保证会话一致性
nf_conntrack  //链接跟踪的模块,用于跟踪一个连接的状态,例如 TCP 握手、数据传输和连接关闭等
```



执行文件来加载模块

~~~powershell
chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack
~~~



**关闭SWAP分区**

为了保证 kubelet 正常工作，k8s强制要求禁用，否则集群初始化失败

```sh
#临时关闭
swapoff -a

#永久关闭
sed -ri 's/.*swap.*/#&/' /etc/fstab
grep ".*swap.*" /etc/fstab
```



检查swap

```sh
free -h
...
Swap:            0B          0B          0B
```



#### Containerd环境准备

添加containerd的yum仓库

```sh
tee /etc/yum.repos.d/containerd.repo <<EOF
[containerd]
name=containerd
baseurl=https://download.docker.com/linux/centos/7/\$basearch/stable
enabled=1
gpgcheck=1
gpgkey=https://download.docker.com/linux/centos/gpg
EOF
```



安装containerd

```sh
yum install containerd.io-1.6.20-3.1.el7.x86_64 -y
```



生成containerd配置文件

```sh
containerd config default | tee /etc/containerd/config.toml
```



启用Cgroup控制组，用于限制进程的资源使用量，如CPU、内存等

~~~powershell
sed -i 's#SystemdCgroup = false#SystemdCgroup = true#' /etc/containerd/config.toml
~~~



替换文件中pause镜像的下载地址

```sh
sed -i 's#sandbox_image = "registry.k8s.io/pause:3.6"#sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.6"#' /etc/containerd/config.toml
```



在k8s环境中，kubelet通过 `containerd.sock` 文件与 containerd 进行通信，对容器进行管理，指定contaienrd套接字文件地址 

```powershell
cat <<EOF | tee /etc/crictl.yaml
runtime-endpoint: unix:///var/run/containerd/containerd.sock
image-endpoint: unix:///var/run/containerd/containerd.sock
timeout: 10
debug: false
EOF

#参数说明
runtime-endpoint	//指定了容器运行时的sock文件位置
image-endpoint		//指定了容器镜像使用的sock文件位置
timeout				//容器运行时或容器镜像服务之间的通信超时时间
debug				//指定了crictl工具的调试模式,false表示调试模式未启用,true则会在输出中包含更多的调试日志信息，有助于故障排除和问题调试
```



启动containerd

```sh
systemctl daemon-reload && systemctl start containerd && systemctl enable containerd && systemctl status containerd
```



#### kubeadm部署k8s集群

kubernetes集群有多种部署方式，目前常用的部署方式有如下两种：

- kubeadm部署方式：kubeadm是一个快速搭建kubernetes的集群工具；
- 二进制包部署方式：从官网下载每个组件的二进制包，依次去安装，部署麻烦；
- 其他方式：通过一些开源的工具搭建，例如：sealos；



通过Kubeadm方式部署k8s集群，需要配置k8s的yum仓库来安装集群所需软件，本实验使用阿里云YUM源

~~~powershell
cat > /etc/yum.repos.d/k8s.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
~~~



安装集群软件，本实验安装k8s 1.27.0版本

- kubeadm：用于初始化集群，并配置集群所需的组件并生成对应的安全证书和令牌；
- kubelet：负责与 Master 节点通信，并根据 Master 节点的调度决策来创建、更新和删除 Pod，同时维护 Node 节点上的容器状态；
- kubectl：用于管理k8集群的一个命令行工具；

~~~powershell
yum -y install kubeadm-1.27.0 kubelet-1.27.0 kubectl-1.27.0
~~~



配置kubelet的Cgroup，用于限制进程的资源使用量，如CPU、内存等

~~~powershell
tee > /etc/sysconfig/kubelet <<EOF
KUBELET_EXTRA_ARGS="--cgroup-driver=systemd"
EOF
~~~



设置kubelet开机自启动即可，集群初始化后会随着集群启动

```sh
systemctl enable kubelet
```



查看集群所需镜像文件

```sh
kubeadm config images list
```



#### k8s集群初始化

在master01节点初始化集群，需要创建集群初始化配置文件

~~~powershell
[root@master01 ~]# kubeadm config print init-defaults > kubeadm-config.yaml
~~~



配置文件需要修改如下内容

```powershell
[root@master01 ~]# cat kubeadm-config.yaml

#...以下是需要修改的内容

#本机的IP地址
advertiseAddress: 192.168.0.10

#本机名称
name: master01

#集群镜像下载地址，修改为阿里云
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers

#集群版本
kubernetesVersion: 1.27.0
```



初始化集群

```powershell
[root@master01 ~]# kubeadm init --config kubeadm-config.yaml
```

```powershell
提示：如果哪个节点出现问题，可以使用下列命令重置当前节点
kubeadm  reset
```



根据提示执行如下命令

~~~powershell
[root@master01 ~]# mkdir -p $HOME/.kube
[root@master01 ~]# cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
[root@master01 ~]# chown $(id -u):$(id -g) $HOME/.kube/config
[root@master01 ~]# export KUBECONFIG=/etc/kubernetes/admin.conf
~~~



#### 节点加入集群

按照自己当前集群生成的token在node01、node02节点执行，加入到集群，加入集群后，在master节点验证

```sh
[root@master01 ~]# kubectl get node
master   NotReady   control-plane,master   
node01   NotReady   <none>                 
node02   NotReady   <none>                 
```



#### 部署Calico网络

Calico 和 Flannel 是两种流行的 k8s 网络插件，它们都为集群中的 Pod 提供网络功能。然而，它们在实现方式和功能上有一些重要区别： 



**网络模型的区别：**

- Calico 使用 BGP（边界网关协议）作为其底层网络模型。它利用 BGP 为每个 Pod 分配一个唯一的 IP 地址，并在集群内部进行路由。[Calico 支持网络策略，可以对流量进行精细控制，允许或拒绝特定的通信]()。
- Flannel 则采用了一个简化的覆盖网络模型。它为每个节点分配一个 IP 地址子网，然后在这些子网之间建立覆盖网络。Flannel 将 Pod 的数据包封装到一个更大的网络数据包中，并在节点之间进行转发。[Flannel 更注重简单和易用性，不提供与 Calico 类似的网络策略功能]()。



在master01节点安装Calico网络即可

~~~powershell
#下载calico文件
[root@master01 ~]# wget https://raw.githubusercontent.com/projectcalico/calico/v3.24.1/manifests/calico.yaml

#创建calico网络
[root@master01 ~]# kubectl apply -f calico.yaml 
~~~



查看kube-system空间的calico的Pod状态，等待Pod状态为Running

```powershell
[root@master01 ~]# kubectl get pod -n kube-system
NAME                                       READY   STATUS
calico-kube-controllers-6799f5f4b4-q4jr8   1/1     Running
calico-node-4kmw2                          1/1     Running
calico-node-c8lg4                          1/1     Running
calico-node-j5cts                          1/1     Running
coredns-7f74c56694-c26qc                   1/1     Running
coredns-7f74c56694-m7xct                   1/1     Running
etcd-master01                              1/1     Running
kube-apiserver-master01                    1/1     Running
kube-controller-manager-master01           1/1     Running
kube-proxy-8mxmf                           1/1     Running
kube-proxy-bqqp6                           1/1     Running
kube-proxy-zlxs2                           1/1     Running
kube-scheduler-master01                    1/1     Running
```



#### 集群验证

查看集群节点

```powershell
kubectl get node
NAME       STATUS   ROLES           AGE   VERSION
master01   Ready    control-plane   53m   v1.27.0
node01     Ready    <none>          50m   v1.27.0
node02     Ready    <none>          50m   v1.27.0
```



#### 部署Nginx程序测试

需要注意的是，containerd 相比于docker , 多了namespace概念, 每个image都会在各自的namespace下可见, 目前k8s会使用k8s.io 作为命名空间；

```powershell
ctr ns ls
NAME   LABELS
k8s.io
```



在k8s中管理镜像和容器可以使用crictl命令更加方便，ctr是containerd自带的CLI命令行工具，crictl是k8s中CRI（容器运行时接口）的客户端，k8s使用该客户端和containerd进行交互

```sh
#node01查看镜像
crictl images
...
docker.io/library/choujiang_ngx      v1

#node02插件镜像
crictl images
...
docker.io/library/choujiang_ngx      v1
```



创建一个nginx的pod（在master01节点执行命令）

```powershell
kubectl create deployment nginx --image=docker.io/library/nginx:1.20.0
```



暴露Pod端口供外部网络访问

```powershell
kubectl expose deployment nginx --port=80 --type=NodePort
```



查看Pod状态

```powershell
[root@master ~]# kubectl get pod
NAME                     READY   STATUS    
nginx-696649f6f9-j8zbj   1/1     Running     
```



查看Pod端口

```powershell
[root@master01 ~]# kubectl get service
NAME         TYPE        CLUSTER-IP       PORT(S)   
kubernetes   ClusterIP   10.96.0.1        443/TCP   
nginx        NodePort    10.103.195.31    80:32554/TCP  
```



浏览器访问测试：[http://集群任意主机IP:32554]()



#### kuboard k8s多集群管理平台

参考地址：https://kuboard.cn/install/v3/install-built-in.html#%E9%83%A8%E7%BD%B2%E8%AE%A1%E5%88%92

> 在K8S集群之外准备一台主机安装docker，并通过docker安装kuboard

```sh
[root@kuboard ~]# yum -y install yum-utils

[root@kuboard ~]# yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

[root@kuboard ~]# yum -y install docker-ce-20.10.9-3.el7

[root@kuboard ~]# systemctl enable --now docker 
```



安装kuboard（http://本机IP）

```sh
docker run -d \
  --restart=unless-stopped \
  --name=kuboard \
  -p 80:80/tcp \
  -p 10081:10081/tcp \
  -e KUBOARD_ENDPOINT="http://192.168.0.30:80" \
  -e KUBOARD_AGENT_SERVER_TCP_PORT="10081" \
  -v /root/kuboard-data:/data \
  eipwork/kuboard:v3
```



访问kuboard页面：[http://本机IP]()

- 用户名：admin
- 密码：Kuboard123        #大写的K

