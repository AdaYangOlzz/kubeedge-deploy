## 说在前面
1. **k8s 只需要安装在 master 节点上**，其他的节点都不用  
2. kubeedge 的运行前提是 master 上必须有 k8s  
3. docker 只是用来发布容器 pods 的  
4. **calico 只需要安装在 master 上**，它是节点通信的插件，如果没有这个，master 上安装 kubeedge 的 coredns 会报错。但是，节点上又不需要安装这个，因为 kubeedge 针对这个做了自己的通信机制  
5. 一些插件比如 calico、edgemesh、sedna、metric-service 还有 kuborad 等，都是通过 yaml 文件启动的，所以实际要下载的是k8s 的控制工具 kubeadm 和 kubeedge 的控制工具 keadm。然后提前准备好刚才的 yaml 文件，启动 k8s 和 kubeedge 后，直接根据 yaml 文件容器创建  
6. namespace 可以看作不同的虚拟项目，service 是指定的任务目标文件，pods 是根据 service 或者其他 yaml 文件创建的具体容器  
7. 一个物理节点上可以有很多个 pods，pods 是可操作的最小单位，一个 service 可以设置很多 pods，一个 service 可以包含很多物理节点  
8. 一个 pods 可以看作一个根据 docker 镜像创建的实例  
9. **如果是主机容器创建任务，要设置 dnsPolicy**（很重要）  sedna-lc 的坑点
10. 拉去 docker 镜像的时候，**一定要先去确认架构是否支持**
## 前期准备
前期准备主要是准备一些安装包、设置源、一些 docker 配置等。

### 云边共用

#### 关闭防火墙
```
ufw disable	
```

#### 开启 ipv 4 转发配置 iptables 参数
将桥接的 `IPv4/IPv6 ` 流量传递到 iptables 的链
```
modprobe br_netfilter

cat >> /etc/sysctl.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sysctl -p 
```

#### 禁用 swap 分区
```bash
swapoff -a                                          #临时关闭
sed -ri 's/.*swap.*/#&/' /etc/fstab                 #永久关闭
```

#### 设置主机名
```bash
hostnamectl set-hostname master   
bash

hostnamectl set-hostname edge1
bash

hostnamectl set-hostname edge2
bash
```

#### 设置 DNS
```bash
#不同主机上使用ifconfig查看ip地址
sudo vim /etc/hosts

#根据自己的ip添加
#ip 主机名
192.168.247.128 master
192.168.247.129 edge1
```

#### 安装 docker 并配置
```bash
sudo apt-get install docker.io -y

sudo systemctl start docker
sudo systemctl enable docker

docker --version

# 如果是 ARM64 架构
# 参考 https://blog.csdn.net/qq_34253926/article/details/121629068
```

```bash
sudo vim /etc/docker/daemon.json

# 2024/6/28更新: dockerHub国内镜像都已失效,根据 利用Cloudflare拉取 文档进行配置拉取相关的内容
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

### 云端
####  添加阿里源
```bash
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

```bash
#查看软件版本
apt-cache madison kubeadm

#安装kubeadm、kubelet、kubectl
sudo apt-get update

# k8s和kubeedge版本对应见 [kubeedge/kubeedge: Kubernetes Native Edge Computing Framework (project under CNCF) (github.com)](https://github.com/kubeedge/kubeedge)

sudo apt-get install -y kubelet=1.22.2-00 kubeadm=1.22.2-00 kubectl=1.22.2-00 
#锁定版本
sudo apt-mark hold kubelet kubeadm kubectl
```

#### 下载配置 calico
**官方解释**：因为在整个 kubernetes 集群里，pod 都是分布在不同的主机上的，为了实现这些 pod 的跨主机通信所以我们必须要安装 CNI 网络插件，这里选择 calico 网络。
**实践经验**：在 k8s 集群初始化后，如果不配置 calico 等相关网络插件，会发现 coredns 一直处在 pending 或 containerCreating 状态，通过 describe 会发现提示缺少网络组件

**步骤 1：在 master 上下载配置 calico 网络的 yaml。**
```bash
#注意对应版本，v1.22和v3.20
wget https://docs.projectcalico.org/v3.20/manifests/calico.yaml --no-check-certificate
```

**步骤 2：修改 calico.yaml 里的 pod 网段。**
```bash
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

**步骤 3：提前下载所需要的镜像。**
```bash
#查看此文件用哪些镜像：
grep image calico.yaml

#image: calico/cni:v3.20.6
#image: calico/cni:v3.20.6
#image: calico/pod2daemon-flexvol:v3.20.6
#image: calico/node:v3.20.6
#image: calico/kube-controllers:v3.20.6
```

在 master 节点中下载镜像
```shell
#换成自己的版本
for i in calico/cni:v3.20.6 calico/pod2daemon-flexvol:v3.20.6 calico/node:v3.20.6 calico/kube-controllers:v3.20.6 ; do docker pull $i ; done
```

==calico 只需要在 master 上存在==

 #### 下载metrics-server
 - 用于追踪边缘节点日志
- 官网安装 (会出错, 拉取镜像超时问题)：
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```
![PixPin_2024-06-28_11-04-04](./img/PixPin_2024-06-28_11-04-04.png)

- 本地安装

```bash
# 先下载官方提供的yaml
wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

修改 yaml 的内容，先看官方的版本，然后去 docker hub 找对应的镜像，**并且添加“- --kubelet-insecure-tls”**

修改 `components.yaml` 文件
```vim
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

然后手动 pull 镜像：`docker pull mingyangtech/klogserver:v0.6.4`

>替代的镜像是去 dockerhub 搜 metrics-server

![PixPin_2024-06-28_11-04-04](./img/PixPin_2024-06-28_11-05-36.png)

#### 下载 keadm

用于 kubeedge 安装和使用
进入官网版本链接：[Release KubeEdge v1.9.2 release · kubeedge/kubeedge (github.com)](https://github.com/kubeedge/kubeedge/releases/tag/v1.9.2)
```bash
#查看自己的架构
uname -u

#下载对应版本以及架构
https://github.com/kubeedge/kubeedge/releases/download/v1.9.2/keadm-v1.9.2-linux-amd64.tar.gz
#解压
tar zxvf keadm-v1.9.2-linux-amd64.tar.gz
#添加执行权限
chmod +x keadm-v1.9.2-linux-amd64/keadm/keadm 
#移动目录
cp keadm-v1.9.2-linux-amd64/keadm/keadm /usr/local/bin/
```

#### 下载 Edgemesh
```bash
git clone https://github.com/kubeedge/edgemesh.git
```

#### 下载 Sedna
##### 官方下载（会超时）
```bash
curl https://raw.githubusercontent.com/kubeedge/sedna/main/scripts/installation/install.sh | SEDNA_ACTION=create bash -
```

![5](./img/5.png)

##### 本地下载

- 下载 `install.sh` 文件（不要保存到 build 中，可以永久使用）
```bash
wget https://raw.githubusercontent.com/kubeedge/sedna/main/scripts/installation/install.sh
```

- 相关的修改（不要保存到 build 中，可以永久使用）
```bash
#改名字
mv install.sh offline-install.sh

#修改文件中的一下内容（sh的语法）
1.第34行：删除  trap "rm -rf '%TMP_DIR'" EXIT
2.第415行：删除  prepare
```


>[!warning]
>记得看 `install.sh` 中 sedna 的版本，去 docker hub 中看镜像是否适用于你的架构!! (主要针对 arm)


- 下载 sedna 的官网 github（不要保存到 build 中，可以永久使用）
官方地址：[http://github.com/kubeedge/sed](https://link.zhihu.com/?target=http%3A//github.com/kubeedge/sedna/tree/main/build)
```bash
git clone https://github.com/kubeedge/sedna.git
```

**因为 sedna 我们做过一定的修改，所以 `git clone https://github.com/AdaYangOlzz/sedna-modified.git`，并且脚本的内容做了一定的修改。并且记得把 lc 的使用主机网络去掉**

完整版 Sedna 安装脚本：
```bash file:offline-install.sh
#!/bin/bash

# Copyright 2021 The KubeEdge Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# Influential env vars:
#
# SEDNA_ACTION    | optional | 'create'/'clean', default is 'create'
# SEDNA_VERSION   | optional | The Sedna version to be installed.
#                              if not specified, it will get latest release version.
# SEDNA_ROOT      | optional | The Sedna offline directory

set -o errexit
set -o nounset
set -o pipefail

TMP_DIR='/opt/sedna'
SEDNA_ROOT='/home/hx/yby/software/kubeedge-1.9.2/sedna'

DEFAULT_SEDNA_VERSION=v0.3.1



get_latest_version() {
  # get Sedna latest release version
  local repo=kubeedge/sedna
  # output of this latest page:
  # ...
  # "tag_name": "v1.0.0",
  # ...
  {
    curl -s https://api.github.com/repos/$repo/releases/latest |
    awk '/"tag_name":/&&$0=$2' |
    sed 's/[",]//g'
  } || echo $DEFAULT_SEDNA_VERSION # fallback
}

: ${SEDNA_VERSION:=$(get_latest_version)}
SEDNA_VERSION=v${SEDNA_VERSION#v}

_download_yamls() {

  yaml_dir=$1
  mkdir -p ${SEDNA_ROOT}/$yaml_dir
  cd ${SEDNA_ROOT}/$yaml_dir
  for yaml in ${yaml_files[@]}; do
    # the yaml file already exists, no need to download
    [ -e "$yaml" ] && continue

    echo downloading $yaml into ${SEDNA_ROOT}/$yaml_dir
    local try_times=30 i=1 timeout=2
    while ! timeout ${timeout}s curl -sSO https://raw.githubusercontent.com/kubeedge/sedna/main/$yaml_dir/$yaml; do
      ((++i>try_times)) && {
        echo timeout to download $yaml
        exit 2
      }
      echo -en "retrying to download $yaml after $[i*timeout] seconds...\r"
    done
  done
}

download_yamls() {
  yaml_files=(
  sedna.io_datasets.yaml
  sedna.io_federatedlearningjobs.yaml
  sedna.io_incrementallearningjobs.yaml
  sedna.io_jointinferenceservices.yaml
  sedna.io_lifelonglearningjobs.yaml
  sedna.io_models.yaml
  )
  _download_yamls build/crds
  yaml_files=(
    gm.yaml
  )
  _download_yamls build/gm/rbac
}

prepare_install(){
  # need to create the namespace first
  kubectl create ns sedna
}

prepare() {
  mkdir -p ${SEDNA_ROOT}

  # we only need build directory
  # here don't use git clone because of large vendor directory
  download_yamls
}

cleanup(){
  kubectl delete ns sedna
}

create_crds() {
  cd ${SEDNA_ROOT}
  kubectl create -f build/crds
}

delete_crds() {
  cd ${SEDNA_ROOT}
  kubectl delete -f build/crds --timeout=90s
}

get_service_address() {
  local service=$1
  local port=$(kubectl -n sedna get svc $service -ojsonpath='{.spec.ports[0].port}')

  # <service-name>.<namespace>:<port>
  echo $service.sedna:$port
}

create_kb(){
  cd ${SEDNA_ROOT}

  kubectl $action -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: kb
  namespace: sedna
spec:
  selector:
    sedna: kb
  type: ClusterIP
  ports:
    - protocol: TCP
      port: 9020
      targetPort: 9020
      name: "tcp-0"  # required by edgemesh, to clean
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kb
  labels:
    sedna: kb
  namespace: sedna
spec:
  replicas: 1
  selector:
    matchLabels:
      sedna: kb
  template:
    metadata:
      labels:
        sedna: kb
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-role.kubernetes.io/edge
                operator: DoesNotExist
      serviceAccountName: sedna
      containers:
      - name: kb
        imagePullPolicy: IfNotPresent
        image: kubeedge/sedna-kb:$SEDNA_VERSION
        env:
          - name: KB_URL
            value: "sqlite:///db/kb.sqlite3"
        volumeMounts:
        - name: kb-url
          mountPath: /db
        resources:
          requests:
            memory: 256Mi
            cpu: 100m
          limits:
            memory: 512Mi
      volumes:
        - name: kb-url
          hostPath:
            path: /opt/kb-data
            type: DirectoryOrCreate
EOF
}

prepare_gm_config_map() {

  KB_ADDRESS=$(get_service_address kb)

  cm_name=${1:-gm-config}
  config_file=${TMP_DIR}/${2:-gm.yaml}

  if [ -n "${SEDNA_GM_CONFIG:-}" ] && [ -f "${SEDNA_GM_CONFIG}" ] ; then
    cp "$SEDNA_GM_CONFIG" $config_file
  else
    cat > $config_file << EOF
kubeConfig: ""
master: ""
namespace: ""
websocket:
  address: 0.0.0.0
  port: 9000
localController:
  server: http://localhost:${SEDNA_LC_BIND_PORT:-9100}
knowledgeBaseServer:
  server: http://$KB_ADDRESS
EOF
  fi

  kubectl $action -n sedna configmap $cm_name --from-file=$config_file
}

create_gm() {

  cd ${SEDNA_ROOT}

  kubectl create -f build/gm/rbac/

  cm_name=gm-config
  config_file_name=gm.yaml
  prepare_gm_config_map $cm_name $config_file_name


  kubectl $action -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: gm
  namespace: sedna
spec:
  selector:
    sedna: gm
  type: ClusterIP
  ports:
    - protocol: TCP
      port: 9000
      targetPort: 9000
      name: "tcp-0"  # required by edgemesh, to clean
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gm
  labels:
    sedna: gm
  namespace: sedna
spec:
  replicas: 1
  selector:
    matchLabels:
      sedna: gm
  template:
    metadata:
      labels:
        sedna: gm
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-role.kubernetes.io/edge
                operator: DoesNotExist
      serviceAccountName: sedna
      containers:
      - name: gm
        image: dislabaiot.xyz/adayoung/sedna-gm:v0.3.11
        command: ["sedna-gm", "--config", "/config/$config_file_name", "-v2"]
        volumeMounts:
        - name: gm-config
          mountPath: /config
        resources:
          requests:
            memory: 32Mi
            cpu: 100m
          limits:
            memory: 256Mi
      volumes:
        - name: gm-config
          configMap:
            name: $cm_name
EOF
}

delete_gm() {
  cd ${SEDNA_ROOT}

  kubectl delete -f build/gm/rbac/

  # no need to clean gm deployment alone
}

create_lc() {

  GM_ADDRESS=$(get_service_address gm)

  kubectl $action -f- <<EOF
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    sedna: lc
  name: lc
  namespace: sedna
spec:
  selector:
    matchLabels:
      sedna: lc
  template:
    metadata:
      labels:
        sedna: lc
    spec:
      containers:
        - name: lc
          image: dislabaiot.xyz/adayoung/sedna-lc:v0.3.11
          env:
            - name: GM_ADDRESS
              value: $GM_ADDRESS
            - name: BIND_PORT
              value: "${SEDNA_LC_BIND_PORT:-9100}"
            - name: NODENAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: ROOTFS_MOUNT_DIR
              # the value of ROOTFS_MOUNT_DIR is same with the mount path of volume
              value: /rootfs
          resources:
            requests:
              memory: 32Mi
              cpu: 100m
            limits:
              memory: 128Mi
          volumeMounts:
            - name: localcontroller
              mountPath: /rootfs
      volumes:
        - name: localcontroller
          hostPath:
            path: /
      restartPolicy: Always
EOF
}

delete_lc() {
  # ns would be deleted in delete_gm
  # so no need to clean lc alone
  return
}

wait_ok() {
  echo "Waiting control components to be ready..."
  kubectl -n sedna wait --for=condition=available --timeout=600s deployment/gm
  kubectl -n sedna wait pod --for=condition=Ready --selector=sedna
  kubectl -n sedna get pod
}

delete_pods() {
  # in case some nodes are not ready, here delete with a 60s timeout, otherwise force delete these
  kubectl -n sedna delete pod --all --timeout=60s || kubectl -n sedna delete pod --all --force --grace-period=0
}

check_kubectl () {
  kubectl get pod >/dev/null
}

check_action() {
  action=${SEDNA_ACTION:-create}
  support_action_list="create delete"
  if ! echo "$support_action_list" | grep -w -q "$action"; then
    echo "\`$action\` not in support action list: create/delete!" >&2
    echo "You need to specify it by setting $(red_text SEDNA_ACTION) environment variable when running this script!" >&2
    exit 2
  fi

}

do_check() {
  check_kubectl
  check_action
}

show_debug_infos() {
  cat - <<EOF
Sedna is $(green_text running):
See GM status: kubectl -n sedna get deploy
See LC status: kubectl -n sedna get ds lc
See Pod status: kubectl -n sedna get pod
EOF
}

NO_COLOR='\033[0m'
RED='\033[0;31m'
GREEN='\033[0;32m'
green_text() {
  echo -ne "$GREEN$@$NO_COLOR"
}

red_text() {
  echo -ne "$RED$@$NO_COLOR"
}

do_check

case "$action" in
  create)
    echo "Installing Sedna $SEDNA_VERSION..."
    prepare_install
    create_crds
    create_kb
    create_gm
    create_lc
    wait_ok
    show_debug_infos
    ;;

  delete)
    # no errexit when fail to clean
    set +o errexit
    delete_pods
    delete_gm
    delete_lc
    delete_crds
    cleanup
    echo "$(green_text Sedna is uninstalled successfully)"
    ;;
esac

```

![4](./img/4.png)

![3](./img/3.png)

### 边端

#### keadm
```bash
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


## 正式安装
### Kubernetes 启动(master)
该步可能会出现 [[#问题一：kube-proxy 报 iptables 问题]] 、 [[#问题二：calico 和 coredns 一直处于初始化]]和[[#问题三：metrics-server一直无法成功]]
##### 删除从前的信息（如果是第一次可跳过）
```bash
swapoff -a
kubeadm reset
systemctl daemon-reload
systemctl restart docker kubelet
rm -rf $HOME/.kube/config

rm -f /etc/kubernetes/kubelet.conf    #删除k8s配置文件
rm -f /etc/kubernetes/pki/ca.crt    #删除K8S证书

rm -rf /etc/cni/net.d/  # 删除网络配置

iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
```

##### 初始化 k8s 集群

```bash
kubeadm init --pod-network-cidr 10.244.0.0/16 \
--image-repository registry.cn-hangzhou.aliyuncs.com/google_containers

#上一步创建成功后，会提示：主节点执行提供的代码
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

#如果只有一个点，需要容纳污点
kubectl taint nodes --all node-role.kubernetes.io/master node-role.kubernetes.io/master-

```

##### 启动 calico
```bash
一定要执行，不然coredns无法初始化
kubectl apply -f calico.yaml

#检查启动状态，需要都running，除了calico可以不用
kubectl get pods -A

```

##### 对 calico 和 kube-proxy 进行配置
Edge 节点加入时可能会自动部署 calico-node 和 kube-proxy，kube-proxy 会部署成功（但是 edgecore 的 log 会提示不应该部署 kube-proxy），calico 会初始化失败。为了避免上述情况，做如下操作：

```bash
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

此时记得 `kubectl get pods -A` 看看所有的 pod 是否都在 running。最后可以 logs 一下，看看组件内部是否有报错。

<img src="./img/kube-proxy.png" alt="kube-proxy" style="zoom:60%;" />

##### 启动 metrics-server

```bash
# 进入metrics-server下载目录
kubectl apply -f components.yaml

# 检验是否部署成功
$ kubectl top nodes
```

### KubeEdge 启动

#### master
可能出现[[#问题四：10002 already in use]]
##### 重启（如果第一次可以跳过）

```bash
keadm reset
```

##### 启动 cloudcore

通过 keadm 启动

```bash
#注意修改--advertise-address的ip地址
keadm init --advertise-address=114.212.81.11 --kubeedge-version=1.9.2


#打开转发路由
kubectl get cm tunnelport -n kubeedge -o yaml
#找到10350 或者10351
#set the rule of trans，设置自己的端口
export CLOUDCOREIPS=xxx.xxx.xxx.xxx

# dport的内容对应tunnelport
iptables -t nat -A OUTPUT -p tcp --dport 10351 -j DNAT --to $CLOUDCOREIPS:10003

cd /etc/kubeedge/
cp $GOPATH/src/github.com/kubeedge/kubeedge/build/tools/certgen.sh /etc/kubeedge/

bash certgen.sh stream

```

##### cloudcore 配置

```
$ vim /etc/kubeedge/config/cloudcore.yaml

modules:
  ..
  cloudStream:
    enable: true
    streamPort: 10003
  ..
  dynamicController:
    enable: true
```

重启 cloudcore：`sudo systemctl restart cloudcore.service`

记得 `journalclt -u cloudcore.service -xe` 看看是否正常运行没有报错
#### Node
可能出现[[#问题五]]
##### reset（如果第一次，请跳过）

```bash
#重启
keadm reset

#重新加入时，报错存在文件夹，直接删除
rm -rf /etc/kubeedge

#docker容器占用(通常是mqtt)
docker ps -a
docker stop mqtt
docker rm mqtt
```

##### 启动

```bash
#注意这里是要加入master的IP地址，token是master上获取的
keadm join --cloudcore-ipport=114.212.81.11:10000 --kubeedge-version=1.9.2 --token=9e1832528ae701aba2c4f7dfb49183ab2487e874c8090e68c19c95880cd93b50.eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE3MTk1NjU4MzF9.1B4su4QwvQy_ZCPs-PIyDT9ixsDozfN1oG4vX59tKDs
```

```bash
#加入后，先查看信息
journalctl -u edgecore.service -f
```

### EdgeMesh 启动
#### 前期准备

可能出现[[#问题六：缺少 TLSStreamCertFile]]

**步骤 1**: 去除 K8s master 节点的污点
```bash
kubectl taint nodes --all node-role.kubernetes.io/master-
```

**步骤 2**: 给 Kubernetes API 服务添加过滤标签
```bash
kubectl label services kubernetes service.edgemesh.kubeedge.io/service-proxy-name=""
```

- **步骤 3**: 启用 KubeEdge 的边缘 Kube-API 端点服务**important**  
  
- 在云端，开启 dynamicController 模块，配置完成后，需要重启 cloudcore `systemctl restart cloudcore`（这一步在 cloudcore 中配置过可跳过）
```bash
vim /etc/kubeedge/config/cloudcore.yaml
modules:
  ..
  cloudStream:
    enable: true
    streamPort: 10003
  ..
  dynamicController:
    enable: true
```

- 在边端，打开 metaServer 模块和 edgeStream
```bash
$ vim /etc/kubeedge/config/edgecore.yaml
edgeStream:  
	enable: true  
	handshakeTimeout: 30  
	readDeadline: 15  
	server: 192.168.0.139:10004  
	tlsTunnelCAFile: /etc/kubeedge/ca/rootCA.crt  
	tlsTunnelCertFile: /etc/kubeedge/certs/server.crt  
	tlsTunnelPrivateKeyFile: /etc/kubeedge/certs/server.key  
	writeDeadline: 15
metaManager:
    metaServer:
      enable: true
...

#重启
systemctl restart edgecore.service
#检查状况
journalctl -u edgecore.service -f
#确定正常运行
```



**步骤 4**: 最后，在边缘节点，测试边缘 Kube-API 端点功能是否正常

```bash
# 边缘节点上
curl 127.0.0.1:10550/api/v1/services
```

如果返回值是空列表，或者响应时长很久（接近 10 s）才拿到返回值，说明你的配置可能有误，请仔细检查。

![api](./img/api.png)



#### 启动 EdgeMesh

此处问题重灾区。~~请熟读背诵 Q&A~~
>[!cite]
> [快速上手 | EdgeMesh](https://edgemesh.netlify.app/zh/guide/#%E6%89%8B%E5%8A%A8%E5%AE%89%E8%A3%85)
> [全网最全EdgeMesh Q&A手册 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/585749690)


```
# 之前有下好的
cd edgemesh
```

- crd 文件部署
```bash
kubectl apply -f build/crds/istio/
```

- 部署 edgemesh-agent
```bash
kubectl apply -f build/agent/resources/
```

- 配置 edge
```
vim /etc/kubeedge/config/edgecore.yaml

edged:
    clusterDNS: 169.254.96.16
    clusterDomain: cluster.local
```

![edgecore](./img/edgecore1.png)

![edgecore](./img/edgecore2.png)



#### 正常运行状态

iptables：

![iptables](./img/iptables.png)



云端logs：

![cloud-agent](./img/cloud-agent.png)



边端logs：

![edge-agent.png](./img/edge-agent.png)



### 安装 Sedna

#### 安装
```bash
#直接进入之前准备好的文件目录运行
SEDNA_ACTION=create bash - offline-install.sh

# 卸载
SEDNA_ACTION=delete bash - offline-install.sh
```

安装完后需要着重去查看 lc 是否连接 gm 成功。问题重灾区 again

#### 正常运行截图

![lc](./img/lc.png)



![gm](./img/gm.png)

### GPU 支持

其实这是探索很艰难的一个过程...先讲最终结果：
#### 开启 GPU 支持
运行下面命令即可
```bash
kubectl apply -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.13.0/nvidia-device-plugin.yml
```

>[!hint]
>在边端 `vim /etc/kubeedge/config/edgecore.yaml`
>将修改使得 `devicePluginEnabled: true`。本身这个插件是提供给 `k8s` 使用的，本地节点采用的 kubelet 进行管理，但是 kubeedge 采用 edgecore，打开 devicePluginEnabled 才可以



edge的docker/daemon.json

```json
{
    "default-runtime": "nvidia",
    "runtimes": {
        "nvidia": {
            "path": "nvidia-container-runtime",
            "runtimeArgs": []
        }
    },
    "registry-mirrors": ["xxxxx"],
    "insecure-registries": ["xxxx"] // 设置的内网仓库
}

```



cloud的docker/daemon.json

```json
{
    "exec-opts": [
        "native.cgroupdriver=systemd"
    ],
    "registry-mirrors": [
        "https://b9pmyelo.mirror.aliyuncs.com",
        "xxxx" // cloudflare 拉取 镜像
    ],
    "default-runtime": "nvidia",
    "runtimes": {
        "nvidia": {
            "args": [],
            "path": "nvidia-container-runtime"
        }
    }
}

```





#### 正常运行截图

云端：

<img src="./img/PixPin_2024-06-28_11-37-51.png" alt="PixPin_2024-06-28_11-37-51.png" style="zoom:67%;" />

![PixPin_2024-06-28_11-38-18.png](./img/PixPin_2024-06-28_11-38-18.png)

边端：

![PixPin_2024-06-28_11-39-07.png](./img/PixPin_2024-06-28_11-39-07.png)

![PixPin_2024-06-28_11-40-16.png](./img/PixPin_2024-06-28_11-40-16.png)



```bash
kubectl run -i -t nvidia --image=jitteam/devicequery
```

![PixPin_2024-06-28_11-40-44.png](./img/PixPin_2024-06-28_11-40-44.png)



#### 可跳过的探索历程...

just 记录一下自己的辛苦💔
`k8s` 使用 GPU 需要 nvidia-device-plugin 插件支持
```bash
kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.7.3/nvidia-device-plugin.yml
 
```

但是会发现问题：
![PixPin_2024-06-28_11-33-06.png](./img/PixPin_2024-06-28_11-33-06.png)
这个问题也遇见了很多次了，大概就是架构问题导致无法运行，需要重新编译 docker，于是开始尝试

```bash
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
![PixPin_2024-06-28_11-33-28.png](./img/PixPin_2024-06-28_11-33-28.png)

![PixPin_2024-06-28_11-33-48.png](./img/PixPin_2024-06-28_11-33-48.png)

![PixPin_2024-06-28_11-34-10.png](./img/PixPin_2024-06-28_11-34-10.png)

此问题的解决：只需要更新动态库配置文件和将它报错的路径链接过去

```bash
root@edge2:/software# ldconfig
root@edge2:/software# whereis /sbin/ldconfig
ldconfig: /sbin/ldconfig
root@edge2:/software# ln -s /sbin/ldconfig /sbin/ldconfig.real
root@edge2:/software# sudo systemctl daemon-reload
root@edge2:/software# sudo systemctl restart docker

```

然而问题只会一个接一个的出现：

![PixPin_2024-06-28_11-34-34.png](./img/PixPin_2024-06-28_11-34-34.png)

```bash
root@cloud:/home/guest/yby/kubeedge/multiedge/test-yaml/gpu# kubectl logs nvidia-device-plugin-daemonset-edge-65lbm -n kube-system
2024/01/04 06:08:15 Loading NVML
2024/01/04 06:08:15 Failed to initialize NVML: could not load NVML library.
2024/01/04 06:08:15 If this is a GPU node, did you set the docker default runtime to `nvidia`?
2024/01/04 06:08:15 You can check the prerequisites at: https://github.com/NVIDIA/k8s-device-plugin#prerequisites
2024/01/04 06:08:15 You can learn how to set the runtime at: https://github.com/NVIDIA/k8s-device-plugin#quick-start
2024/01/04 06:08:15 If this is not a GPU node, you should set up a toleration or nodeSelector to only deploy this plugin on GPU nodes
2024/01/04 06:08:15 Error: failed to initialize NVML: could not load NVML library

```

>[!quote]
> [nvidia-smi installation on Jetson TX2 - Jetson & Embedded Systems / Jetson TX2 - NVIDIA Developer Forums](https://forums.developer.nvidia.com/t/nvidia-smi-installation-on-jetson-tx2/111495)

![PixPin_2024-06-28_11-34-59.png](./img/PixPin_2024-06-28_11-34-59.png)

官方回复 jetson 系列不能用 NVML...**但是柳暗花明又一村!!这个插件开发者兼容了 jetson 板子！**
>[!quote]
> [Add support for arm64 and Jetson boards. (!20) · 合并请求 · nvidia / kubernetes / device-plugin · GitLab](https://gitlab.com/nvidia/kubernetes/device-plugin/-/merge_requests/20)
> [NVIDIA/k8s-device-plugin at v0.13.0 (github.com)](https://github.com/NVIDIA/k8s-device-plugin/tree/v0.13.0)

然后就是 [[#开启 GPU 支持]]的部分了


## 可能出现的问题
### 问题一：kube-proxy 报 iptables 问题
```
E0627 09:28:54.054930 1 proxier.go:1598] Failed to execute iptables-restore: exit status 1 (iptables-restore: line 86 failed ) I0627 09:28:54.054962 1 proxier.go:879] Sync failed; retrying in 30s
```

解决：直接清理 iptables
```
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
```

### 问题二：calico 和 coredns 一直处于初始化
用 `kubectl describe <podname>` 应该会有 failed 的报错，大致内容是跟 network 和 sandbox 相关的 
```
Failed to create pod sandbox: rpc error: code = Unknown desc = [failed to set up sandbox container "7f5b66ebecdfc2c206027a2afcb9d1a58ec5db1a6a10a91d4d60c0079236e401" network for pod "calico-kube-controllers-577f77cb5c-99t8z": networkPlugin cni failed to set up pod "calico-kube-controllers-577f77cb5c-99t8z_kube-system" network: error getting ClusterInformation: Get "https://[10.96.0.1]:443/apis/crd.projectcalico.org/v1/clusterinformations/default": dial tcp 10.96.0. 1:443: i/o timeout, failed to clean up sandbox container "7f5b66ebecdfc2c206027a2afcb9d1a58ec5db1a6a10a91d4d60c0079236e401" network for pod "calico-kube-controllers-577f77cb5c-99t8z": networkPlugin cni failed to teardown pod "calico-kube-controllers-577f77cb5c-99t8z_kube-system" network: error getting ClusterInformation: Get "https://[10.96.0.1]:443/apis/crd.projectcalico.org/v1/clusterinformations/default": dial tcp 10.96.0. 1:443: i/o timeout]
```

原因：应该不是第一次初始化 k8s 集群，所以会遇到这样的问题。出现的原因是之前没有删除 k8s 的网络配置（文档里的每一步都有它存在的意义...）

解决：
```bash
rm -rf /etc/cni/net.d/  # 删除网络配置
# maybe需要重新init一下
```

### 问题三：metrics-server一直无法成功
原因：master 没有加污点
解决：`kubectl taint nodes --all node-role.kubernetes.io/master node-role.kubernetes.io/master-`

### 问题四：10002 already in use
`journalclt -u cloudcore.service -xe` 时看到 xxx already in use

原因：应该是之前的记录没有清理干净
解决：找到占用端口的进程，直接 Kill 即可

```bash
lsof -i:xxxx
kill xxxxx
```

### 问题五：edgecore 符号链接已存在
```
execute keadm command failed:  failed to exec 'bash -c sudo ln /etc/kubeedge/edgecore.service /etc/systemd/system/edgecore.service && sudo systemctl daemon-reload && sudo systemctl enable edgecore && sudo systemctl start edgecore', err: ln: failed to create hard link '/etc/systemd/system/edgecore.service': File exists
, err: exit status 1
```

在尝试创建符号链接时，目标路径已经存在，因此无法创建。这通常是因为 `edgecore.service` 已经存在于 `/etc/systemd/system/` 目录中
```bash
sudo rm /etc/systemd/system/edgecore.service
```

### 问题六：缺少 TLSStreamCertFile
```bash
 TLSStreamPrivateKeyFile: Invalid value: "/etc/kubeedge/certs/stream.key": TLSStreamPrivateKeyFile not exist
12月 14 23:02:23 cloud.kubeedge cloudcore[196229]:   TLSStreamCertFile: Invalid value: "/etc/kubeedge/certs/stream.crt": TLSStreamCertFile not exist
12月 14 23:02:23 cloud.kubeedge cloudcore[196229]:   TLSStreamCAFile: Invalid value: "/etc/kubeedge/ca/streamCA.crt": TLSStreamCAFile not exist
12月 14 23:02:23 cloud.kubeedge cloudcore[196229]: ]

```

查看 `/etc/kubeedge` 下是否有 `certgen.sh` 并且 `bash certgen.sh stream`

### 问题七：edgemesh 的 log 边边互联成功，云边无法连接
#### 排查
先复习一下**定位模型**，确定**被访问节点**上的 edgemesh-agent(右)容器是否存在、是否处于正常运行中。

**这个情况是非常经常出现的**，因为 master 节点一般都有污点，会驱逐其他 pod，进而导致 edgemesh-agent 部署不上去。这种情况可以通过去除节点污点，使 edgemesh-agent 部署上去解决。

如果访问节点和被访问节点的 edgemesh-agent 都正常启动了，但是还报这个错误，可能是因为访问节点和被访问节点没有互相发现导致，请这样排查：

1. 首先每个节点上的 edgemesh-agent 都具有 peer ID，比如

```bash

edge2: 
I'm {12D3KooWPpY4GqqNF3sLC397fMz5ZZfxmtMTNa1gLYFopWbHxZDt: [/ip4/127.0.0.1/tcp/20006 /ip4/192.168.1.4/tcp/20006]}

edge1.kubeedge:
I'm {12D3KooWFz1dKY8L3JC8wAY6sJ5MswvPEGKysPCfcaGxFmeH7wkz: [/ip4/127.0.0.1/tcp/20006 /ip4/192.168.1.2/tcp/20006]}

注意：
a. peer ID是根据节点名称哈希出来的，相同的节点名称会哈希出相同的peer ID
b. 另外，节点名称不是服务器名称，是k8s node name，请用kubectl get nodes查看
```

2.如果访问节点和被访问节点处于同一个局域网内（**所有节点应该具备内网 IP（10.0.0.0/8、172.16.0.0/12、192.168.0.0/16**），请看[全网最全EdgeMesh Q&A手册 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/585749690)**问题十二**同一个局域网内 edgemesh-agent 互相发现对方时的日志是 `[MDNS] Discovery found peer: <被访问端peer ID: [被访问端IP列表(可能会包含中继节点IP)]>` ^d3939d

3.如果访问节点和被访问节点跨子网，这时候应该看看 relayNodes 设置的正不正确，为什么中继节点没办法协助两个节点交换 peer 信息。详细材料请阅读：[KubeEdge EdgeMesh 高可用架构详解](https://link.zhihu.com/?target=https%3A//mp.weixin.qq.com/s/4whnkMM9oOaWRsI1ICsvSA)。跨子网的 edgemesh-agent 互相发现对方时的日志是 `[DHT] Discovery found peer: <被访问端peer ID: [被访问端IP列表(可能会包含中继节点IP)]>`（适用于我的情况）

#### 解决
在部署 edgemesh 进行 `kubectl apply -f build/agent/resources/` 操作时，修改 04-configmap，添加 relayNode（根本原因在于，不符合“访问节点和被访问节点处于同一个局域网内”，所以需要添加 relayNode）

<img src="./img/relayNode.png" alt="relayNode.png" style="zoom:67%;" />

### 问题八：master 的gpu 存在但是找不到 gpu 资源
主要针对的是服务器的情况，可以使用 `nvidia-smi` 查看显卡情况。

>[!quote]
> [Installing the NVIDIA Container Toolkit — NVIDIA Container Toolkit 1.14.3 documentation](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html#configuration)

按上述内容进行配置，然后还需要 `vim /etc/docker/daemon.json`，添加 default-runtime。按照 quote 的内容设置后会有“runtimes”但是 default-runtime 不会设置，可能会导致找不到 GPU 资源
```
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

```

### 问题九：jeston 的 gpu 存在但是找不到 gpu 资源
理论上 `k8s-device-plugin` 已经支持了 tegra 即 jetson 系列板子，会在查看 GPU 之前判断是否是 tegra 架构，如果是则采用 tegra 下查看 GPU 的方式（原因在 [[#GPU 支持]]里 quote 过了 ），但是很奇怪的是明明是 tegra 的架构却没有检测到：
```bash
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

![PixPin_2024-06-28_11-28-29.png](./img/PixPin_2024-06-28_11-28-29.png)

```bash
$ dpkg -l '*nvidia*'
```

![PixPin_2024-06-28_11-29-20.png](./img/PixPin_2024-06-28_11-29-20.png)

![PixPin_2024-06-28_11-29-40.png](./img/PixPin_2024-06-28_11-29-40.png)

>[!quote]
> [Plug in does not detect Tegra device Jetson Nano · Issue #377 · NVIDIA/k8s-device-plugin (github.com)](https://github.com/NVIDIA/k8s-device-plugin/issues/377)
>
>Note that looking at the initial logs that you provided you may have been using `v1.7.0` of the NVIDIA Container Toolkit. This is quite an old version and we greatly improved our support for Tegra-based systems with the `v1.10.0` release. It should also be noted that in order to use the GPU Device Plugin on Tegra-based systems (specifically targetting the integrated GPUs) at least `v1.11.0` of the NVIDIA Container Toolkit is required.
>
>There are no Tegra-specific changes in the `v1.12.0` release, so using the `v1.11.0` release should be sufficient in this case.

那么应该需要升级**NVIDIA Container Toolkit**

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt-get update

sudo apt-get install -y nvidia-container-toolkit
```

### 问题十：lc127.0.0. 53:53 no such host/connection refused
原因：解析 gm.sedna 这个域名失败
解决：
1. 如果安装 sedna 脚本有将 hostNetwork 去掉，则检查 edgecore.yaml 的 clusterDNS 部分，**着重注意是否没有二次设置然后被后设置的覆盖掉**
2. 如果没有将 hostNetwork 去掉，则将宿主机的 `/etc/resolv.conf` 添加 `nameserver 169.254.96.16`

### 问题十一：169.254.96.16:no such host
检查 edgemesh 的配置是否正确：
1. 检查 iptables 的链的顺序如[混合代理 | EdgeMesh](https://edgemesh.netlify.app/zh/advanced/hybird-proxy.html) 所示
2. 着重检查 clusterDNS



### 问题十二： `kubectl logs <pod-name>` 超时

![timeout](./img/timeout.png)

原因：借鉴 [Kubernetes 边缘节点抓不到监控指标？试试这个方法！ - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/379962934)

![PixPin_2024-06-28_11-24-28.png](./img/PixPin_2024-06-28_11-24-28.png)

可以发现报错访问的端口就是 10350，而在 kubeedge 中 10350 应该会进行转发，所以应该是 cloudcore 的设置问题。

解决：[Enable Kubectl logs/exec to debug pods on the edge | KubeEdge](https://kubeedge.io/docs/advanced/debug/) 根据这个链接设置即可。



### 问题十三： `kubectl logs <pod-name>` 卡住 

可能的原因：之前 `kubectl logs` 时未结束就 ctrl+c 结束了导致后续卡住
解决：重启 edgecore/cloudcore `systemctl restart edgecore.service`



### 问题十四：CloudCore报certficate错误

![PixPin_2024-06-28_11-26-15.png](./img/PixPin_2024-06-28_11-26-15.png)

原因：因为是重装，主节点 token 变了，但是边缘节点一直以过去的 token 尝试进行连接

解决：边端用新的 token 连接就好



###  问题十五：删除命名空间卡在 terminating

理论上一直等待应该是可以的(但是我等了半个钟也没成功啊!!)
**方法一**，但是没啥用，依旧卡住

```bash
kubectl delete ns sedna --force --grace-period=0
```

**方法二**：
```bash
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

### 问题十六：强制删除 pod 之后部署不成功

#### 问题描述
因为现在 edge 的 pod 是通过创建 deployment 由 deployment 进行创建，但是通过 `kubectl delete deploy <deploy-name>` 删除 deployment 后，pod 一直卡在了 terminating 状态，于是采用了 `kubectl delete pod edgeworker-deployment-7g5hs-58dffc5cd7-b77wz --force --grace-period=0` 命令进行了删除。
然后发现重新部署时候发现 assigned to edge 的 pod 都卡在 pending 状态。

#### 解决
因为--force 是不会实际终止运行的，所以本身原来的 docker 可能还在运行，现在的做法是手动去对应的边缘节点上删除对应的容器（包括 pause，关于 pause 可以看这篇文章[大白话 K8S（03）：从 Pause 容器理解 Pod 的本质 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/464712164)），然后重启 edgecore: `systemctl restart edgecore.service`

### 问题十七：删除 deployment、pod 等，容器依旧自动重启
```
 journalctl -u edgecore.service  -f
```

![[Pasted image 20240313150604.png]]

解决：重启 edgecore
```
systemctl restart edgecore.service
```

### 问题十八：大面积 Evicted（disk pressure）
#### 原因
- node 上的 kubelet 负责采集资源占用数据，并和预先设置的 threshold 值进行比较，如果超过 threshold 值，kubelet 会杀掉一些 Pod 来回收相关资源，[K8sg官网解读kubernetes配置资源不足处理](https://links.jianshu.com/go?to=https%3A%2F%2Fkubernetes.io%2Fdocs%2Ftasks%2Fadminister-cluster%2Fout-of-resource%2F)
  
- 默认启动时，node 的可用空间低于15%的时候，该节点上讲会执行 eviction 操作，由于磁盘已经达到了85%,在怎么驱逐也无法正常启动就会一直重启，Pod 状态也是 pending 中

 #### 临时解决方法
- 修改配置文件增加传参数,添加此配置项`--eviction-hard=nodefs.available<5%`
```bash
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
```bash
vim /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml --eviction-hard=nodefs.available<5%"

最后添加上--eviction-hard=nodefs.available<5%
```

然后重启 kubelet
```bash
systemctl daemon-reload
systemctl  restart kubelet
```

会发现可以正常部署了(但是只是急救一下，磁盘空间感觉还是得清理下)