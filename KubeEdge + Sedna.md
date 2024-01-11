# KubeEdge + Sedna 安装部署(By AdaYang)

## 说在前面

1.  k8s 只需要安装在 master 节点上，其他的节点都不用  
2.  kubeedge的运行前提是master上必须有k8s  
3.  docker只是用来发布容器pods的  
4.  **calico只需要安装在master上**，它是节点通信的插件，如果没有这个，master上安装kubeedge的coredns会报错。但是，节点上又不需要安装这个，因为kubeedge针对这个做了自己的通信机制  
5.  一些插件比如calico、edgemesh、sedna、metric-service还有kuborad等，都是通过yaml文件启动的，所以实际要下载的是**k8s的控制工具kubeadm和kubeedge的控制工具keadm**。然后提前准备好刚才的yaml文件，启动k8s和kubeedge后，直接根据yaml文件容器创建  
6.  namespace可以看作不同的虚拟项目，service是指定的任务目标文件，pods是根据service或者其他yaml文件创建的具体容器  
7.  一个物理节点上可以有很多个pods，pods是可操作的最小单位，一个service可以设置很多pods，一个service可以包含很多物理节点  
8.  一个pods可以看作一个根据docker镜像创建的实例  
9.  如果是主机容器创建任务，要设置dnsPolicy（很重要）  
10. 拉去 docker 镜像的时候，**一定要先去确认架构是否支持**

## Master 前期准备

### **关闭防火墙**

``` bash
ufw disable 
```

### **开启 ipv 4 转发 配置 iptables 参数**

将桥接的 `IPv4/IPv6` 流量传递到 iptables 的链

``` bash
modprobe br_netfilter
#方法1
cat >> /etc/sysctl.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sysctl -p   #执行此命令生效配置
```

### **禁用 swap 分区**

``` bash
swapoff -a                                          #临时关闭
sed -ri 's/.*swap.*/#&/' /etc/fstab                 #永久关闭
```

### **设置主机名**

不同虚拟机上设置

``` bash
hostnamectl set-hostname master    
bash

hostnamectl set-hostname node
bash

hostnamectl set-hostname node
bash
```

### **设置 DNS**

``` bash
#不同主机上使用ifconfig查看ip地址
sudo vim /etc/hosts

#根据自己的ip添加
#ip 主机名
192.168.247.128 master
192.168.247.129 node-1
```

### **安装 docker 并配置**

``` bash
sudo apt-get install docker.io -y

sudo systemctl start docker
sudo systemctl enable docker

docker --version

# 如果是 ARM64 架构
# 参考 https://blog.csdn.net/qq_34253926/article/details/121629068
```

一定要配置的

``` bash
sudo vim /etc/docker/daemon.json

#master添加：
{
  "registry-mirrors": ["https://b9pmyelo.mirror.aliyuncs.com"],
  "exec-opts": ["native.cgroupdriver=systemd"]
}

#node添加：
{
  "registry-mirrors": ["https://b9pmyelo.mirror.aliyuncs.com"],
  "exec-opts": ["native.cgroupdriver=cgroupfs"]
}

#都要执行
sudo systemctl daemon-reload
sudo systemctl restart docker

#查看修改后的docker Cgroup的参数
docker info | grep Cgroup
```

### **Master 节点安装 kubernetes**

#### 添加阿里源

``` bash
curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -
cat > /etc/apt/sources.list.d/kubernetes.list <<EOF
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
apt-get update
```

#### 安装相关组件

需要在每台机器上安装以下的软件包：
- `kubeadm`：用来初始化集群的指令。  
- `kubelet`：在集群中的每个节点上用来启动 Pod 和容器等。  
- `kubectl`：用来与集群通信的命令行工具。

``` bash
#查看软件版本
apt-cache madison kubeadm

#安装kubeadm、kubelet、kubectl
sudo apt-get update
sudo apt-get install -y kubelet=1.22.2-00 kubeadm=1.22.2-00 kubectl=1.22.2-00 
#锁定版本
sudo apt-mark hold kubelet kubeadm kubectl
```

``` bash
kubeadm init --pod-network-cidr 10.244.0.0/16 \
--image-repository registry.cn-hangzhou.aliyuncs.com/google_containers
```

#### 安装 calico

官方解释：因为在整个 kubernetes 集群里，pod 都是分布在不同的主机上的，为了实现这些 pod 的跨主机通信所以我们必须要安装 CNI 网络插件，这里选择 calico 网络。

**步骤1：在 master 上下载配置 calico 网络的 yaml。**

``` bash
#注意对应版本，v1.22和v3.20
wget https://docs.projectcalico.org/v3.20/manifests/calico.yaml --no-check-certificate
```

**步骤2：修改 calico.yaml 里的 pod 网段。**

``` bash
#把calico.yaml里pod所在网段改成kubeadm init时选项--pod-network-cidr所指定的网段，
#直接用vim编辑打开此文件查找192，按如下标记进行修改：

# no effect. This should fall within `--cluster-cidr`.
# - name: CALICO_IPV4POOL_CIDR
#   value: "192.168.0.0/16"
# Disable file logging so `kubectl logs` works.
- name: CALICO_DISABLE_FILE_LOGGING
  value: "true"

#把两个#及#后面的空格去掉，并把192.168.0.0/16改成10.244.0.0/16

# no effect. This should fall within `--cluster-cidr`.
- name: CALICO_IPV4POOL_CIDR
  value: "10.244.0.0/16"
# Disable file logging so `kubectl logs` works.
- name: CALICO_DISABLE_FILE_LOGGING
  value: "true"
```

**步骤3：提前下载所需要的镜像。**

``` bash
#查看此文件用哪些镜像：
grep image calico.yaml

#image: calico/cni:v3.20.6
#image: calico/cni:v3.20.6
#image: calico/pod2daemon-flexvol:v3.20.6
#image: calico/node:v3.20.6
#image: calico/kube-controllers:v3.20.6
```

在 master 节点中下载镜像

``` shell
#换成自己的版本
for i in calico/cni:v3.20.6 calico/pod2daemon-flexvol:v3.20.6 calico/node:v3.20.6 calico/kube-controllers:v3.20.6 ; do docker pull $i ; done
```

#### 安装可视化工具 kuboard

``` bash
#master上
wget https://kuboard.cn/install-script/kuboard.yaml
#所有节点下载镜像
docker pull eipwork/kuboard:latest
```

#### Kube-shell

[kube-shell](https://link.zhihu.com/?target=https%3A//github.com/cloudnativelabs/kube-shell)（推荐，kubectl 编辑的小工具）

``` bash
#安装
pip install kube-shell

#启动
kube-shell

#退出
exit
```

### 安装 KubeEdge

安装版本：v1.9.2

#### 安装keadm

进入官网版本链接：[Release KubeEdge v1.9.2 release · kubeedge/kubeedge (github.com)](https://github.com/kubeedge/kubeedge/releases/tag/v1.9.2)

右键复制链接地址(注意架构），节点中输入：

``` bash
#查看自己的架构
uname -u

#下载对应版本以及架构
https://github.com/kubeedge/kubeedge/releases/download/v1.9.2/keadm-v1.9.2-linux-amd64.tar.gz
#解压
tar zxvf keadm-v1.9.2-linux-amd64.tar.gz
#添加执行权限
chmod +x keadm-v1.9.2-linux-amd64/keadm/keadm 
#移动目录
mv keadm-v1.9.2-linux-amd64/keadm/keadm /usr/local/bin/
```

#### 安装 metrics-server(后续主要用于HPA)

- 用于追踪边缘节点日志
- 官网安装 (会出错, 拉取镜像超时问题)：

``` bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

> \[!info\]
> \![\[Pasted image 20231103172017.png\]\]

- 本地安装

``` bash
# 先下载官方提供的yaml
wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

修改 yaml 的内容，先看官方的版本，然后去 docker hub 找对应的镜像，**并且添加"- --kubelet-insecure-tls"**
\![\[Pasted image 20231103173826.png\]\]

去 docker hub 找镜像
\![\[Pasted image 20231103173853.png\]\]

修改 `components.yaml` 文件

``` vim
    spec:
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=4443
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        - --kubelet-insecure-tls
        image: mingyangtech/klogserver:v0.6.4
```

\![\[Pasted image 20231103180109.png\]\]

然后手动 pull 镜像

``` bash
docker pull mingyangtech/klogserver:v0.6.4
```

> \[!hint\]
> 一定要写版本号，否则会报错
> \![\[Pasted image 20231103174059.png\]\]

#### EdgeMesh

kubeedge的通讯组件，把github源码下载下来

``` bash
git clone https://github.com/kubeedge/edgemesh.git
```

#### Sedna

##### 官方下载（会超时）

``` bash
curl https://raw.githubusercontent.com/kubeedge/sedna/main/scripts/installation/install.sh | SEDNA_ACTION=create bash -
```

> \[!info\] 超时报错
> \![\[Pasted image 20231103170317.png\]\]

##### 本地安装

- 下载 `install.sh` 文件（不要保存到 build 中，可以永久使用）

``` bash
wget https://raw.githubusercontent.com/kubeedge/sedna/main/scripts/installation/install.sh
```

- 相关的修改（不要保存到 build 中，可以永久使用）

``` bash
#改名字
mv install.sh offline-install.sh

#修改文件中的一下内容（sh的语法）
1.第28行：TMP_DIR='/opt/sedna'
2.第34行：删除  trap "rm -rf '%TMP_DIR'" EXIT
3.第415行：删除  prepare
```

> \[!warning\]
> 记得看 `install.sh` 中 sedna 的版本，去 docker hub 中看镜像是否适用于你的架构!! (主要针对 arm)

- 下载 sedna 的官网 github（不要保存到 build 中，可以永久使用）
  官方地址：[http://github.com/kubeedge/sed](https://link.zhihu.com/?target=http%3A//github.com/kubeedge/sedna/tree/main/build)

``` bash
git clone https://github.com/kubeedge/sedna.git
```

- 创建安装目录（和上面的路径要一样）, 并且复制文件

``` bash
mkdir /opt/sedna
#把下载的sedna文件中的build目录复制到该地址
mv ./sedna /opt/sedna/
```

## Node 节点前期准备

### 关闭防火墙

``` bash
ufw disable
setenforce 0  #临时关闭
```

### 开启 ipv4转发，配置 iptables 参数

**将桥接的 IPv4/IPv6 流量传递到 iptables 的链**

``` bash
modprobe br_netfilter
#方法1
cat >> /etc/sysctl.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sysctl -p   #执行此命令生效配置
```

### 禁用 swap 分区

``` bash
swapoff -a                                          #临时关闭
sed -ri 's/.*swap.*/#&/' /etc/fstab                 #永久关闭
```

### 设置主机名

在不同的虚拟机上执行

``` text
hostnamectl set-hostname master    
bash

hostnamectl set-hostname node
bash
```

### 安装 docker

``` bash
sudo apt-get install docker.io -y

sudo systemctl start docker
sudo systemctl enable docker

docker --version

# 如果是 ARM64 架构
# 参考 https://blog.csdn.net/qq_34253926/article/details/121629068
```

一定要配置的

``` bash
sudo vim /etc/docker/daemon.json

#master添加：
{
  "registry-mirrors": ["https://b9pmyelo.mirror.aliyuncs.com"],
  "exec-opts": ["native.cgroupdriver=systemd"]
}

#node添加：
{
  "registry-mirrors": ["https://b9pmyelo.mirror.aliyuncs.com"],
  "exec-opts": ["native.cgroupdriver=cgroupfs"]
}

#都要执行
sudo systemctl daemon-reload
sudo systemctl restart docker

#查看修改后的docker Cgroup的参数
docker info | grep Cgroup
```

### 安装 KubeEdge

- 安装版本：v1.9.2

### 安装keadm

进入官网版本链接：[Release KubeEdge v1.9.2 release · kubeedge/kubeedge (github.com)](https://github.com/kubeedge/kubeedge/releases/tag/v1.9.2)

右键复制链接地址(注意架构），节点中输入：

``` bash
#查看自己的架构
uname -a

#下载对应版本以及架构
wget https://github.com/kubeedge/kubeedge/releases/download/v1.9.2/keadm-v1.9.2-linux-arm64.tar.gz
#解压
tar zxvf keadm-v1.9.2-linux-arm64.tar.gz
#添加执行权限
chmod +x keadm-v1.9.2-linux-arm64/keadm/keadm 
#移动目录
mv keadm-v1.9.2-linux-arm64/keadm/keadm /usr/local/bin/
```

## 运行

### K8s + KubeEdge

#### Kubernetes 启动

##### 删除从前的信息（如果是第一次可跳过）

``` bash
swapoff -a
kubeadm reset
systemctl daemon-reload
systemctl restart docker kubelet
rm -rf $HOME/.kube/config

rm -f /etc/kubernetes/kubelet.conf    #删除k8s配置文件
rm -f /etc/kubernetes/pki/ca.crt    #删除K8S证书

iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
```

##### 初始化 k8s 集群

``` bash
kubeadm init --pod-network-cidr 10.244.0.0/16 \
--image-repository registry.cn-hangzhou.aliyuncs.com/google_containers

#上一步创建成功后，会提示：主节点执行提供的代码
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

#如果只有一个点，需要容纳污点
kubectl taint nodes --all node-role.kubernetes.io/master node-role.kubernetes.io/master-
```

> \[!note\] 可能的问题

```bash
(未理解原因)使用下面的命令可能会报错：
kubeadm init  
--apiserver-advertise-address=192.168.247.128  
--image-repository registry.aliyuncs.com/google_containers  
--kubernetes-version v1.22.10  
--service-cidr=10.96.0.0/12  
--pod-network-cidr=10.244.0.0/16  
--ignore-preflight-errors=all
(最后加上 --v=5 会输出一些日志)

报错内容如下：

\[kubelet-check\] Initial timeout of 40s passed.

    Unfortunately, an error has occurred:
        timed out waiting for the condition
    
    This error is likely caused by:
        - The kubelet is not running
        - The kubelet is unhealthy due to a misconfiguration of the node in some way (required cgroups disabled)
    
    If you are on a systemd-powered system, you can try to troubleshoot the error with the following commands:
        - 'systemctl status kubelet'
        - 'journalctl -xeu kubelet'
    
    Additionally, a control plane component may have crashed or exited when started by the container runtime.
    To troubleshoot, list all containers using your preferred container runtimes CLI.
    
    Here is one example how you may list all Kubernetes containers running in docker:
        - 'docker ps -a | grep kube | grep -v pause'
        Once you have found the failing container, you can inspect its logs with:
        - 'docker logs CONTAINERID'

error execution phase wait-control-plane: couldn't initialize a Kubernetes cluster
To see the stack trace of this error execute with --v=5 or higher
```



##### 启动 calico 和 kuboard

    #进入这两个文件的目录，一定要执行，不然coredns无法初始化
    kubectl apply -f calico.yaml
    kubectl apply -f kuboard.yaml
    
    #检查启动状态，需要都running，除了calico可以不用
    kubectl get pods -A
    
    #获取 kuboard  token
    echo $(kubectl -n kube-system get secret $(kubectl -n kube-system get secret | grep kuboard-user | awk '{print $1}') -o go-template='{{.data.token}}' | base64 -d)

#### KubeEdge 启动

##### Master

###### 重启（如果第一次可以跳过）

``` bash
keadm reset
```

###### 启动 cloudcore

通过keadm启动

``` bash
#注意修改--advertise-address的ip地址
keadm init --advertise-address=114.212.81.11 --kubeedge-version=1.9.2


#打开转发路由
kubectl get cm tunnelport -n kubeedge -o yaml
#找到10350 或者10351
#set the rule of trans，设置自己的端口
CLOUDCOREIPS=xxx.xxx.xxx.xxx

# dport的内容对应tunnelport
iptables -t nat -A OUTPUT -p tcp --dport 10351 -j DNAT --to $CLOUDCOREIPS:10003
```

Edge 节点加入时可能会自动部署 calico-node 和 kube-proxy，kube-proxy 会部署成功（但是 edgecore 的 log 会提示不应该部署 kube-proxy），calico 会初始化失败。为了避免上述情况，做如下操作：

``` bash
kubectl get daemonset -n kube-system | grep -v NAME | awk '{print $1}' |xargs -n 1 kubectl patch daemonset -n kube-system --type='json' -p='[{"op":"replace","path":"/spec/template/spec/affinity","value":{"nodeAffinity":{"requireDuringSchedulingIgnoredDuringExecution":{"nodeSelectorTerms":[{"matchExpressions":[{"key":"node-role.kubernetes.io/edge","operator":"DoesNotExist"}]}]}}}}]'

$ kubectl edit daemonset -n kube-system kube-proxy

# 添加以下内容
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-role.kubernetes.io/edge
                operator: DoesNotExist

# calico-node添加相同内容
$ kubectl edit daemonset -n kube-system calico-node
```

\![\[Pasted image 20231104180203.png\]\]

###### 启动 metric-service

方便跟踪日志

``` bash
#进入自己的下载目录
kubectl apply -f components.yaml
#line 144
#image: xin053/metrics-server:v0.6.2

# 检验是否部署成功
$ kubectl top nodes
```

###### `kubectl logs/exec`

[Enable Kubectl logs/exec to debug pods on the edge \| KubeEdge](https://kubeedge.io/docs/advanced/debug/)

``` bash
export CLOUDCOREIPS="114.212.81.11"

echo $CLOUDCOREIPS

cd /etc/kubeedge/
cp $GOPATH/src/github.com/kubeedge/kubeedge/build/tools/certgen.sh /etc/kubeedge/

bash certgen.sh stream
```

设置 iptables 规则（理论上启动 cloudcore 就设置好了）

``` bash
iptables -t nat -A OUTPUT -p tcp --dport 10350 -j DNAT --to $CLOUDCOREIPS:10003
```

更新配置
- 修改：`/etc/kubeedge/config/cloudcore.yaml`

    ```bash
    cloudStream:  
        enable: true  
        streamPort: 10003  
        tlsStreamCAFile: /etc/kubeedge/ca/streamCA.crt  
        tlsStreamCertFile: /etc/kubeedge/certs/stream.crt  
        tlsStreamPrivateKeyFile: /etc/kubeedge/certs/stream.key  
        tlsTunnelCAFile: /etc/kubeedge/ca/rootCA.crt  
        tlsTunnelCertFile: /etc/kubeedge/certs/server.crt  
        tlsTunnelPrivateKeyFile: /etc/kubeedge/certs/server.key  
        tunnelPort: 10004
    ```

    

- 修改：`/etc/kubeedge/config/edgecore.yaml`

``` bash
edgeStream:  
    enable: true  
    handshakeTimeout: 30  
    readDeadline: 15  
    server: 192.168.0.139:10004  
    tlsTunnelCAFile: /etc/kubeedge/ca/rootCA.crt  
    tlsTunnelCertFile: /etc/kubeedge/certs/server.crt  
    tlsTunnelPrivateKeyFile: /etc/kubeedge/certs/server.key  
    writeDeadline: 15
```

重启
- 在云端：

``` bash
sudo systemctl restart cloudcore.service
```

- 在边缘侧：

``` bash
sudo systemctl restart edgecore.service
```

##### Node

###### 启动 edgecore

消除之前的信息（如果第一次，请跳过）

``` bash
#重启
keadm reset

#重新加入时，报错存在文件夹，直接删除
rm -rf /etc/kubeedge

#docker容器占用(通常是mqtt)
docker ps -a
docker stop mqtt
docker rm mqtt
```

###### 启动

``` bash
#注意这里是要加入master的IP地址，token是master上获取的
keadm join --cloudcore-ipport=114.212.81.11:10000 --kubeedge-version=1.9.2 --token=621812b3b27f180cceb29a707bb49146abcc8ea04bda810d53d8a0e3fc4c023f.eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE3MDQyNTIzNjR9.oqFu_1ETGlOhTErR8VBkkipDUMQ6tYCzGRLjQQ8Uxog
```

如果一直显示下载问题，可以手动下载对应的文件：

\![\[Pasted image 20240105150904.png\]\]

如果 check 时候提醒 check 未通过，询问是否删除源文件重新下载，填 n，否则它又要重新去下载。

==重要==

``` bash
#加入后，先查看信息
journalctl -u edgecore.service -f
```

如果修改了 daemonset 的话此时 `docker ps` 只会有 mqtt
\![\[Pasted image 20231104180646.png\]\]

> \[!可能出现的问题\]

``` bash
execute keadm command failed:  failed to exec 'bash -c sudo ln /etc/kubeedge/edgecore.service /etc/systemd/system/edgecore.service && sudo systemctl daemon-reload && sudo systemctl enable edgecore && sudo systemctl start edgecore', err: ln: failed to create hard link '/etc/systemd/system/edgecore.service': File exists
, err: exit status 1
```

在尝试创建符号链接时，目标路径已经存在，因此无法创建。这通常是因为 `edgecore.service` 已经存在于 `/etc/systemd/system/` 目录中

``` bash
sudo rm /etc/systemd/system/edgecore.service
```

#### `kubectl` 安装

1.  下载 kubectl

``` bash
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.17.0/bin/linux/amd64/kubectl
```

2.  使 kubectl 二进制可执行文件

``` bash
chmod +x ./kubectl
```

3.  将二进制文件移到 PATH中

``` bash
sudo mv ./kubectl /usr/local/bin/kubectl
```

4.  测试安装版本

``` bash
kubectl version --client
```

5.  将 master 节点下/etc/kubernetes/admin. Conf 复制到 edge 节点，配置环境变量

``` bash
vim /etc/profile 
export KUBECONFIG=/etc/kubernetes/admin.conf【修改地址】 
source /etc/profile
```

### 启动 EdgeMesh

#### 前期准备

**步骤1**: 去除 K8s master 节点的污点

``` bash
$ kubectl taint nodes --all node-role.kubernetes.io/master-
```

**步骤2**: 给 Kubernetes API 服务添加过滤标签

``` bash
$ kubectl label services kubernetes service.edgemesh.kubeedge.io/service-proxy-name=""
```

- **步骤3**: 启用 KubeEdge 的边缘 Kube-API 端点服务**important**

- 在云端，开启 dynamicController 模块，配置完成后，需要重启 cloudcore(**1.10+ 版本，cloudcore 以容器方式运行**)

``` bash
#keadm安装的
kubectl edit cm -n kubeedge cloudcore
#修改
modules:
  ...
  dynamicController:
    enable: true
...

#执行完后,检查一下
kubectl describe cm -n kubeedge cloudcore

#如果不放心，直接去kuboard在kubeedge上把cloudcore删除掉，然后会根据新的模板创建新的容器
```

\![\[Pasted image 20231104175821.png\]\]

\![\[Pasted image 20231104180445.png\]\]

**KubeEdge 1.9.2 版本：**
修改配置，`vim /etc/kubeedge/config/cloudcore.yaml`，重启 cloudcore 后才生效。重启命令：`systemctl restart cloudcore`

``` bash
modules:
  ..
  cloudStream:
    enable: true
    streamPort: 10003
  ..
  dynamicController:
    enable: true
```

**在边缘节点，打开 metaServer 模块（如果KubeEdge \< 1.8.0，还需关闭旧版 edgeMesh 模块），配置完成后，需要重启 edgecore**

``` bash
$ vim /etc/kubeedge/config/edgecore.yaml
modules:
  ...
  edgeMesh:
    enable: false
  ...
  metaManager:
    metaServer:
      enable: true
...
```

\![\[Pasted image 20231104180820.png\]\]

在边缘节点配置 clusterDNS 和 clusterDomain，配置完成后，需要重启 edgecore。

``` bash
# 1.12+以下操作
$ vim /etc/kubeedge/config/edgecore.yaml
modules:
  ...
  edged:
    ...
    tailoredKubeletConfig:
      ...
      clusterDNS:
      - 169.254.96.16
      clusterDomain: cluster.local
...

# 1.12-以下操作
$ vim /etc/kubeedge/config/edgecore.yaml
modules:
  ...
  edged:
    clusterDNS: 169.254.96.16
    clusterDomain: cluster.local

#重启
systemctl restart edgecore.service
#检查状况
journalctl -u edgecore.service -f
#确定正常运行
```

\![\[Pasted image 20231104181004.png\]\]

**添加的内容在 tailoredKubeletConfig 下第一二位才生效，不理解但是不在这个位置就不生效了** 即一定是图上这样的顺序

**步骤4**: 最后，在边缘节点，测试边缘 Kube-API 端点功能是否正常

``` bash
# 边缘节点上
curl 127.0.0.1:10550/api/v1/services
```

\![\[Pasted image 20231105211356.png\]\]

> \[!note\]
> 如果返回值是空列表，或者响应时长很久（接近 10 s）才拿到返回值，说明你的配置可能有误，请仔细检查。

> \[!可能的问题\]

``` bash
 TLSStreamPrivateKeyFile: Invalid value: "/etc/kubeedge/certs/stream.key": TLSStreamPrivateKeyFile not exist
12月 14 23:02:23 cloud.kubeedge cloudcore[196229]:   TLSStreamCertFile: Invalid value: "/etc/kubeedge/certs/stream.crt": TLSStreamCertFile not exist
12月 14 23:02:23 cloud.kubeedge cloudcore[196229]:   TLSStreamCAFile: Invalid value: "/etc/kubeedge/ca/streamCA.crt": TLSStreamCAFile not exist
12月 14 23:02:23 cloud.kubeedge cloudcore[196229]: ]
```

查看 `/etc/kubeedge` 下是否有 `certgen.sh` 并且 `bash certgen.sh stream`

> \[!补充\]
> 为了在云端可以 logs 边端的 pod 的信息，需要修改 /etc/kubeedge/config/edgecore.yaml 文件，将 `edgeStream` 下 `enable` 改为 true

\![\[Pasted image 20240102173732.png\]\]

#### 安装

- **步骤1**: 获取 EdgeMesh

``` bash
#或者自己提前下载好
git clone https://github.com/kubeedge/edgemesh.git
cd edgemesh
```

- **步骤2**: 安装 CRDs

``` bash
kubectl apply -f build/crds/istio/
```

- **步骤 3**: 部署 edgemesh-agent

`resources` 中的内容需要进行修改，具体看 

``` bash
kubectl apply -f build/agent/resources/
```

> \[!cite\]
> [快速上手 \| EdgeMesh](https://edgemesh.netlify.app/zh/guide/#%E6%89%8B%E5%8A%A8%E5%AE%89%E8%A3%85)
> [全网最全EdgeMesh Q&A手册 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/585749690)

- **步骤4**: 检验部署结果

``` bash
kubectl get all -n kubeedge -o wide
```

> \[!warning\]
> 记得查看所有 `edgemesh-agent` 的信息，排查所有 `fail` 和 `warning` (warning 尽量解决)

#### 运行正常截图

云端 agent------
\![\[Pasted image 20231120162458.png\]\]
**云端和边端不在同一子网记得设置 relayNode**

边端 agent------
\![\[Pasted image 20231120162613.png\]\]

###  Sedna

#### 安装

``` bash
#直接进入之前准备好的文件目录运行
SEDNA_ACTION=create bash - offline-install.sh

# 卸载
SEDNA_ACTION=delete bash - offline-install.sh
```

#### 正常运行情况

如图展示正常运行情况：
`lc-edge` :
\![\[Pasted image 20231120113839.png\]\]

`gm`
\![\[Pasted image 20231120112502.png\]\]

**目前存在问题：连接具有偶然性？**
\![\[Pasted image 20231120142020.png\]\]

#### Demo

跑官方给的 Joint inference [Using Joint Inference Service in Helmet Detection Scenario --- Sedna 0.4.1 documentation](https://sedna.readthedocs.io/en/latest/examples/joint_inference/helmet_detection_inference/README.html)

##### 准备模型

- 边缘节点准备小模型

``` bash
mkdir -p /data/little-model
cd /data/little-model
wget https://kubeedge.obs.cn-north-1.myhuaweicloud.com/examples/helmet-detection-inference/little-model.tar.gz
tar -zxvf little-model.tar.gz
```

- 云准备大模型

``` bash
mkdir -p /data/big-model
cd /data/big-model
wget https://kubeedge.obs.cn-north-1.myhuaweicloud.com/examples/helmet-detection-inference/big-model.tar.gz
tar -zxvf big-model.tar.gz
```

##### 拉取镜像

**注意看镜像适用的架构**

``` bash
# 边
docker pull kubeedge/sedna-example-joint-inference-helmet-detection-little:v0.3.0

# 云
docker pull kubeedge/sedna-example-joint-inference-helmet-detection-big:v0.3.0
```

##### 部署服务

- 为云准备大模型

``` bash
kubectl create -f - <<EOF
apiVersion: sedna.io/v1alpha1
kind:  Model
metadata:
  name: helmet-detection-inference-big-model
  namespace: default
spec:
  url: "/data/big-model/yolov3_darknet.pb"
  format: "pb"
EOF
```

- 为边准备小模型

``` bash
kubectl create -f - <<EOF
apiVersion: sedna.io/v1alpha1
kind: Model
metadata:
  name: helmet-detection-inference-little-model
  namespace: default
spec:
  url: "/data/little-model/yolov3_resnet18.pb"
  format: "pb"
EOF
```

- 边端创建输出目录
  **记得提前创建，否则会启 pod 时会一直 pending, edgecore 会有报错内容，大致是 volumemounts 失败**

``` bash
mkdir -p /joint_inference/output
```

- 准备 Service 的 yaml 文件
  
  


    >[!tip]
    >1. 删除重新部署的话除了 `delete -f ` 上面的 yaml 文件，还要手动删除一些 pod 和 **svc**。否则可能会没有效果，此时 edgecore、lc 可能会有相关报错
    >2. 如果删清了还是没有对应的 pod，注意查看 model 和 service 的 metadata 下的 name 是否对应...

##### 模拟视频流（边）

- step1: install the open source video streaming server [EasyDarwin](https://github.com/EasyDarwin/EasyDarwin/tree/dev).
- step2: start EasyDarwin server.
- step3: download [video](https://kubeedge.obs.cn-north-1.myhuaweicloud.com/examples/helmet-detection-inference/video.tar.gz).
- step 4: push a video stream to the url (e.g., `rtsp://localhost/video`) that the inference service can connect.




    >[!cite]
    > `arm` 架构下的 easydarwin
    > [Ubuntu1604交叉编译全志T7开发板ARM版Easydarwin - 食铁兽 - FFmpeg/OpenCV/Qt/Presagis/VAPSXT/VegaPrime/STAGE/TerraVista/Ondulus/Creator/HeliSIM/FlightSIM/CDB/V5D/VxWorks/Wayland/破解入门技术分享 (feater.top)](https://feater.top/linux/cross-compile-easydarwin-under-ubuntu-t7-arm)


    wget https://github.com/EasyDarwin/EasyDarwin/releases/download/v8.1.0/EasyDarwin-linux-8.1.0-1901141151.tar.gz
    tar -zxvf EasyDarwin-linux-8.1.0-1901141151.tar.gz
    cd EasyDarwin-linux-8.1.0-1901141151
    ./start.sh
    
    mkdir -p /data/video
    cd /data/video
    wget https://kubeedge.obs.cn-north-1.myhuaweicloud.com/examples/helmet-detection-inference/video.tar.gz
    tar -zxvf video.tar.gz
    
    ffmpeg -re -i /data/video/video.mp4 -vcodec libx264 -f rtsp rtsp://localhost/video

\![\[Pasted image 20231105212428.png\]\]

> \[!tip\]
> 如果 pod 出现 `CrashLoopBackOff` 的状态的话可借鉴 [7 张图解 CrashLoopBackOff，如何发现问题并解决它？ - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/558957039)

##### 查看结果

查看/joint_inference/output（即上述创建的输出目录）

\![\[Pasted image 20231105212955.png\]\]

云端 pod 的日志如下，说明云边通信成功：
\![\[Pasted image 20231120160053.png\]\]

## GPU 支持

### 前期尝试

`k8s` 使用 GPU 需要 nvidia-device-plugin 插件支持

``` bash
kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.7.3/nvidia-device-plugin.yml
```

但是会发现问题：
\![\[Pasted image 20240103141531.png\]\]

这个问题也遇见了很多次了，大概就是架构问题导致无法运行，需要重新编译 docker

``` bash
git clone -b 1.0.0-beta6 https://github.com/nvidia/k8s-device-plugin.git

cd k8s-device-plugin

wget https://labs.windriver.com/downloads/0001-arm64-add-support-for-arm64-architectures.patch

wget https://labs.windriver.com/downloads/0002-nvidia-Add-support-for-tegra-boards.patch

wget https://labs.windriver.com/downloads/0003-main-Add-support-for-tegra-boards.patch

git am 000*.patch

应用：arm64: add support for arm64 architectures
应用：nvidia: Add support for tegra boards
应用：main: Add support for tegra boards

docker buildx  build --platform linux/arm64 -t adayoung/k8s-device-plugin:v0.1 -f docker/arm64/Dockerfile.ubuntu16.04 .
```

**出现问题**：
\![\[Pasted image 20240103171407.png\]\]

\![\[Pasted image 20240103171340.png\]\]

\![\[Pasted image 20240104142333.png\]\]

此问题的解决：只需要更新动态库配置文件和将它报错的路径链接过去

``` bash
root@edge2:/software# ldconfig
root@edge2:/software# whereis /sbin/ldconfig
ldconfig: /sbin/ldconfig
root@edge2:/software# ln -s /sbin/ldconfig /sbin/ldconfig.real
root@edge2:/software# sudo systemctl daemon-reload
root@edge2:/software# sudo systemctl restart docker
```

然而问题只会一个接一个的出现：

\![\[Pasted image 20240104142528.png\]\]

``` bash
root@cloud:/home/guest/yby/kubeedge/multiedge/test-yaml/gpu# kubectl logs nvidia-device-plugin-daemonset-edge-65lbm -n kube-system
2024/01/04 06:08:15 Loading NVML
2024/01/04 06:08:15 Failed to initialize NVML: could not load NVML library.
2024/01/04 06:08:15 If this is a GPU node, did you set the docker default runtime to `nvidia`?
2024/01/04 06:08:15 You can check the prerequisites at: https://github.com/NVIDIA/k8s-device-plugin#prerequisites
2024/01/04 06:08:15 You can learn how to set the runtime at: https://github.com/NVIDIA/k8s-device-plugin#quick-start
2024/01/04 06:08:15 If this is not a GPU node, you should set up a toleration or nodeSelector to only deploy this plugin on GPU nodes
2024/01/04 06:08:15 Error: failed to initialize NVML: could not load NVML library
```

> \[!quote\]
> [nvidia-smi installation on Jetson TX2 - Jetson & Embedded Systems / Jetson TX2 - NVIDIA Developer Forums](https://forums.developer.nvidia.com/t/nvidia-smi-installation-on-jetson-tx2/111495)

\![\[Pasted image 20240104142637.png\]\]

官方回复 jetson 系列不能用 NVML...
但是这个插件开发者兼容了 jetson 板子！
\>\[!quote\]
\> [Add support for arm64 and Jetson boards. (!20) · 合并请求 · nvidia / kubernetes / device-plugin · GitLab](https://gitlab.com/nvidia/kubernetes/device-plugin/-/merge_requests/20)
\>[NVIDIA/k8s-device-plugin at v0.13.0 (github.com)](https://github.com/NVIDIA/k8s-device-plugin/tree/v0.13.0)

### 部署

运行下面命令即可：

``` bash
kubectl apply -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.13.0/nvidia-device-plugin.yml
```

> \[!hint\]
> 在边端 `vim /etc/kubeedge/config/edgecore.yaml`
> 将修改使得 `devicePluginEnabled: true`。本身这个插件是提供给 `k8s` 使用的，本地节点采用的 kubelet 进行管理，但是 kubeedge 采用 edgecore，打开 devicePluginEnabled 才可以

### 正常运行

- **云端**：
  \![\[Pasted image 20240104154705.png\]\]

\![\[Pasted image 20240104154743.png\]\]

- 边端（jetson）
  \![\[Pasted image 20240104155133.png\]\]

\![\[Pasted image 20240104155207.png\]\]

``` bash
kubectl run -i -t nvidia --image=jitteam/devicequery
```

\![\[nvidia.png\]\]
### 可能出现的问题
#### 是 GPU 节点但是找不到 gpu 资源
主要针对的是服务器的情况，可以使用 `nvidia-smi` 查看显卡情况。

> \[!quote\]
> [Installing the NVIDIA Container Toolkit --- NVIDIA Container Toolkit 1.14.3 documentation](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html#configuration)

按上述内容进行配置，然后还需要 `vim /etc/docker/daemon.json`，添加 default-runtime。按照 quote 的内容设置后会有"runtimes"但是 default-runtime 不会设置，可能会导致找不到 GPU 资源

    {
        "exec-opts": [
            "native.cgroupdriver=systemd"
        ],
        "registry-mirrors": [
            "https://b9pmyelo.mirror.aliyuncs.com"
        ],
        "default-runtime": "nvidia",
        "runtimes": {
            "nvidia": {
                "args": [],
                "path": "nvidia-container-runtime"
            }
        }
    }

#### jeston 系列板子问题

理论上 `k8s-device-plugin` 已经支持了 tegra 即 jetson 系列板子，会在查看 GPU 之前判断是否是 tegra 架构，如果是则采用 tegra 下查看 GPU 的方式（原因在 

``` bash
2024/01/04 07:43:58 Retreiving plugins.
2024/01/04 07:43:58 Detected non-NVML platform: could not load NVML: libnvidia-ml.so.1: cannot open shared object file: No such file or directory
2024/01/04 07:43:58 Detected non-Tegra platform: /sys/devices/soc0/family file not found
2024/01/04 07:43:58 Incompatible platform detected
2024/01/04 07:43:58 If this is a GPU node, did you configure the NVIDIA Container Toolkit?
2024/01/04 07:43:58 You can check the prerequisites at: https://github.com/NVIDIA/k8s-device-plugin#prerequisites
2024/01/04 07:43:58 You can learn how to set the runtime at: https://github.com/NVIDIA/k8s-device-plugin#quick-start
2024/01/04 07:43:58 If this is not a GPU node, you should set up a toleration or nodeSelector to only deploy this plugin on GPU nodes
2024/01/04 07:43:58 No devices found. Waiting indefinitely.
```

\![\[Pasted image 20240104160107.png\]\]

``` bash
$ dpkg -l '*nvidia*'
```

\![\[Pasted image 20240104170241.png\]\]

\![\[Pasted image 20240104170030.png\]\]

> \[!quote\]
> [Plug in does not detect Tegra device Jetson Nano · Issue \#377 · NVIDIA/k8s-device-plugin (github.com)](https://github.com/NVIDIA/k8s-device-plugin/issues/377)
>
> Note that looking at the initial logs that you provided you may have been using `v1.7.0` of the NVIDIA Container Toolkit. This is quite an old version and we greatly improved our support for Tegra-based systems with the `v1.10.0` release. It should also be noted that in order to use the GPU Device Plugin on Tegra-based systems (specifically targetting the integrated GPUs) at least `v1.11.0` of the NVIDIA Container Toolkit is required.
>
> There are no Tegra-specific changes in the `v1.12.0` release, so using the `v1.11.0` release should be sufficient in this case.

那么应该需要升级**NVIDIA Container Toolkit**

``` bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt-get update

sudo apt-get install -y nvidia-container-toolkit
```

## 可能出现的问题

### `kubectl logs <pod-name>` 超时

\![\[e33bf25de9ae55563d629577621f522.png\]\]

原因：借鉴 [Kubernetes 边缘节点抓不到监控指标？试试这个方法！ - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/379962934)
\![\[e3552fea70b2050f44fd100889d7b7d.png\]\]
可以发现报错访问的端口就是 10350，而在 kubeedge 中 10350 应该会进行转发，所以应该是 cloudcore 的设置问题。

解决：[Enable Kubectl logs/exec to debug pods on the edge \| KubeEdge](https://kubeedge.io/docs/advanced/debug/) 根据这个链接设置即可。

### `kubectl logs <pod-name>` 卡住

可能的原因：之前 `kubectl logs` 时未结束就 ctrl+c 结束了导致后续卡住
解决：重启 edgecore `systemctl restart edgecore.service`

### Edgemesh log 报错

#### 问题描述

1. **节点情况**：![[Pasted image 20231118155810.png]]
2. **EdgeMesh 日志**
  **边缘节点**：![[Pasted image 20231118155843.png]]
  **服务器**：![[Pasted image 20231118155915.png]]
  EdgeMesh 监听的端口是 20006，**发现服务器并没有监听 20006 端口：**![[Pasted image 20231118162239.png]]
  边端正常运行情况：
  ![[Pasted image 20231118165901.png]]



#### 排查

先复习一下**定位模型**，确定**被访问节点**上的 edgemesh-agent(右)容器是否存在、是否处于正常运行中。

**这个情况是非常经常出现的**，因为master节点一般都有污点，会驱逐其他pod，进而导致edgemesh-agent部署不上去。这种情况可以通过去除节点污点，使edgemesh-agent部署上去解决。

如果访问节点和被访问节点的edgemesh-agent都正常启动了，但是还报这个错误，可能是因为访问节点和被访问节点没有互相发现导致，请这样排查：

1. 首先每个节点上的edgemesh-agent都具有peer ID，比如


    edge2: 
    I'm {12D3KooWPpY4GqqNF3sLC397fMz5ZZfxmtMTNa1gLYFopWbHxZDt: [/ip4/127.0.0.1/tcp/20006 /ip4/192.168.1.4/tcp/20006]}
    
    edge1.kubeedge:
    I'm {12D3KooWFz1dKY8L3JC8wAY6sJ5MswvPEGKysPCfcaGxFmeH7wkz: [/ip4/127.0.0.1/tcp/20006 /ip4/192.168.1.2/tcp/20006]}
    
    注意：
    a. peer ID是根据节点名称哈希出来的，相同的节点名称会哈希出相同的peer ID
    b. 另外，节点名称不是服务器名称，是k8s node name，请用kubectl get nodes查看

2.如果访问节点和被访问节点处于同一个局域网内（**所有节点应该具备内网IP（10.0.0.0/8、172.16.0.0/12、192.168.0.0/16**），请看[全网最全EdgeMesh Q&A手册 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/585749690)**问题十二**同一个局域网内edgemesh-agent互相发现对方时的日志是 `[MDNS] Discovery found peer: <被访问端peer ID: [被访问端IP列表(可能会包含中继节点IP)]>` \^d3939d

3.如果访问节点和被访问节点跨子网，这时候应该看看 relayNodes 设置的正不正确，为什么中继节点没办法协助两个节点交换 peer 信息。详细材料请阅读：[KubeEdge EdgeMesh 高可用架构详解](https://link.zhihu.com/?target=https%3A//mp.weixin.qq.com/s/4whnkMM9oOaWRsI1ICsvSA)。跨子网的 edgemesh-agent 互相发现对方时的日志是 `[DHT] Discovery found peer: <被访问端peer ID: [被访问端IP列表(可能会包含中继节点IP)]>`（适用于我的情况）

#### 解决

在部署 edgemesh 进行 `kubectl apply -f build/agent/resources/` 操作时，修改 04-configmap，添加 relayNode（根本原因在于，不符合 

> \[!warning\]
> 请根据你的 K8s 集群设置 build/agent/resources/04-configmap.yaml 的 relayNodes，并重新生成 PSK 密码。

\![\[Pasted image 20231120173453.png\]\]

### Sedna lc 报错

#### `lc` **报错内容**

\![\[Pasted image 20231118175658.png\]\]

原因：域名解析问题。
#### 解决
临时解决方案：(理论上它会提示不要手动修改...)

在宿主机上修改 `/etc/resolv.conf`

`bash file:resolv.conf nameserver 169.254.96.16`

\![\[Pasted image 20231120160301.png\]\]

#### 仍存在的问题

\![\[Pasted image 20231206190513.png\]\]

在 $edge2$ 上连接到 gm 存在偶然性，最终虽然一定能连上，但是会尝试很多次。

\![\[Pasted image 20231206190652.png\]\]

### 在 kube-shell 下修改了命名空间

命令行下修改命名空间：

``` bash
kubectl config set-context $(kubectl config current-context) --namespace=default
```

### build gm 报错

#### 问题描述

\![\[Pasted image 20231121172643.png\]\]

``` bash
fetch http://mirrors.ustc.edu.cn/alpine/v3.15/main/x86_64/APKINDEX.tar.gz
ERROR: http://mirrors.ustc.edu.cn/alpine/v3.15/main/: No such file or directory
WARNING: Ignoring http://mirrors.ustc.edu.cn/alpine/v3.15/main/: No such file or directory
1 errors; 15 distinct packages available
The command '/bin/sh -c apk update' returned a non-zero code: 1
make: *** [Makefile:158: gmimage] Error 1
```

#### 解决

由于在宿主机上直接 wget 链接是可以下载的，推测是 docker 内网络问题，于是指定网络。`make gmimage` 实际对应的命令是 `docker build --build-arg GO_LDFLAGS="" --network=host -t kubeedge/sedna-gm:v0.3.0 -f build/gm/Dockerfile .`
最后再添加上 `--network=host`

即最终的命令为：

``` bash
docker build --build-arg GO_LDFLAGS="" --network=host -t kubeedge/sedna-gm:v0.3.0 -f build/gm/Dockerfile .
```

### 官方提供的镜像只适合于 `x86` 架构

#### 解决

使用**交叉编译**：
##### 安装 buildx
`buildx` 是 Docker 官方提供的一个构建工具，它可以帮助用户快速、高效地构建 Docker 镜像，并支持多种平台的构建。使用 `buildx`，用户可以在单个命令中构建多种架构的镜像，例如 x86 和 ARM 架构，而无需手动操作多个构建命令。此外，`buildx` 还支持 Dockerfile 的多阶段构建和缓存，这可以大大提高镜像构建的效率和速度。

如果需要手动安装，可以从 GitHub 发布页面[下载](https://link.zhihu.com/?target=https%3A//github.com/docker/buildx/releases/tag/v0.10.4)对应平台的最新二进制文件，重命名为 `docker-buildx`，然后将其放到 Docker 插件目录下（Linux/Mac 系统为 `$HOME/.docker/cli-plugins`，Windows 系统为 `%USERPROFILE%\.docker\cli-plugins`）。
给插件增加可执行权限 `chmod +x ~/.docker/cli-plugins/docker-buildx`

``` bash
# 我在虚拟机上进行的交叉编译，是x86架构
wget https://github.com/docker/buildx/releases/download/v0.10.4/buildx-v0.10.4.darwin-amd64

# 重命名
mv buildx-v0.10.4.darwin-amd64 docker-buildx

# 放到对应目录
mv docker-buildx $HOME/.docker/cli-plugins

# 添加权限
chmod +x ~/.docker/cli-plugins/docker-buildx

# 验证可用
docker buildx version
```

##### 创建 mybuilder

要使用 `buildx` 构建跨平台镜像，我们需要先创建一个 `builder`，可以翻译为「构建器」。

``` bash
$ docker buildx create --name mybuilder --use
mybuilder

$ docker buildx ls
NAME/NODE    DRIVER/ENDPOINT             STATUS  BUILDKIT             PLATFORMS
mybuilder *  docker-container                                         
  mybuilder0 unix:///var/run/docker.sock running v0.12.3              linux/amd64, linux/amd64/v2, linux/amd64/v3, linux/amd64/v4, linux/386
default      docker                                                   
  default    default                     running v0.11.6+0a15675913b7 linux/amd64, linux/amd64/v2, linux/amd64/v3, linux/amd64/v4, linux/386
```

其中 \* 所在的即为正在使用的

##### 启动 mybuilder

如果 mybuilder 的状态为 `inactive` 或者 `stopped` 需要手动进行启动（上面的操作中--use 会自动启动）

``` bash
$ docker buildx inspect --bootstrap mybuilder
[+] Building 16.8s (1/1) FINISHED
 => [internal] booting buildkit                                                                                                                                  16.8s
 => => pulling image moby/buildkit:buildx-stable-1                                                                                                               16.1s
 => => creating container buildx_buildkit_mybuilder0                                                                                                              0.7s
Name:   mybuilder
Driver: docker-container

Nodes:
Name:      mybuilder0
Endpoint:  unix:///var/run/docker.sock
Status:    running
Buildkit:  v0.12.3
Platforms: linux/amd64, linux/amd64/v2, linux/amd64/v3, linux/amd64/v4, linux/386
```

##### 构建镜像

以 sedna 的 lc 为例，lc 运行在云端和边端，其中云端是 x 86 架构，边端是 arm 架构

``` bash
$ docker buildx build --platform linux/arm64,linux/amd64 --build-arg GO_LDFLAGS="" -t adayoung/sedna-lc:v0.3.3 -f build/lc/Dockerfile . --push
```

### 大面积 Evicted（disk pressure）

#### 原因

- node 上的 kubelet 负责采集资源占用数据，并和预先设置的 threshold 值进行比较，如果超过 threshold 值，kubelet 会杀掉一些 Pod 来回收相关资源，[K8sg官网解读kubernetes配置资源不足处理](https://links.jianshu.com/go?to=https%3A%2F%2Fkubernetes.io%2Fdocs%2Ftasks%2Fadminister-cluster%2Fout-of-resource%2F)

- 默认启动时，node 的可用空间低于15%的时候，该节点上讲会执行 eviction 操作，由于磁盘已经达到了85%,在怎么驱逐也无法正常启动就会一直重启，Pod 状态也是 pending 中

#### 临时解决方法

- 修改配置文件增加传参数,添加此配置项`--eviction-hard=nodefs.available<5%`

``` bash
root@cloud:/usr/lib/systemd/system# systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/lib/systemd/system/kubelet.service; enabled; vendor preset: enabled)
    Drop-In: /etc/systemd/system/kubelet.service.d
             └─10-kubeadm.conf
     Active: active (running) since Fri 2023-12-15 09:17:50 CST; 5min ago
       Docs: https://kubernetes.io/docs/home/
   Main PID: 1070209 (kubelet)
      Tasks: 59 (limit: 309024)
     Memory: 67.2M
     CGroup: /system.slice/kubelet.service
```

可以看到配置文件目录所在位置是 `/etc/systemd/system/kubelet.service.d`，配置文件是 `10-kubeadm.conf`

``` bash
vim /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml --eviction-hard=nodefs.available<5%"

最后添加上--eviction-hard=nodefs.available<5%
```

\![\[Pasted image 20231215092603.png\]\]

然后重启 kubelet

``` bash
systemctl daemon-reload
systemctl  restart kubelet
```

会发现可以正常部署了(但是只是急救一下，磁盘空间感觉还是得清理下)

### 删除命名空间卡在 terminating

\![\[Pasted image 20231214145720.png\]\]

理论上一直等待应该是可以的(但是我等了半个钟也没成功啊!!)
**方法一**，但是没啥用，依旧卡住

``` bash
kubectl delete ns sedna --force --grace-period=0
```

**方法二**：

``` bash
开启一个代理终端
$ kubectl proxy
Starting to serve on 127.0.0.1:8001

再开启一个操作终端
将test namespace的配置文件输出保存
$ kubectl get ns sedna -o json > sedna.json
删除spec及status部分的内容还有metadata字段后的","号，切记！
剩下内容大致如下
guest@cloud:~/yby$ cat sedna.json
{
    "apiVersion": "v1",
    "kind": "Namespace",
    "metadata": {
        "creationTimestamp": "2023-12-14T09:12:13Z",
        "deletionTimestamp": "2023-12-14T09:15:25Z",
        "managedFields": [
            {
                "apiVersion": "v1",
                "fieldsType": "FieldsV1",
                "fieldsV1": {
                    "f:status": {
                        "f:phase": {}
                    }
                },
                "manager": "kubectl-create",
                "operation": "Update",
                "time": "2023-12-14T09:12:13Z"
            },
            {
                "apiVersion": "v1",
                "fieldsType": "FieldsV1",
                "fieldsV1": {
                    "f:status": {
                        "f:conditions": {
                            ".": {},
                            "k:{\"type\":\"NamespaceContentRemaining\"}": {
                                ".": {},
                                "f:lastTransitionTime": {},
                                "f:message": {},
                                "f:reason": {},
                                "f:status": {},
                                "f:type": {}
                            },
                            "k:{\"type\":\"NamespaceDeletionContentFailure\"}": {
                                ".": {},
                                "f:lastTransitionTime": {},
                                "f:message": {},
                                "f:reason": {},
                                "f:status": {},
                                "f:type": {}
                            },
                            "k:{\"type\":\"NamespaceDeletionDiscoveryFailure\"}": {
                                ".": {},
                                "f:lastTransitionTime": {},
                                "f:message": {},
                                "f:reason": {},
                                "f:status": {},
                                "f:type": {}
                            },
                            "k:{\"type\":\"NamespaceDeletionGroupVersionParsingFailure\"}": {
                                ".": {},
                                "f:lastTransitionTime": {},
                                "f:message": {},
                                "f:reason": {},
                                "f:status": {},
                                "f:type": {}
                            },
                            "k:{\"type\":\"NamespaceFinalizersRemaining\"}": {
                                ".": {},
                                "f:lastTransitionTime": {},
                                "f:message": {},
                                "f:reason": {},
                                "f:status": {},
                                "f:type": {}
                            }
                        }
                    }
                },
                "manager": "kube-controller-manager",
                "operation": "Update",
                "time": "2023-12-14T09:15:30Z"
            }
        ],
        "name": "sedna",
        "resourceVersion": "3351515",
        "uid": "99cb8afb-a4c1-45e6-960d-ff1b4894773d"
    }
}


调接口删除
# curl -k -H "Content-Type: application/json" -X PUT --data-binary @sedna.json http://127.0.0.1:8001/api/v1/namespaces/sedna/finalize
{
  "kind": "Namespace",
  "apiVersion": "v1",
  "metadata": {
    "name": "sedna",
    "uid": "99cb8afb-a4c1-45e6-960d-ff1b4894773d",
    "resourceVersion": "3351515",
    "creationTimestamp": "2023-12-14T09:12:13Z",
    "deletionTimestamp": "2023-12-14T09:15:25Z",
    "managedFields": [
      {
        "manager": "curl",
        "operation": "Update",
        "apiVersion": "v1",
        "time": "2023-12-14T09:42:38Z",
        "fieldsType": "FieldsV1",
        "fieldsV1": {"f:status":{"f:phase":{}}}
      }
    ]
  },
  "spec": {

  },
  "status": {
    "phase": "Terminating",
    "conditions": [
      {
        "type": "NamespaceDeletionDiscoveryFailure",
        "status": "True",
        "lastTransitionTime": "2023-12-14T09:15:30Z",
        "reason": "DiscoveryFailed",
        "message": "Discovery failed for some groups, 1 failing: unable to retrieve the complete list of server APIs: metrics.k8s.io/v1beta1: the server is currently unable to handle the request"
      },
      {
        "type": "NamespaceDeletionGroupVersionParsingFailure",
        "status": "False",
        "lastTransitionTime": "2023-12-14T09:15:30Z",
        "reason": "ParsedGroupVersions",
        "message": "All legacy kube types successfully parsed"
      },
      {
        "type": "NamespaceDeletionContentFailure",
        "status": "False",
        "lastTransitionTime": "2023-12-14T09:15:30Z",
        "reason": "ContentDeleted",
        "message": "All content successfully deleted, may be waiting on finalization"
      },
      {
        "type": "NamespaceContentRemaining",
        "status": "False",
        "lastTransitionTime": "2023-12-14T09:15:30Z",
        "reason": "ContentRemoved",
        "message": "All content successfully removed"
      },
      {
        "type": "NamespaceFinalizersRemaining",
        "status": "False",
        "lastTransitionTime": "2023-12-14T09:15:30Z",
        "reason": "ContentHasNoFinalizers",
        "message": "All content-preserving finalizers finished"
      }
    ]
  }
}
```

### 强制删除 pod 之后部署不成功

#### 问题描述

因为现在 edge 的 pod 是通过创建 deployment 由 deployment 进行创建，但是通过 `kubectl delete deploy <deploy-name>` 删除 deployment 后，pod 一直卡在了 terminating 状态，于是采用了 `kubectl delete pod edgeworker-deployment-7g5hs-58dffc5cd7-b77wz --force --grace-period=0` 命令进行了删除。
然后发现重新部署时候发现 assigned to edge 的 pod 都卡在 pending 状态。

#### 解决

因为--force 是不会实际终止运行的，所以本身原来的 docker 可能还在运行，现在的做法是手动去对应的边缘节点上删除对应的容器（包括 pause，关于 pause 可以看这篇文章[大白话 K8S（03）：从 Pause 容器理解 Pod 的本质 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/464712164)），然后重启 edgecore: `systemctl restart edgecore.service`

\![\[Pasted image 20231219113638.png\]\]

应该就可以解决这个问题了。
### HPA 相关
#### 报错
我的 hpa 命令：

``` bash
kubectl autoscale deployment hpa-demo --cpu-percent=10 --min=1 --max=10
```

发现 warning：（有这个 warning 就不能自动扩容了）

``` bash
 HPA was unable to compute the replica count: failed to get cpu utilization:
      missing request for cpu"}]'
```

\![\[Pasted image 20231213160052.png\]\]

回去看部署的 yaml 文件：

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hpa-demo
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```

根据提示应该在部署的时候设置 cpu 的 request（因为 autoscale 的标准设置为 cpu ）

重新部署完 `hpa-demo.yaml`，删除原来的 hpa 后重新输入命令

``` bash
kubectl autoscale deployment hpa-demo --cpu-percent=10 --min=1 --max=10

kubectl describe hpa hpa-demo
```

发现新的问题：

``` bash
Events:
  Type     Reason                        Age   From                       Message
  ----     ------                        ----  ----                       -------
  Warning  FailedGetResourceMetric       8s    horizontal-pod-autoscaler  failed to get cpu utilization: unable to get metrics for resource cpu: no metrics returned from resource metrics API
  Warning  FailedComputeMetricsReplicas  8s    horizontal-pod-autoscaler  invalid metrics (1 invalid out of 1), first error is: failed to get cpu utilization: unable to get metrics for resource cpu: no metrics returned from resource metrics API
```

\![\[Pasted image 20231213160634.png\]\]

推断是 metrics-server 没有正常工作...
原本是想修改 metrics-server 的部署文件输出更多的 log 进行排查，结果重启完就能用了...(没有 warning，是否 autoscale 需要继续测试)

\![\[Pasted image 20231213162128.png\]\]

\![\[Pasted image 20231213161715.png\]\]

> \[!hint\]
> 修改 metrics-server 部署文件获得更多日志输出的办法（同集群 init 的方法 `--v=5` 数字越大输入日志越多）
> \![\[Pasted image 20231213161845.png\]\]

#### 正常运行截图

增大负载进行测试，我们来创建一个 busybox 的 Pod，并且循环访问上面创建的 Pod：
压测的 ip 填对应的 podip

``` bash
$ kubectl run -it --image busybox test-hpa --restart=Never --rm /bin/sh
If you don't see a command prompt, try pressing enter.
/ # while true; do wget -q -O- http://10.244.4.97; done
```

发现超过了设定的 cpu_percent 后开始自动扩容了
\![\[Pasted image 20231213163529.png\]\]

\![\[Pasted image 20231213163817.png\]\]

> Q：但是当负载降低了以后为啥不自动销毁 pod
> A：**从 Kubernetes `v1.12` 版本开始我们可以通过设置 `kube-controller-manager` 组件的 `--horizontal-pod-autoscaler-downscale-stabilization` 参数来设置一个持续时间，用于指定在当前操作完成后，`HPA` 必须等待多长时间才能执行另一次缩放操作**。默认为 5 分钟，也就是默认需要等待 5 分钟后才会开始自动缩放。

**five minutes later**  
发现开始了 terminating 的操作
\![\[Pasted image 20231213164324.png\]\]
