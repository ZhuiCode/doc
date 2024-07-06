注意k8s和kubeedge的版本对应在https://github.com/kubeedge/kubeedge有详细说明

  |    |K8s1.20|K8s 1.21|K8s1.22|K8s 1.23|K8s 1.24|	K8s1.25|K8s1.26|K8s 1.27|
   ------
  |KubeEdge 1.12| 	✓	✓	✓	-	-	-	-	-

  KubeEdge 1.13	+	✓	✓	✓	-	-	-	-

  KubeEdge 1.14	+	+	✓	✓	✓	-	-	-

  KubeEdge 1.15	+	+	+	+	✓	✓	✓	-

  KubeEdge HEAD (master)	+	+	+	+	+	✓	✓	✓

参考文档：
https://blog.csdn.net/qq_38733943/article/details/129529813

关闭分区交换

sudo swapon -s # 查看分区情况
关闭命令：systemctl disable dphys-swapfile，记得重启
输入后没有任何反应，应该是没有配置的原因。
通过free -h 查看内存确认一下
设置主机名为master
hostnamectl set-hostname master 
安装运行时
更新软件并安装必要的系统工具
sudo apt-get -y update 
sudo apt-get -y install containerd docker-io # 使apt支持ssl传输 还有一些不太清楚
修改镜像源
sudo vim /etc/docker/daemon.json
添加以下内容，registry-mirrors为镜像源，后续docker拉取镜像的时候，会通过这个镜像源进行拉取，解决国内无法访问外网镜像站的问题。
exec-opts设定执行选项，设定docker为systemd技术（k8s集群需要）。
注意：树莓派安装docker时，默认系统是cgroup的，且不支持systemd，如果不匹配的话可能会出现边缘节点连接上但master看不到，此处如果docker默认是systemd要改成cgroup或直接将有systemd的那行删除.注意阿里云账号下列网址提供了docker镜像加速器地址，可以根据需要修改
https://cr.console.aliyun.com/cn-zhangjiakou/instances/mirrors
{
  "exec-opts": ["native.cgroupdriver=systemd"]    
}

重新载入daemon：
sudo systemctl daemon-reload
重启docker
sudo systemctl restart docker
查看修改后的docker Cgroup的参数，为systemd即修改成功：
docker info | grep Cgroup


安装k8s-1.26
以下操作均在Master节点上进行
组件安装
安装完了 docker 就可以下载 k8s 的三个主要组件kubelet、kubeadm以及kubectl了。
kubelet: k8s 的核心服务
kubeadm: 这个是用于快速安装 k8s 的一个集成工具，我们在master和node上的 k8s 部署都将使用它来完成。
kubectl: k8s 的命令行工具，部署完成之后后续的操作都要用它来执行
其实这三个的下载很简单，直接用apt-get就好了，但是因为某些原因，它们的下载地址不存在了。所以我们需要用国内的镜像站来下载，也很简单。
由于前面装docker时已经装了一些系统必要的应用了，比如允许apt使用ssl传输等。这里可以直接进行安装。
curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add - 
# 下载gpg 密钥
sudo vim /etc/apt/sources.list.d/kubernetes.list
# 添加下面的镜像源
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
sudo apt-get update # 更新源
sudo apt install kubectl=1.26.0-00 kubelet=1.26.0-00 kubeadm=1.26.0-00 # 安装三件套
sudo apt-mark hold kubelet kubeadm kubectl
查看组件版本
kubectl version#查看k8s版本，如下所示
初始化k8s
提前下载镜像
kubeadm config images list// 查看镜像依赖版本

images=(                                                                                                                          
kube-apiserver:v1.28.11                                                                                                                                                           
kube-controller-manager:v1.28.11                                                                                                                                                     
kube-scheduler:v1.28.11                                                                                                                                                       
kube-proxy:v1.28.11                                                                                                                                                               
pause:3.9                                                                                                                                                                               
etcd:3.5.9-0                                                                                                                                                                            
coredns:v1.10.1                                                                                                                                                                  
)                                                                                                                                                                                       
for imageName in ${images[@]};do                                                                                                                                                        
    docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName                                                                                                          
    docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName                                                                                     
    docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName                                                                                                           
done
查看下载镜像
docker images
初始化K8s
kubeadm init  --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 --apiserver-advertise-address=192.168.0.124 --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers    --ignore-preflight-errors=all
上述命令执行成功返回如下打印，表明程序运行正常。注意修改--apiserver-advertise-address值

mkdir -p $HOME/.kube                                                                                                                                                                  
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config                                                                                                                              
sudo chown $(id -u):$(id -g) $HOME/.kube/config

export KUBECONFIG=/etc/kubernetes/admin.conf
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

kubeadm join 192.168.31.169:6443 --token gcvl9j.x4xnqtg26xa7jp7p --discovery-token-ca-cert-hash sha256:8519dd316dbb95b62e68cb9f00d3213aef2633ed934f482c2229688919052daa
此时可以看到部署的节点了
kubectl get nodes # 这里能看到raspberry了 但是是notready 

部署网络
wget https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml
kubectl create -f tigera-operator.yaml
wget https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml
#注意按下图修改custom-resources.yaml
kubectl create -f custom-resources.yaml


修改CIDR地址:图中的cidr值为kubeadm init命令中的 --pod-network-cidr值

此时你的raspberry节点至少应该是Ready状态
kubectl get nodes #显示ready  如果没有ready 过一会再试一下就可以了


问题记录 
● 问题一

前四个error使用kubeadm reset可解决
第五个error
错误分析：
[ERROR CRI]：
CRI（Container Runtime Interface）的远程调用接口，这个接口定义了容器运行时的各项核心操作，比如：启动一个容器需要的所有参数。没有容器运行时就创建不了容器
container runtime is not running: 
推测容器运行时没启动
CRI v1 runtime API is not implemented for endpoint"unix:///var/run/containerd/containerd.sock ：
没安装容器运行时或者默认的socket文件位置没找到
解决：
1. 使用systemctl status containerd 查看状态，若是 Active: active (running) 表示容器运行时正常运行
2. 查看  containerd config default > /etc/containerd/config.toml文件，这个是容器运行时的配置文件，修改配置文件的
     disabled_plugins : ["cri"] 将这行用#注释或者将"cri"删除，然后重启容器运行时
     systemctl restart containerd
第六个errorc	
1.  systemctl disable dphys-swapfile
2. 需要在原始 /boot/cmdline.txt 文件中加上 cgroup_enable=memory 和 cgroup_memory=1 两项参数，最后重新启动服务器即可，该问题一般是由树莓派的兼容性和相关配置错误引起的 Cgroup 问题，内容如下所示：
console=serial0,115200 console=tty1 root=PARTUUID=bed65e87-02 rootfstype=ext4 fsck.repair=yes cgroup_enable=memory cgroup_memory=1 rootwait quiet splash plymouth.ignore-serial-consoles
 cfg80211.ieee80211_regdom=GB

● 问题二

解决思路：
 经过确认，k8s 1.24中启用了CRI sandbox(pause) image的配置支持。前通过kubeadm init –image-repository设置的镜像地址，不再会传递给cri运行时去下载pause镜像。而是需要在cri运行时的配置文件中设置
containerd 中默认已经实现了 CRI，但是是以 plugin 的形式配置的，以前 Docker 中自带的 containerd 默认是将 CRI 这个插件禁用掉了的（使用配置 disabled_plugins = ["cri"]），所以这里我们重新生成默认的配置文件来覆盖掉：
containerd config default > /etc/containerd/config.toml
修改默认的 pause 镜像为国内的地址，替换 [plugins."io.containerd.grpc.v1.cri"] 下面的sandbox_image：
[plugins."io.containerd.grpc.v1.cri"]
sandbox_image="registry.aliyuncs.com/k8sxio/pause:3.9"......
上述操作可以解决如下问题：
RunPodSandbox from runtime service failed

systemctl daemon-reload
systemctl restart containerd
systemctl restart kubelet
● 问题三

解决思路
修改文件/etc/resolv.conf,注释掉下图第三行，增加4和5行
raspi@raspberry:~ $ cat /etc/resolv.conf                                                                                                                                                
# Generated by NetworkManager                                                                                                                                                           
#nameserver 192.168.0.1                                                                                                                                                                 
nameserver 8.8.8.8                                                                                                                                                                      
nameserver 8.8.4.4
● 问题四

解决思路
● 系统时间问题
● etcd、kube-apiserver服务反复重启，查看/var/log/pods目录下的组件日志确认
时间同步问题，通过对时软件或者联网即可
反复重启的原因如下：未正确设置cgroups导致，在containerd的配置文件/etc/containerd/config.toml中，修改SystemdCgroup配置为true。


然后重启 containerd
systemctl restart containerd
部署Kubeedge-1.15
安装keadm
      以v1.15.1为例，先去github下载好keadm的tar.gz文件，通过ftp传输到服务器上的/home/raspi文件夹里。下载方法是去https://github.com/kubeedge/kubeedge/releases找到对应的版本。解压之后发现文件夹里面核心就是一个二进制的keamd文件，将这个文件配置到环境变量。如下：
tar -zxvf keadm-v1.15.1-linux-amd64.tar.gz
cd keadm-v1.15.1-linux-arm64/keadm && chmod +x keadm && mv keadm /usr/local/bin
部署cloudcore
keadm init --kubeedge-version=1.15.1  --advertise-address=10.15.41.190 # 这里填你的公网ip
# kubeedge要求这里需要暴露给边端一个可以访问的公网ip

部署edgecore-加入边缘节点
keadm gettoken #获取token
keadm join --cloudcore-ipport=10.15.41.190:10000 --token=65c4445519de17998e7ff4b851fe897345b67b97bcf36f5317dec3a737b16496.eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE3MDk4MTE1MTZ9.lKsSqk6KGBtDUuzRWzGNrm6OA6mGIHATeiebUSJc5H0 --kubeedge-version=1.15.0 --runtimetype=remote  --with-mqtt=false
问题记录
● 问题一

思路一：
这是因为cloudcore没有污点容忍，默认master节点是不部署应用的，可以用下面的命令查看污点：
#查看污点,如下图所示
kubectl describe nodes raspberry | grep Taints

#删除污点
kubectl taint nodes raspberry node-role.kubernetes.io/control-plane:NoSchedule-
# 然后重置
keadm reset
#再重新启动

K8S Dashboard部署--以v2.7.0为例
获取yaml文件
wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml

增加开放网口
在yaml文件中增加下图中的40行和44行

部署dashboard
kubectl apply -f recommended.yaml 注意k8s和kubeedge的版本对应在https://github.com/kubeedge/kubeedge有详细说明
	K8s 1.20	K8s 1.21	K8s1.22	K8s 1.23	K8s 1.24	K8s 1.25	K8s1.26	K8s 1.27
KubeEdge 1.12	✓	✓	✓	-	-	-	-	-
KubeEdge 1.13	+	✓	✓	✓	-	-	-	-
KubeEdge 1.14	+	+	✓	✓	✓	-	-	-
KubeEdge 1.15	+	+	+	+	✓	✓	✓	-
KubeEdge HEAD (master)	+	+	+	+	+	✓	✓	✓
参考文档：
https://blog.csdn.net/qq_38733943/article/details/129529813
关闭分区交换
sudo swapon -s # 查看分区情况
关闭命令：systemctl disable dphys-swapfile，记得重启
输入后没有任何反应，应该是没有配置的原因。
通过free -h 查看内存确认一下
设置主机名为master
hostnamectl set-hostname master 
安装运行时
更新软件并安装必要的系统工具
sudo apt-get -y update 
sudo apt-get -y install containerd docker-io # 使apt支持ssl传输 还有一些不太清楚
修改镜像源
sudo vim /etc/docker/daemon.json
添加以下内容，registry-mirrors为镜像源，后续docker拉取镜像的时候，会通过这个镜像源进行拉取，解决国内无法访问外网镜像站的问题。
exec-opts设定执行选项，设定docker为systemd技术（k8s集群需要）。
注意：树莓派安装docker时，默认系统是cgroup的，且不支持systemd，如果不匹配的话可能会出现边缘节点连接上但master看不到，此处如果docker默认是systemd要改成cgroup或直接将有systemd的那行删除.注意阿里云账号下列网址提供了docker镜像加速器地址，可以根据需要修改
https://cr.console.aliyun.com/cn-zhangjiakou/instances/mirrors
{
  "exec-opts": ["native.cgroupdriver=systemd"]    
}

重新载入daemon：
sudo systemctl daemon-reload
重启docker
sudo systemctl restart docker
查看修改后的docker Cgroup的参数，为systemd即修改成功：
docker info | grep Cgroup


安装k8s-1.26
以下操作均在Master节点上进行
组件安装
安装完了 docker 就可以下载 k8s 的三个主要组件kubelet、kubeadm以及kubectl了。
kubelet: k8s 的核心服务
kubeadm: 这个是用于快速安装 k8s 的一个集成工具，我们在master和node上的 k8s 部署都将使用它来完成。
kubectl: k8s 的命令行工具，部署完成之后后续的操作都要用它来执行
其实这三个的下载很简单，直接用apt-get就好了，但是因为某些原因，它们的下载地址不存在了。所以我们需要用国内的镜像站来下载，也很简单。
由于前面装docker时已经装了一些系统必要的应用了，比如允许apt使用ssl传输等。这里可以直接进行安装。
curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add - 
# 下载gpg 密钥
sudo vim /etc/apt/sources.list.d/kubernetes.list
# 添加下面的镜像源
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
sudo apt-get update # 更新源
sudo apt install kubectl=1.26.0-00 kubelet=1.26.0-00 kubeadm=1.26.0-00 # 安装三件套
sudo apt-mark hold kubelet kubeadm kubectl
查看组件版本
kubectl version#查看k8s版本，如下所示
初始化k8s
提前下载镜像
kubeadm config images list// 查看镜像依赖版本

images=(                                                                                                                          
kube-apiserver:v1.28.11                                                                                                                                                           
kube-controller-manager:v1.28.11                                                                                                                                                     
kube-scheduler:v1.28.11                                                                                                                                                       
kube-proxy:v1.28.11                                                                                                                                                               
pause:3.9                                                                                                                                                                               
etcd:3.5.9-0                                                                                                                                                                            
coredns:v1.10.1                                                                                                                                                                  
)                                                                                                                                                                                       
for imageName in ${images[@]};do                                                                                                                                                        
    docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName                                                                                                          
    docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName                                                                                     
    docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName                                                                                                           
done
查看下载镜像
docker images
初始化K8s
kubeadm init  --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 --apiserver-advertise-address=192.168.0.124 --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers    --ignore-preflight-errors=all
上述命令执行成功返回如下打印，表明程序运行正常。注意修改--apiserver-advertise-address值

mkdir -p $HOME/.kube                                                                                                                                                                  
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config                                                                                                                              
sudo chown $(id -u):$(id -g) $HOME/.kube/config

export KUBECONFIG=/etc/kubernetes/admin.conf
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

kubeadm join 192.168.31.169:6443 --token gcvl9j.x4xnqtg26xa7jp7p --discovery-token-ca-cert-hash sha256:8519dd316dbb95b62e68cb9f00d3213aef2633ed934f482c2229688919052daa
此时可以看到部署的节点了
kubectl get nodes # 这里能看到raspberry了 但是是notready 

部署网络
wget https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml
kubectl create -f tigera-operator.yaml
wget https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml
#注意按下图修改custom-resources.yaml
kubectl create -f custom-resources.yaml


修改CIDR地址:图中的cidr值为kubeadm init命令中的 --pod-network-cidr值

此时你的raspberry节点至少应该是Ready状态
kubectl get nodes #显示ready  如果没有ready 过一会再试一下就可以了


问题记录 
● 问题一

前四个error使用kubeadm reset可解决
第五个error
错误分析：
[ERROR CRI]：
CRI（Container Runtime Interface）的远程调用接口，这个接口定义了容器运行时的各项核心操作，比如：启动一个容器需要的所有参数。没有容器运行时就创建不了容器
container runtime is not running: 
推测容器运行时没启动
CRI v1 runtime API is not implemented for endpoint"unix:///var/run/containerd/containerd.sock ：
没安装容器运行时或者默认的socket文件位置没找到
解决：
1. 使用systemctl status containerd 查看状态，若是 Active: active (running) 表示容器运行时正常运行
2. 查看  containerd config default > /etc/containerd/config.toml文件，这个是容器运行时的配置文件，修改配置文件的
     disabled_plugins : ["cri"] 将这行用#注释或者将"cri"删除，然后重启容器运行时
     systemctl restart containerd
第六个errorc	
1.  systemctl disable dphys-swapfile
2. 需要在原始 /boot/cmdline.txt 文件中加上 cgroup_enable=memory 和 cgroup_memory=1 两项参数，最后重新启动服务器即可，该问题一般是由树莓派的兼容性和相关配置错误引起的 Cgroup 问题，内容如下所示：
console=serial0,115200 console=tty1 root=PARTUUID=bed65e87-02 rootfstype=ext4 fsck.repair=yes cgroup_enable=memory cgroup_memory=1 rootwait quiet splash plymouth.ignore-serial-consoles
 cfg80211.ieee80211_regdom=GB

● 问题二

解决思路：
 经过确认，k8s 1.24中启用了CRI sandbox(pause) image的配置支持。前通过kubeadm init –image-repository设置的镜像地址，不再会传递给cri运行时去下载pause镜像。而是需要在cri运行时的配置文件中设置
containerd 中默认已经实现了 CRI，但是是以 plugin 的形式配置的，以前 Docker 中自带的 containerd 默认是将 CRI 这个插件禁用掉了的（使用配置 disabled_plugins = ["cri"]），所以这里我们重新生成默认的配置文件来覆盖掉：
containerd config default > /etc/containerd/config.toml
修改默认的 pause 镜像为国内的地址，替换 [plugins."io.containerd.grpc.v1.cri"] 下面的sandbox_image：
[plugins."io.containerd.grpc.v1.cri"]
sandbox_image="registry.aliyuncs.com/k8sxio/pause:3.9"......
上述操作可以解决如下问题：
RunPodSandbox from runtime service failed

systemctl daemon-reload
systemctl restart containerd
systemctl restart kubelet
● 问题三

解决思路
修改文件/etc/resolv.conf,注释掉下图第三行，增加4和5行
raspi@raspberry:~ $ cat /etc/resolv.conf                                                                                                                                                
# Generated by NetworkManager                                                                                                                                                           
#nameserver 192.168.0.1                                                                                                                                                                 
nameserver 8.8.8.8                                                                                                                                                                      
nameserver 8.8.4.4
● 问题四

解决思路
● 系统时间问题
● etcd、kube-apiserver服务反复重启，查看/var/log/pods目录下的组件日志确认
时间同步问题，通过对时软件或者联网即可
反复重启的原因如下：未正确设置cgroups导致，在containerd的配置文件/etc/containerd/config.toml中，修改SystemdCgroup配置为true。


然后重启 containerd
systemctl restart containerd
部署Kubeedge-1.15
安装keadm
      以v1.15.1为例，先去github下载好keadm的tar.gz文件，通过ftp传输到服务器上的/home/raspi文件夹里。下载方法是去https://github.com/kubeedge/kubeedge/releases找到对应的版本。解压之后发现文件夹里面核心就是一个二进制的keamd文件，将这个文件配置到环境变量。如下：
tar -zxvf keadm-v1.15.1-linux-amd64.tar.gz
cd keadm-v1.15.1-linux-arm64/keadm && chmod +x keadm && mv keadm /usr/local/bin
部署cloudcore
keadm init --kubeedge-version=1.15.1  --advertise-address=10.15.41.190 # 这里填你的公网ip
# kubeedge要求这里需要暴露给边端一个可以访问的公网ip

部署edgecore-加入边缘节点
keadm gettoken #获取token
keadm join --cloudcore-ipport=10.15.41.190:10000 --token=65c4445519de17998e7ff4b851fe897345b67b97bcf36f5317dec3a737b16496.eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE3MDk4MTE1MTZ9.lKsSqk6KGBtDUuzRWzGNrm6OA6mGIHATeiebUSJc5H0 --kubeedge-version=1.15.0 --runtimetype=remote  --with-mqtt=false
问题记录
● 问题一

思路一：
这是因为cloudcore没有污点容忍，默认master节点是不部署应用的，可以用下面的命令查看污点：
#查看污点,如下图所示
kubectl describe nodes raspberry | grep Taints

#删除污点
kubectl taint nodes raspberry node-role.kubernetes.io/control-plane:NoSchedule-
# 然后重置
keadm reset
#再重新启动

K8S Dashboard部署--以v2.7.0为例
获取yaml文件
wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml

增加开放网口
在yaml文件中增加下图中的40行和44行

部署dashboard
kubectl apply -f recommended.yaml 

注意k8s和kubeedge的版本对应在https://github.com/kubeedge/kubeedge有详细说明
	K8s 1.20	K8s 1.21	K8s1.22	K8s 1.23	K8s 1.24	K8s 1.25	K8s1.26	K8s 1.27
KubeEdge 1.12	✓	✓	✓	-	-	-	-	-
KubeEdge 1.13	+	✓	✓	✓	-	-	-	-
KubeEdge 1.14	+	+	✓	✓	✓	-	-	-
KubeEdge 1.15	+	+	+	+	✓	✓	✓	-
KubeEdge HEAD (master)	+	+	+	+	+	✓	✓	✓
参考文档：
https://blog.csdn.net/qq_38733943/article/details/129529813
关闭分区交换
sudo swapon -s # 查看分区情况
关闭命令：systemctl disable dphys-swapfile，记得重启
输入后没有任何反应，应该是没有配置的原因。
通过free -h 查看内存确认一下
设置主机名为master
hostnamectl set-hostname master 
安装运行时
更新软件并安装必要的系统工具
sudo apt-get -y update 
sudo apt-get -y install containerd docker-io # 使apt支持ssl传输 还有一些不太清楚
修改镜像源
sudo vim /etc/docker/daemon.json
添加以下内容，registry-mirrors为镜像源，后续docker拉取镜像的时候，会通过这个镜像源进行拉取，解决国内无法访问外网镜像站的问题。
exec-opts设定执行选项，设定docker为systemd技术（k8s集群需要）。
注意：树莓派安装docker时，默认系统是cgroup的，且不支持systemd，如果不匹配的话可能会出现边缘节点连接上但master看不到，此处如果docker默认是systemd要改成cgroup或直接将有systemd的那行删除.注意阿里云账号下列网址提供了docker镜像加速器地址，可以根据需要修改
https://cr.console.aliyun.com/cn-zhangjiakou/instances/mirrors
{
  "exec-opts": ["native.cgroupdriver=systemd"]    
}

重新载入daemon：
sudo systemctl daemon-reload
重启docker
sudo systemctl restart docker
查看修改后的docker Cgroup的参数，为systemd即修改成功：
docker info | grep Cgroup


安装k8s-1.26
以下操作均在Master节点上进行
组件安装
安装完了 docker 就可以下载 k8s 的三个主要组件kubelet、kubeadm以及kubectl了。
kubelet: k8s 的核心服务
kubeadm: 这个是用于快速安装 k8s 的一个集成工具，我们在master和node上的 k8s 部署都将使用它来完成。
kubectl: k8s 的命令行工具，部署完成之后后续的操作都要用它来执行
其实这三个的下载很简单，直接用apt-get就好了，但是因为某些原因，它们的下载地址不存在了。所以我们需要用国内的镜像站来下载，也很简单。
由于前面装docker时已经装了一些系统必要的应用了，比如允许apt使用ssl传输等。这里可以直接进行安装。
curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add - 
# 下载gpg 密钥
sudo vim /etc/apt/sources.list.d/kubernetes.list
# 添加下面的镜像源
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
sudo apt-get update # 更新源
sudo apt install kubectl=1.26.0-00 kubelet=1.26.0-00 kubeadm=1.26.0-00 # 安装三件套
sudo apt-mark hold kubelet kubeadm kubectl
查看组件版本
kubectl version#查看k8s版本，如下所示
初始化k8s
提前下载镜像
kubeadm config images list// 查看镜像依赖版本

images=(                                                                                                                          
kube-apiserver:v1.28.11                                                                                                                                                           
kube-controller-manager:v1.28.11                                                                                                                                                     
kube-scheduler:v1.28.11                                                                                                                                                       
kube-proxy:v1.28.11                                                                                                                                                               
pause:3.9                                                                                                                                                                               
etcd:3.5.9-0                                                                                                                                                                            
coredns:v1.10.1                                                                                                                                                                  
)                                                                                                                                                                                       
for imageName in ${images[@]};do                                                                                                                                                        
    docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName                                                                                                          
    docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName                                                                                     
    docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName                                                                                                           
done
查看下载镜像
docker images
初始化K8s
kubeadm init  --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 --apiserver-advertise-address=192.168.0.124 --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers    --ignore-preflight-errors=all
上述命令执行成功返回如下打印，表明程序运行正常。注意修改--apiserver-advertise-address值

mkdir -p $HOME/.kube                                                                                                                                                                  
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config                                                                                                                              
sudo chown $(id -u):$(id -g) $HOME/.kube/config

export KUBECONFIG=/etc/kubernetes/admin.conf
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

kubeadm join 192.168.31.169:6443 --token gcvl9j.x4xnqtg26xa7jp7p --discovery-token-ca-cert-hash sha256:8519dd316dbb95b62e68cb9f00d3213aef2633ed934f482c2229688919052daa
此时可以看到部署的节点了
kubectl get nodes # 这里能看到raspberry了 但是是notready 

部署网络
wget https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml
kubectl create -f tigera-operator.yaml
wget https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml
#注意按下图修改custom-resources.yaml
kubectl create -f custom-resources.yaml


修改CIDR地址:图中的cidr值为kubeadm init命令中的 --pod-network-cidr值

此时你的raspberry节点至少应该是Ready状态
kubectl get nodes #显示ready  如果没有ready 过一会再试一下就可以了


问题记录 
● 问题一

前四个error使用kubeadm reset可解决
第五个error
错误分析：
[ERROR CRI]：
CRI（Container Runtime Interface）的远程调用接口，这个接口定义了容器运行时的各项核心操作，比如：启动一个容器需要的所有参数。没有容器运行时就创建不了容器
container runtime is not running: 
推测容器运行时没启动
CRI v1 runtime API is not implemented for endpoint"unix:///var/run/containerd/containerd.sock ：
没安装容器运行时或者默认的socket文件位置没找到
解决：
1. 使用systemctl status containerd 查看状态，若是 Active: active (running) 表示容器运行时正常运行
2. 查看  containerd config default > /etc/containerd/config.toml文件，这个是容器运行时的配置文件，修改配置文件的
     disabled_plugins : ["cri"] 将这行用#注释或者将"cri"删除，然后重启容器运行时
     systemctl restart containerd
第六个errorc	
1.  systemctl disable dphys-swapfile
2. 需要在原始 /boot/cmdline.txt 文件中加上 cgroup_enable=memory 和 cgroup_memory=1 两项参数，最后重新启动服务器即可，该问题一般是由树莓派的兼容性和相关配置错误引起的 Cgroup 问题，内容如下所示：
console=serial0,115200 console=tty1 root=PARTUUID=bed65e87-02 rootfstype=ext4 fsck.repair=yes cgroup_enable=memory cgroup_memory=1 rootwait quiet splash plymouth.ignore-serial-consoles
 cfg80211.ieee80211_regdom=GB

● 问题二

解决思路：
 经过确认，k8s 1.24中启用了CRI sandbox(pause) image的配置支持。前通过kubeadm init –image-repository设置的镜像地址，不再会传递给cri运行时去下载pause镜像。而是需要在cri运行时的配置文件中设置
containerd 中默认已经实现了 CRI，但是是以 plugin 的形式配置的，以前 Docker 中自带的 containerd 默认是将 CRI 这个插件禁用掉了的（使用配置 disabled_plugins = ["cri"]），所以这里我们重新生成默认的配置文件来覆盖掉：
containerd config default > /etc/containerd/config.toml
修改默认的 pause 镜像为国内的地址，替换 [plugins."io.containerd.grpc.v1.cri"] 下面的sandbox_image：
[plugins."io.containerd.grpc.v1.cri"]
sandbox_image="registry.aliyuncs.com/k8sxio/pause:3.9"......
上述操作可以解决如下问题：
RunPodSandbox from runtime service failed

systemctl daemon-reload
systemctl restart containerd
systemctl restart kubelet
● 问题三

解决思路
修改文件/etc/resolv.conf,注释掉下图第三行，增加4和5行
raspi@raspberry:~ $ cat /etc/resolv.conf                                                                                                                                                
# Generated by NetworkManager                                                                                                                                                           
#nameserver 192.168.0.1                                                                                                                                                                 
nameserver 8.8.8.8                                                                                                                                                                      
nameserver 8.8.4.4
● 问题四

解决思路
● 系统时间问题
● etcd、kube-apiserver服务反复重启，查看/var/log/pods目录下的组件日志确认
时间同步问题，通过对时软件或者联网即可
反复重启的原因如下：未正确设置cgroups导致，在containerd的配置文件/etc/containerd/config.toml中，修改SystemdCgroup配置为true。


然后重启 containerd
systemctl restart containerd
部署Kubeedge-1.15
安装keadm
      以v1.15.1为例，先去github下载好keadm的tar.gz文件，通过ftp传输到服务器上的/home/raspi文件夹里。下载方法是去https://github.com/kubeedge/kubeedge/releases找到对应的版本。解压之后发现文件夹里面核心就是一个二进制的keamd文件，将这个文件配置到环境变量。如下：
tar -zxvf keadm-v1.15.1-linux-amd64.tar.gz
cd keadm-v1.15.1-linux-arm64/keadm && chmod +x keadm && mv keadm /usr/local/bin
部署cloudcore
keadm init --kubeedge-version=1.15.1  --advertise-address=10.15.41.190 # 这里填你的公网ip
# kubeedge要求这里需要暴露给边端一个可以访问的公网ip

部署edgecore-加入边缘节点
keadm gettoken #获取token
keadm join --cloudcore-ipport=10.15.41.190:10000 --token=65c4445519de17998e7ff4b851fe897345b67b97bcf36f5317dec3a737b16496.eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE3MDk4MTE1MTZ9.lKsSqk6KGBtDUuzRWzGNrm6OA6mGIHATeiebUSJc5H0 --kubeedge-version=1.15.0 --runtimetype=remote  --with-mqtt=false
问题记录
● 问题一

思路一：
这是因为cloudcore没有污点容忍，默认master节点是不部署应用的，可以用下面的命令查看污点：
#查看污点,如下图所示
kubectl describe nodes raspberry | grep Taints

#删除污点
kubectl taint nodes raspberry node-role.kubernetes.io/control-plane:NoSchedule-
# 然后重置
keadm reset
#再重新启动

K8S Dashboard部署--以v2.7.0为例
获取yaml文件
wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml

增加开放网口
在yaml文件中增加下图中的40行和44行

部署dashboard
kubectl apply -f recommended.yaml 


查看Pod信息
kubectl get pod -n kubernetes-dashboard -o wide



创建用户
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin    # 默认内置的 ClusterRole, 超级用户（Super-User）角色（cluster-admin）
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
kubectl apply -f admin-user.yaml 
serviceaccount/admin-user created
clusterrolebinding.rbac.authorization.k8s.io/admin-user created
创建长期token
apiVersion: v1
kind: Secret
metadata:
  name: admin-user-secret
  namespace: kubernetes-dashboard 
  annotations:
    kubernetes.io/service-account.name: admin-user
type: kubernetes.io/service-account-token
kubectl apply -f admin-user-secret.yaml

kubectl get secret -n kubernetes-dashboard 

访问dashboard
获取token
kubectl get secret admin-user-secret -n kubernetes-dashboard -o jsonpath={".data.token"} | base64 -d
浏览器输入： https://IP:30003，填入token，即可访问dashboard注意k8s和kubeedge的版本对应在https://github.com/kubeedge/kubeedge有详细说明
	K8s 1.20	K8s 1.21	K8s1.22	K8s 1.23	K8s 1.24	K8s 1.25	K8s1.26	K8s 1.27
KubeEdge 1.12	✓	✓	✓	-	-	-	-	-
KubeEdge 1.13	+	✓	✓	✓	-	-	-	-
KubeEdge 1.14	+	+	✓	✓	✓	-	-	-
KubeEdge 1.15	+	+	+	+	✓	✓	✓	-
KubeEdge HEAD (master)	+	+	+	+	+	✓	✓	✓
参考文档：
https://blog.csdn.net/qq_38733943/article/details/129529813
关闭分区交换
sudo swapon -s # 查看分区情况
关闭命令：systemctl disable dphys-swapfile，记得重启
输入后没有任何反应，应该是没有配置的原因。
通过free -h 查看内存确认一下
设置主机名为master
hostnamectl set-hostname master 
安装运行时
更新软件并安装必要的系统工具
sudo apt-get -y update 
sudo apt-get -y install containerd docker-io # 使apt支持ssl传输 还有一些不太清楚
修改镜像源
sudo vim /etc/docker/daemon.json
添加以下内容，registry-mirrors为镜像源，后续docker拉取镜像的时候，会通过这个镜像源进行拉取，解决国内无法访问外网镜像站的问题。
exec-opts设定执行选项，设定docker为systemd技术（k8s集群需要）。
注意：树莓派安装docker时，默认系统是cgroup的，且不支持systemd，如果不匹配的话可能会出现边缘节点连接上但master看不到，此处如果docker默认是systemd要改成cgroup或直接将有systemd的那行删除.注意阿里云账号下列网址提供了docker镜像加速器地址，可以根据需要修改
https://cr.console.aliyun.com/cn-zhangjiakou/instances/mirrors
{
  "exec-opts": ["native.cgroupdriver=systemd"]    
}

重新载入daemon：
sudo systemctl daemon-reload
重启docker
sudo systemctl restart docker
查看修改后的docker Cgroup的参数，为systemd即修改成功：
docker info | grep Cgroup


安装k8s-1.26
以下操作均在Master节点上进行
组件安装
安装完了 docker 就可以下载 k8s 的三个主要组件kubelet、kubeadm以及kubectl了。
kubelet: k8s 的核心服务
kubeadm: 这个是用于快速安装 k8s 的一个集成工具，我们在master和node上的 k8s 部署都将使用它来完成。
kubectl: k8s 的命令行工具，部署完成之后后续的操作都要用它来执行
其实这三个的下载很简单，直接用apt-get就好了，但是因为某些原因，它们的下载地址不存在了。所以我们需要用国内的镜像站来下载，也很简单。
由于前面装docker时已经装了一些系统必要的应用了，比如允许apt使用ssl传输等。这里可以直接进行安装。
curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add - 
# 下载gpg 密钥
sudo vim /etc/apt/sources.list.d/kubernetes.list
# 添加下面的镜像源
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
sudo apt-get update # 更新源
sudo apt install kubectl=1.26.0-00 kubelet=1.26.0-00 kubeadm=1.26.0-00 # 安装三件套
sudo apt-mark hold kubelet kubeadm kubectl
查看组件版本
kubectl version#查看k8s版本，如下所示
初始化k8s
提前下载镜像
kubeadm config images list// 查看镜像依赖版本

images=(                                                                                                                          
kube-apiserver:v1.28.11                                                                                                                                                           
kube-controller-manager:v1.28.11                                                                                                                                                     
kube-scheduler:v1.28.11                                                                                                                                                       
kube-proxy:v1.28.11                                                                                                                                                               
pause:3.9                                                                                                                                                                               
etcd:3.5.9-0                                                                                                                                                                            
coredns:v1.10.1                                                                                                                                                                  
)                                                                                                                                                                                       
for imageName in ${images[@]};do                                                                                                                                                        
    docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName                                                                                                          
    docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName                                                                                     
    docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName                                                                                                           
done
查看下载镜像
docker images
初始化K8s
kubeadm init  --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 --apiserver-advertise-address=192.168.0.124 --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers    --ignore-preflight-errors=all
上述命令执行成功返回如下打印，表明程序运行正常。注意修改--apiserver-advertise-address值

mkdir -p $HOME/.kube                                                                                                                                                                  
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config                                                                                                                              
sudo chown $(id -u):$(id -g) $HOME/.kube/config

export KUBECONFIG=/etc/kubernetes/admin.conf
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

kubeadm join 192.168.31.169:6443 --token gcvl9j.x4xnqtg26xa7jp7p --discovery-token-ca-cert-hash sha256:8519dd316dbb95b62e68cb9f00d3213aef2633ed934f482c2229688919052daa
此时可以看到部署的节点了
kubectl get nodes # 这里能看到raspberry了 但是是notready 

部署网络
wget https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml
kubectl create -f tigera-operator.yaml
wget https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml
#注意按下图修改custom-resources.yaml
kubectl create -f custom-resources.yaml


修改CIDR地址:图中的cidr值为kubeadm init命令中的 --pod-network-cidr值

此时你的raspberry节点至少应该是Ready状态
kubectl get nodes #显示ready  如果没有ready 过一会再试一下就可以了


问题记录 
● 问题一

前四个error使用kubeadm reset可解决
第五个error
错误分析：
[ERROR CRI]：
CRI（Container Runtime Interface）的远程调用接口，这个接口定义了容器运行时的各项核心操作，比如：启动一个容器需要的所有参数。没有容器运行时就创建不了容器
container runtime is not running: 
推测容器运行时没启动
CRI v1 runtime API is not implemented for endpoint"unix:///var/run/containerd/containerd.sock ：
没安装容器运行时或者默认的socket文件位置没找到
解决：
1. 使用systemctl status containerd 查看状态，若是 Active: active (running) 表示容器运行时正常运行
2. 查看  containerd config default > /etc/containerd/config.toml文件，这个是容器运行时的配置文件，修改配置文件的
     disabled_plugins : ["cri"] 将这行用#注释或者将"cri"删除，然后重启容器运行时
     systemctl restart containerd
第六个errorc	
1.  systemctl disable dphys-swapfile
2. 需要在原始 /boot/cmdline.txt 文件中加上 cgroup_enable=memory 和 cgroup_memory=1 两项参数，最后重新启动服务器即可，该问题一般是由树莓派的兼容性和相关配置错误引起的 Cgroup 问题，内容如下所示：
console=serial0,115200 console=tty1 root=PARTUUID=bed65e87-02 rootfstype=ext4 fsck.repair=yes cgroup_enable=memory cgroup_memory=1 rootwait quiet splash plymouth.ignore-serial-consoles
 cfg80211.ieee80211_regdom=GB

● 问题二

解决思路：
 经过确认，k8s 1.24中启用了CRI sandbox(pause) image的配置支持。前通过kubeadm init –image-repository设置的镜像地址，不再会传递给cri运行时去下载pause镜像。而是需要在cri运行时的配置文件中设置
containerd 中默认已经实现了 CRI，但是是以 plugin 的形式配置的，以前 Docker 中自带的 containerd 默认是将 CRI 这个插件禁用掉了的（使用配置 disabled_plugins = ["cri"]），所以这里我们重新生成默认的配置文件来覆盖掉：
containerd config default > /etc/containerd/config.toml
修改默认的 pause 镜像为国内的地址，替换 [plugins."io.containerd.grpc.v1.cri"] 下面的sandbox_image：
[plugins."io.containerd.grpc.v1.cri"]
sandbox_image="registry.aliyuncs.com/k8sxio/pause:3.9"......
上述操作可以解决如下问题：
RunPodSandbox from runtime service failed

systemctl daemon-reload
systemctl restart containerd
systemctl restart kubelet
● 问题三

解决思路
修改文件/etc/resolv.conf,注释掉下图第三行，增加4和5行
raspi@raspberry:~ $ cat /etc/resolv.conf                                                                                                                                                
# Generated by NetworkManager                                                                                                                                                           
#nameserver 192.168.0.1                                                                                                                                                                 
nameserver 8.8.8.8                                                                                                                                                                      
nameserver 8.8.4.4
● 问题四

解决思路
● 系统时间问题
● etcd、kube-apiserver服务反复重启，查看/var/log/pods目录下的组件日志确认
时间同步问题，通过对时软件或者联网即可
反复重启的原因如下：未正确设置cgroups导致，在containerd的配置文件/etc/containerd/config.toml中，修改SystemdCgroup配置为true。


然后重启 containerd
systemctl restart containerd
部署Kubeedge-1.15
安装keadm
      以v1.15.1为例，先去github下载好keadm的tar.gz文件，通过ftp传输到服务器上的/home/raspi文件夹里。下载方法是去https://github.com/kubeedge/kubeedge/releases找到对应的版本。解压之后发现文件夹里面核心就是一个二进制的keamd文件，将这个文件配置到环境变量。如下：
tar -zxvf keadm-v1.15.1-linux-amd64.tar.gz
cd keadm-v1.15.1-linux-arm64/keadm && chmod +x keadm && mv keadm /usr/local/bin
部署cloudcore
keadm init --kubeedge-version=1.15.1  --advertise-address=10.15.41.190 # 这里填你的公网ip
# kubeedge要求这里需要暴露给边端一个可以访问的公网ip

部署edgecore-加入边缘节点
keadm gettoken #获取token
keadm join --cloudcore-ipport=10.15.41.190:10000 --token=65c4445519de17998e7ff4b851fe897345b67b97bcf36f5317dec3a737b16496.eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE3MDk4MTE1MTZ9.lKsSqk6KGBtDUuzRWzGNrm6OA6mGIHATeiebUSJc5H0 --kubeedge-version=1.15.0 --runtimetype=remote  --with-mqtt=false
问题记录
● 问题一

思路一：
这是因为cloudcore没有污点容忍，默认master节点是不部署应用的，可以用下面的命令查看污点：
#查看污点,如下图所示
kubectl describe nodes raspberry | grep Taints

#删除污点
kubectl taint nodes raspberry node-role.kubernetes.io/control-plane:NoSchedule-
# 然后重置
keadm reset
#再重新启动

K8S Dashboard部署--以v2.7.0为例
获取yaml文件
wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml

增加开放网口
在yaml文件中增加下图中的40行和44行

部署dashboard
kubectl apply -f recommended.yaml 


查看Pod信息
kubectl get pod -n kubernetes-dashboard -o wide



创建用户
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin    # 默认内置的 ClusterRole, 超级用户（Super-User）角色（cluster-admin）
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
kubectl apply -f admin-user.yaml 
serviceaccount/admin-user created
clusterrolebinding.rbac.authorization.k8s.io/admin-user created
创建长期token
apiVersion: v1
kind: Secret
metadata:
  name: admin-user-secret
  namespace: kubernetes-dashboard 
  annotations:
    kubernetes.io/service-account.name: admin-user
type: kubernetes.io/service-account-token
kubectl apply -f admin-user-secret.yaml

kubectl get secret -n kubernetes-dashboard 

访问dashboard
获取token
kubectl get secret admin-user-secret -n kubernetes-dashboard -o jsonpath={".data.token"} | base64 -d
浏览器输入： https://IP:30003，填入token，即可访问dashboard注意k8s和kubeedge的版本对应在https://github.com/kubeedge/kubeedge有详细说明
	K8s 1.20	K8s 1.21	K8s1.22	K8s 1.23	K8s 1.24	K8s 1.25	K8s1.26	K8s 1.27
KubeEdge 1.12	✓	✓	✓	-	-	-	-	-
KubeEdge 1.13	+	✓	✓	✓	-	-	-	-
KubeEdge 1.14	+	+	✓	✓	✓	-	-	-
KubeEdge 1.15	+	+	+	+	✓	✓	✓	-
KubeEdge HEAD (master)	+	+	+	+	+	✓	✓	✓
参考文档：
https://blog.csdn.net/qq_38733943/article/details/129529813
关闭分区交换
sudo swapon -s # 查看分区情况
关闭命令：systemctl disable dphys-swapfile，记得重启
输入后没有任何反应，应该是没有配置的原因。
通过free -h 查看内存确认一下
设置主机名为master
hostnamectl set-hostname master 
安装运行时
更新软件并安装必要的系统工具
sudo apt-get -y update 
sudo apt-get -y install containerd docker-io # 使apt支持ssl传输 还有一些不太清楚
修改镜像源
sudo vim /etc/docker/daemon.json
添加以下内容，registry-mirrors为镜像源，后续docker拉取镜像的时候，会通过这个镜像源进行拉取，解决国内无法访问外网镜像站的问题。
exec-opts设定执行选项，设定docker为systemd技术（k8s集群需要）。
注意：树莓派安装docker时，默认系统是cgroup的，且不支持systemd，如果不匹配的话可能会出现边缘节点连接上但master看不到，此处如果docker默认是systemd要改成cgroup或直接将有systemd的那行删除.注意阿里云账号下列网址提供了docker镜像加速器地址，可以根据需要修改
https://cr.console.aliyun.com/cn-zhangjiakou/instances/mirrors
{
  "exec-opts": ["native.cgroupdriver=systemd"]    
}

重新载入daemon：
sudo systemctl daemon-reload
重启docker
sudo systemctl restart docker
查看修改后的docker Cgroup的参数，为systemd即修改成功：
docker info | grep Cgroup


安装k8s-1.26
以下操作均在Master节点上进行
组件安装
安装完了 docker 就可以下载 k8s 的三个主要组件kubelet、kubeadm以及kubectl了。
kubelet: k8s 的核心服务
kubeadm: 这个是用于快速安装 k8s 的一个集成工具，我们在master和node上的 k8s 部署都将使用它来完成。
kubectl: k8s 的命令行工具，部署完成之后后续的操作都要用它来执行
其实这三个的下载很简单，直接用apt-get就好了，但是因为某些原因，它们的下载地址不存在了。所以我们需要用国内的镜像站来下载，也很简单。
由于前面装docker时已经装了一些系统必要的应用了，比如允许apt使用ssl传输等。这里可以直接进行安装。
curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add - 
# 下载gpg 密钥
sudo vim /etc/apt/sources.list.d/kubernetes.list
# 添加下面的镜像源
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
sudo apt-get update # 更新源
sudo apt install kubectl=1.26.0-00 kubelet=1.26.0-00 kubeadm=1.26.0-00 # 安装三件套
sudo apt-mark hold kubelet kubeadm kubectl
查看组件版本
kubectl version#查看k8s版本，如下所示
初始化k8s
提前下载镜像
kubeadm config images list// 查看镜像依赖版本

images=(                                                                                                                          
kube-apiserver:v1.28.11                                                                                                                                                           
kube-controller-manager:v1.28.11                                                                                                                                                     
kube-scheduler:v1.28.11                                                                                                                                                       
kube-proxy:v1.28.11                                                                                                                                                               
pause:3.9                                                                                                                                                                               
etcd:3.5.9-0                                                                                                                                                                            
coredns:v1.10.1                                                                                                                                                                  
)                                                                                                                                                                                       
for imageName in ${images[@]};do                                                                                                                                                        
    docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName                                                                                                          
    docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName                                                                                     
    docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName                                                                                                           
done
查看下载镜像
docker images
初始化K8s
kubeadm init  --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 --apiserver-advertise-address=192.168.0.124 --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers    --ignore-preflight-errors=all
上述命令执行成功返回如下打印，表明程序运行正常。注意修改--apiserver-advertise-address值

mkdir -p $HOME/.kube                                                                                                                                                                  
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config                                                                                                                              
sudo chown $(id -u):$(id -g) $HOME/.kube/config

export KUBECONFIG=/etc/kubernetes/admin.conf
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

kubeadm join 192.168.31.169:6443 --token gcvl9j.x4xnqtg26xa7jp7p --discovery-token-ca-cert-hash sha256:8519dd316dbb95b62e68cb9f00d3213aef2633ed934f482c2229688919052daa
此时可以看到部署的节点了
kubectl get nodes # 这里能看到raspberry了 但是是notready 

部署网络
wget https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml
kubectl create -f tigera-operator.yaml
wget https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml
#注意按下图修改custom-resources.yaml
kubectl create -f custom-resources.yaml


修改CIDR地址:图中的cidr值为kubeadm init命令中的 --pod-network-cidr值

此时你的raspberry节点至少应该是Ready状态
kubectl get nodes #显示ready  如果没有ready 过一会再试一下就可以了


问题记录 
● 问题一

前四个error使用kubeadm reset可解决
第五个error
错误分析：
[ERROR CRI]：
CRI（Container Runtime Interface）的远程调用接口，这个接口定义了容器运行时的各项核心操作，比如：启动一个容器需要的所有参数。没有容器运行时就创建不了容器
container runtime is not running: 
推测容器运行时没启动
CRI v1 runtime API is not implemented for endpoint"unix:///var/run/containerd/containerd.sock ：
没安装容器运行时或者默认的socket文件位置没找到
解决：
1. 使用systemctl status containerd 查看状态，若是 Active: active (running) 表示容器运行时正常运行
2. 查看  containerd config default > /etc/containerd/config.toml文件，这个是容器运行时的配置文件，修改配置文件的
     disabled_plugins : ["cri"] 将这行用#注释或者将"cri"删除，然后重启容器运行时
     systemctl restart containerd
第六个errorc	
1.  systemctl disable dphys-swapfile
2. 需要在原始 /boot/cmdline.txt 文件中加上 cgroup_enable=memory 和 cgroup_memory=1 两项参数，最后重新启动服务器即可，该问题一般是由树莓派的兼容性和相关配置错误引起的 Cgroup 问题，内容如下所示：
console=serial0,115200 console=tty1 root=PARTUUID=bed65e87-02 rootfstype=ext4 fsck.repair=yes cgroup_enable=memory cgroup_memory=1 rootwait quiet splash plymouth.ignore-serial-consoles
 cfg80211.ieee80211_regdom=GB

● 问题二

解决思路：
 经过确认，k8s 1.24中启用了CRI sandbox(pause) image的配置支持。前通过kubeadm init –image-repository设置的镜像地址，不再会传递给cri运行时去下载pause镜像。而是需要在cri运行时的配置文件中设置
containerd 中默认已经实现了 CRI，但是是以 plugin 的形式配置的，以前 Docker 中自带的 containerd 默认是将 CRI 这个插件禁用掉了的（使用配置 disabled_plugins = ["cri"]），所以这里我们重新生成默认的配置文件来覆盖掉：
containerd config default > /etc/containerd/config.toml
修改默认的 pause 镜像为国内的地址，替换 [plugins."io.containerd.grpc.v1.cri"] 下面的sandbox_image：
[plugins."io.containerd.grpc.v1.cri"]
sandbox_image="registry.aliyuncs.com/k8sxio/pause:3.9"......
上述操作可以解决如下问题：
RunPodSandbox from runtime service failed

systemctl daemon-reload
systemctl restart containerd
systemctl restart kubelet
● 问题三

解决思路
修改文件/etc/resolv.conf,注释掉下图第三行，增加4和5行
raspi@raspberry:~ $ cat /etc/resolv.conf                                                                                                                                                
# Generated by NetworkManager                                                                                                                                                           
#nameserver 192.168.0.1                                                                                                                                                                 
nameserver 8.8.8.8                                                                                                                                                                      
nameserver 8.8.4.4
● 问题四

解决思路
● 系统时间问题
● etcd、kube-apiserver服务反复重启，查看/var/log/pods目录下的组件日志确认
时间同步问题，通过对时软件或者联网即可
反复重启的原因如下：未正确设置cgroups导致，在containerd的配置文件/etc/containerd/config.toml中，修改SystemdCgroup配置为true。


然后重启 containerd
systemctl restart containerd
部署Kubeedge-1.15
安装keadm
      以v1.15.1为例，先去github下载好keadm的tar.gz文件，通过ftp传输到服务器上的/home/raspi文件夹里。下载方法是去https://github.com/kubeedge/kubeedge/releases找到对应的版本。解压之后发现文件夹里面核心就是一个二进制的keamd文件，将这个文件配置到环境变量。如下：
tar -zxvf keadm-v1.15.1-linux-amd64.tar.gz
cd keadm-v1.15.1-linux-arm64/keadm && chmod +x keadm && mv keadm /usr/local/bin
部署cloudcore
keadm init --kubeedge-version=1.15.1  --advertise-address=10.15.41.190 # 这里填你的公网ip
# kubeedge要求这里需要暴露给边端一个可以访问的公网ip

部署edgecore-加入边缘节点
keadm gettoken #获取token
keadm join --cloudcore-ipport=10.15.41.190:10000 --token=65c4445519de17998e7ff4b851fe897345b67b97bcf36f5317dec3a737b16496.eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE3MDk4MTE1MTZ9.lKsSqk6KGBtDUuzRWzGNrm6OA6mGIHATeiebUSJc5H0 --kubeedge-version=1.15.0 --runtimetype=remote  --with-mqtt=false
问题记录
● 问题一

思路一：
这是因为cloudcore没有污点容忍，默认master节点是不部署应用的，可以用下面的命令查看污点：
#查看污点,如下图所示
kubectl describe nodes raspberry | grep Taints

#删除污点
kubectl taint nodes raspberry node-role.kubernetes.io/control-plane:NoSchedule-
# 然后重置
keadm reset
#再重新启动

K8S Dashboard部署--以v2.7.0为例
获取yaml文件
wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml

增加开放网口
在yaml文件中增加下图中的40行和44行

部署dashboard
kubectl apply -f recommended.yaml 


查看Pod信息
kubectl get pod -n kubernetes-dashboard -o wide



创建用户
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin    # 默认内置的 ClusterRole, 超级用户（Super-User）角色（cluster-admin）
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
kubectl apply -f admin-user.yaml 
serviceaccount/admin-user created
clusterrolebinding.rbac.authorization.k8s.io/admin-user created
创建长期token
apiVersion: v1
kind: Secret
metadata:
  name: admin-user-secret
  namespace: kubernetes-dashboard 
  annotations:
    kubernetes.io/service-account.name: admin-user
type: kubernetes.io/service-account-token
kubectl apply -f admin-user-secret.yaml

kubectl get secret -n kubernetes-dashboard 

访问dashboard
获取token
kubectl get secret admin-user-secret -n kubernetes-dashboard -o jsonpath={".data.token"} | base64 -d
浏览器输入： https://IP:30003，填入token，即可访问dashboard注意k8s和kubeedge的版本对应在https://github.com/kubeedge/kubeedge有详细说明
	K8s 1.20	K8s 1.21	K8s1.22	K8s 1.23	K8s 1.24	K8s 1.25	K8s1.26	K8s 1.27
KubeEdge 1.12	✓	✓	✓	-	-	-	-	-
KubeEdge 1.13	+	✓	✓	✓	-	-	-	-
KubeEdge 1.14	+	+	✓	✓	✓	-	-	-
KubeEdge 1.15	+	+	+	+	✓	✓	✓	-
KubeEdge HEAD (master)	+	+	+	+	+	✓	✓	✓
参考文档：
https://blog.csdn.net/qq_38733943/article/details/129529813
关闭分区交换
sudo swapon -s # 查看分区情况
关闭命令：systemctl disable dphys-swapfile，记得重启
输入后没有任何反应，应该是没有配置的原因。
通过free -h 查看内存确认一下
设置主机名为master
hostnamectl set-hostname master 
安装运行时
更新软件并安装必要的系统工具
sudo apt-get -y update 
sudo apt-get -y install containerd docker-io # 使apt支持ssl传输 还有一些不太清楚
修改镜像源
sudo vim /etc/docker/daemon.json
添加以下内容，registry-mirrors为镜像源，后续docker拉取镜像的时候，会通过这个镜像源进行拉取，解决国内无法访问外网镜像站的问题。
exec-opts设定执行选项，设定docker为systemd技术（k8s集群需要）。
注意：树莓派安装docker时，默认系统是cgroup的，且不支持systemd，如果不匹配的话可能会出现边缘节点连接上但master看不到，此处如果docker默认是systemd要改成cgroup或直接将有systemd的那行删除.注意阿里云账号下列网址提供了docker镜像加速器地址，可以根据需要修改
https://cr.console.aliyun.com/cn-zhangjiakou/instances/mirrors
{
  "exec-opts": ["native.cgroupdriver=systemd"]    
}

重新载入daemon：
sudo systemctl daemon-reload
重启docker
sudo systemctl restart docker
查看修改后的docker Cgroup的参数，为systemd即修改成功：
docker info | grep Cgroup


安装k8s-1.26
以下操作均在Master节点上进行
组件安装
安装完了 docker 就可以下载 k8s 的三个主要组件kubelet、kubeadm以及kubectl了。
kubelet: k8s 的核心服务
kubeadm: 这个是用于快速安装 k8s 的一个集成工具，我们在master和node上的 k8s 部署都将使用它来完成。
kubectl: k8s 的命令行工具，部署完成之后后续的操作都要用它来执行
其实这三个的下载很简单，直接用apt-get就好了，但是因为某些原因，它们的下载地址不存在了。所以我们需要用国内的镜像站来下载，也很简单。
由于前面装docker时已经装了一些系统必要的应用了，比如允许apt使用ssl传输等。这里可以直接进行安装。
curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add - 
# 下载gpg 密钥
sudo vim /etc/apt/sources.list.d/kubernetes.list
# 添加下面的镜像源
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
sudo apt-get update # 更新源
sudo apt install kubectl=1.26.0-00 kubelet=1.26.0-00 kubeadm=1.26.0-00 # 安装三件套
sudo apt-mark hold kubelet kubeadm kubectl
查看组件版本
kubectl version#查看k8s版本，如下所示
初始化k8s
提前下载镜像
kubeadm config images list// 查看镜像依赖版本

images=(                                                                                                                          
kube-apiserver:v1.28.11                                                                                                                                                           
kube-controller-manager:v1.28.11                                                                                                                                                     
kube-scheduler:v1.28.11                                                                                                                                                       
kube-proxy:v1.28.11                                                                                                                                                               
pause:3.9                                                                                                                                                                               
etcd:3.5.9-0                                                                                                                                                                            
coredns:v1.10.1                                                                                                                                                                  
)                                                                                                                                                                                       
for imageName in ${images[@]};do                                                                                                                                                        
    docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName                                                                                                          
    docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName                                                                                     
    docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName                                                                                                           
done
查看下载镜像
docker images
初始化K8s
kubeadm init  --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 --apiserver-advertise-address=192.168.0.124 --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers    --ignore-preflight-errors=all
上述命令执行成功返回如下打印，表明程序运行正常。注意修改--apiserver-advertise-address值

mkdir -p $HOME/.kube                                                                                                                                                                  
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config                                                                                                                              
sudo chown $(id -u):$(id -g) $HOME/.kube/config

export KUBECONFIG=/etc/kubernetes/admin.conf
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

kubeadm join 192.168.31.169:6443 --token gcvl9j.x4xnqtg26xa7jp7p --discovery-token-ca-cert-hash sha256:8519dd316dbb95b62e68cb9f00d3213aef2633ed934f482c2229688919052daa
此时可以看到部署的节点了
kubectl get nodes # 这里能看到raspberry了 但是是notready 

部署网络
wget https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml
kubectl create -f tigera-operator.yaml
wget https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml
#注意按下图修改custom-resources.yaml
kubectl create -f custom-resources.yaml


修改CIDR地址:图中的cidr值为kubeadm init命令中的 --pod-network-cidr值

此时你的raspberry节点至少应该是Ready状态
kubectl get nodes #显示ready  如果没有ready 过一会再试一下就可以了


问题记录 
● 问题一

前四个error使用kubeadm reset可解决
第五个error
错误分析：
[ERROR CRI]：
CRI（Container Runtime Interface）的远程调用接口，这个接口定义了容器运行时的各项核心操作，比如：启动一个容器需要的所有参数。没有容器运行时就创建不了容器
container runtime is not running: 
推测容器运行时没启动
CRI v1 runtime API is not implemented for endpoint"unix:///var/run/containerd/containerd.sock ：
没安装容器运行时或者默认的socket文件位置没找到
解决：
1. 使用systemctl status containerd 查看状态，若是 Active: active (running) 表示容器运行时正常运行
2. 查看  containerd config default > /etc/containerd/config.toml文件，这个是容器运行时的配置文件，修改配置文件的
     disabled_plugins : ["cri"] 将这行用#注释或者将"cri"删除，然后重启容器运行时
     systemctl restart containerd
第六个errorc	
1.  systemctl disable dphys-swapfile
2. 需要在原始 /boot/cmdline.txt 文件中加上 cgroup_enable=memory 和 cgroup_memory=1 两项参数，最后重新启动服务器即可，该问题一般是由树莓派的兼容性和相关配置错误引起的 Cgroup 问题，内容如下所示：
console=serial0,115200 console=tty1 root=PARTUUID=bed65e87-02 rootfstype=ext4 fsck.repair=yes cgroup_enable=memory cgroup_memory=1 rootwait quiet splash plymouth.ignore-serial-consoles
 cfg80211.ieee80211_regdom=GB

● 问题二

解决思路：
 经过确认，k8s 1.24中启用了CRI sandbox(pause) image的配置支持。前通过kubeadm init –image-repository设置的镜像地址，不再会传递给cri运行时去下载pause镜像。而是需要在cri运行时的配置文件中设置
containerd 中默认已经实现了 CRI，但是是以 plugin 的形式配置的，以前 Docker 中自带的 containerd 默认是将 CRI 这个插件禁用掉了的（使用配置 disabled_plugins = ["cri"]），所以这里我们重新生成默认的配置文件来覆盖掉：
containerd config default > /etc/containerd/config.toml
修改默认的 pause 镜像为国内的地址，替换 [plugins."io.containerd.grpc.v1.cri"] 下面的sandbox_image：
[plugins."io.containerd.grpc.v1.cri"]
sandbox_image="registry.aliyuncs.com/k8sxio/pause:3.9"......
上述操作可以解决如下问题：
RunPodSandbox from runtime service failed

systemctl daemon-reload
systemctl restart containerd
systemctl restart kubelet
● 问题三

解决思路
修改文件/etc/resolv.conf,注释掉下图第三行，增加4和5行
raspi@raspberry:~ $ cat /etc/resolv.conf                                                                                                                                                
# Generated by NetworkManager                                                                                                                                                           
#nameserver 192.168.0.1                                                                                                                                                                 
nameserver 8.8.8.8                                                                                                                                                                      
nameserver 8.8.4.4
● 问题四

解决思路
● 系统时间问题
● etcd、kube-apiserver服务反复重启，查看/var/log/pods目录下的组件日志确认
时间同步问题，通过对时软件或者联网即可
反复重启的原因如下：未正确设置cgroups导致，在containerd的配置文件/etc/containerd/config.toml中，修改SystemdCgroup配置为true。


然后重启 containerd
systemctl restart containerd
部署Kubeedge-1.15
安装keadm
      以v1.15.1为例，先去github下载好keadm的tar.gz文件，通过ftp传输到服务器上的/home/raspi文件夹里。下载方法是去https://github.com/kubeedge/kubeedge/releases找到对应的版本。解压之后发现文件夹里面核心就是一个二进制的keamd文件，将这个文件配置到环境变量。如下：
tar -zxvf keadm-v1.15.1-linux-amd64.tar.gz
cd keadm-v1.15.1-linux-arm64/keadm && chmod +x keadm && mv keadm /usr/local/bin
部署cloudcore
keadm init --kubeedge-version=1.15.1  --advertise-address=10.15.41.190 # 这里填你的公网ip
# kubeedge要求这里需要暴露给边端一个可以访问的公网ip

部署edgecore-加入边缘节点
keadm gettoken #获取token
keadm join --cloudcore-ipport=10.15.41.190:10000 --token=65c4445519de17998e7ff4b851fe897345b67b97bcf36f5317dec3a737b16496.eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE3MDk4MTE1MTZ9.lKsSqk6KGBtDUuzRWzGNrm6OA6mGIHATeiebUSJc5H0 --kubeedge-version=1.15.0 --runtimetype=remote  --with-mqtt=false
问题记录
● 问题一

思路一：
这是因为cloudcore没有污点容忍，默认master节点是不部署应用的，可以用下面的命令查看污点：
#查看污点,如下图所示
kubectl describe nodes raspberry | grep Taints

#删除污点
kubectl taint nodes raspberry node-role.kubernetes.io/control-plane:NoSchedule-
# 然后重置
keadm reset
#再重新启动

K8S Dashboard部署--以v2.7.0为例
获取yaml文件
wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml

增加开放网口
在yaml文件中增加下图中的40行和44行

部署dashboard
kubectl apply -f recommended.yaml 


查看Pod信息
kubectl get pod -n kubernetes-dashboard -o wide



创建用户
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin    # 默认内置的 ClusterRole, 超级用户（Super-User）角色（cluster-admin）
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
kubectl apply -f admin-user.yaml 
serviceaccount/admin-user created
clusterrolebinding.rbac.authorization.k8s.io/admin-user created
创建长期token
apiVersion: v1
kind: Secret
metadata:
  name: admin-user-secret
  namespace: kubernetes-dashboard 
  annotations:
    kubernetes.io/service-account.name: admin-user
type: kubernetes.io/service-account-token
kubectl apply -f admin-user-secret.yaml

kubectl get secret -n kubernetes-dashboard 

访问dashboard
获取token
kubectl get secret admin-user-secret -n kubernetes-dashboard -o jsonpath={".data.token"} | base64 -d
浏览器输入： https://IP:30003，填入token，即可访问dashboard
查看Pod信息
kubectl get pod -n kubernetes-dashboard -o wide



创建用户
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin    # 默认内置的 ClusterRole, 超级用户（Super-User）角色（cluster-admin）
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
kubectl apply -f admin-user.yaml 
serviceaccount/admin-user created
clusterrolebinding.rbac.authorization.k8s.io/admin-user created
创建长期token
apiVersion: v1
kind: Secret
metadata:
  name: admin-user-secret
  namespace: kubernetes-dashboard 
  annotations:
    kubernetes.io/service-account.name: admin-user
type: kubernetes.io/service-account-token
kubectl apply -f admin-user-secret.yaml

kubectl get secret -n kubernetes-dashboard 

访问dashboard
获取token
kubectl get secret admin-user-secret -n kubernetes-dashboard -o jsonpath={".data.token"} | base64 -d
浏览器输入： https://IP:30003，填入token，即可访问dashboard


查看Pod信息
kubectl get pod -n kubernetes-dashboard -o wide



创建用户
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin    # 默认内置的 ClusterRole, 超级用户（Super-User）角色（cluster-admin）
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
kubectl apply -f admin-user.yaml 
serviceaccount/admin-user created
clusterrolebinding.rbac.authorization.k8s.io/admin-user created
创建长期token
apiVersion: v1
kind: Secret
metadata:
  name: admin-user-secret
  namespace: kubernetes-dashboard 
  annotations:
    kubernetes.io/service-account.name: admin-user
type: kubernetes.io/service-account-token
kubectl apply -f admin-user-secret.yaml

kubectl get secret -n kubernetes-dashboard 

访问dashboard
获取token
kubectl get secret admin-user-secret -n kubernetes-dashboard -o jsonpath={".data.token"} | base64 -d
浏览器输入： https://IP:30003，填入token，即可访问dashboard