### k8s集群全貌
![https://ping666.com/wp-content/uploads/2024/12/kubelet-architect.png](https://ping666.com/wp-content/uploads/2024/12/kubelet-architect.png "https://ping666.com/wp-content/uploads/2024/12/kubelet-architect.png")

k8s在1.24版本移除了dockershim后，缩短了调用路径kubelet -> containerd(cri plugin) -> runc -> container。
老版本的调用路径为kubelet（dockershim） -> dockerd -> containerd -> runc -> container。

### 机器环境准备
使用VirtualBox（使用的版本为7.1.4）创建虚拟机，Linux镜像为`AnolisOS-8.9-x86_64-minimal.iso`。采用NAT Network组网。

- 选择NAT Network网络模式
  VirtualBox支持的所有组网模式如下：
  - Bridged Adapter
    虚拟机会被分配到一个独立网络中，和真实主机的网络功能完全一致。
    功能很强大，但是配置过程过于繁琐，特定场景才用。
  - NAT
    虚拟机可以使用主机网络主动访问外网，反向则不行。
    虚拟机一般需要访问外网部署、升级程序，这个不满足。
  - NAT Network
    虚拟机可以使用主机网络主动访问外网，反向则不行。同时，虚拟机之间可以互相访问。
    这种模式最常用。
  - Internal Network
    虚拟机之间互通，但是和外界隔离。
  - Host Only Adapter
    只能和主机互通。
  - Not Attached
    没有网络。
  - Generic Driver
    由外接的驱动控制网络。

  在VirtualBox|File|Tools|Network Manager|NAT Networks下创建一个名为k8s的网络。然后创建虚拟机时选择NAT-Network网络模式。默认网络为10.0.2.0/24。
  建议同时配置Port Forwarding：
  `ssh-master TCP 192.168.101.128 10022 10.0.2.10 22`
  这样在主机ssh root@192.168.101.128:10022就可以登录进虚拟机10.0.2.10了。

- 组网规划
  | 主机名 | 作用 | 资源 | IP | Host转发端口 | 其他 |
  | ----------- | ----------- | ----------- | ----------- | ----------- | ----------- |
  | k8s-master | 控制平面 | 2C4G | 10.0.2.10 | 10022 |
  | k8s-node1 | Node1 | 1C4G | 10.0.2.11 | 10122 |
  | k8s-node2 | Node2 | 1C4G | 10.0.2.12 | 10222 |
  | k8s-node3 | Node3 | 1C4G | 10.0.2.13 | 10322 |
  | k8s-registry | 本地仓库 | 1C2G | 10.0.2.99 | 19922 | 仓库端口5000 |
  
  Node节点的内存偏大是为了后面部署Java应用。

- 机器初始化
  每台虚拟机都需要做如下初始化。
  1. 设置hosts和主机名
     /etc/hosts文件
     ```ini
     10.0.2.10 k8s-master k8s-master.x.com
     10.0.2.11 k8s-node1 node1.x.com
     10.0.2.12 k8s-node2 node2.x.com
     10.0.2.13 k8s-node3 node3.x.com
     10.0.2.99 k8s-registry registry.x.com
     ```
     给机器取个有辨识度的名字，比如`hostnamectl set-hostname k8s-master`
  2. 设置IP
    设置固定IP方便测试，熟练了以后可以改成DHCP模式。
     接口文件 `/etc/sysconfig/network-scripts/ifcfg-enp0s3`
     ```ini
     BOOTPROTO=static
     ONBOOT=yes #以上两项修改，以下为新增，其他项不变
     IPADDR=10.0.2.14 # 主机IP，每台都要改
     NETMASK=255.255.255.0
     GATEWAY=10.0.2.1
     DNS1=114.114.114.114
     DNS2=8.8.8.8
     ```
  3. 检查机器唯一标识
     ```bash
     cat /sys/class/dmi/id/product_uuid #机器号
     ifconfig -a #MAC地址
     ```

**建议：**

- 一台机器做完所有的初始化后，然后VirtualBox直接clone会比较方便。clone时要选择重新生成MAC Address。


### 安装VirualBox增强
- 主机安装增强
  [http://download.virtualbox.org/virtualbox/7.1.4/Oracle_VirtualBox_Extension_Pack-7.1.4.vbox-extpack](http://download.virtualbox.org/virtualbox/7.1.4/Oracle_VirtualBox_Extension_Pack-7.1.4.vbox-extpack "http://download.virtualbox.org/virtualbox/7.1.4/Oracle_VirtualBox_Extension_Pack-7.1.4.vbox-extpack")
  安装增强后
  - 在Settings|General|Advanced下把拖放和共享剪切板设置为双向
  - 在Settings|General|Storage|Controller:SATA下打开`Solid-state Drive`
- 虚拟机安装增强
  [http://download.virtualbox.org/virtualbox/7.1.4/VBoxGuestAdditions_7.1.4.iso](http://download.virtualbox.org/virtualbox/7.1.4/VBoxGuestAdditions_7.1.4.iso "http://download.virtualbox.org/virtualbox/7.1.4/VBoxGuestAdditions_7.1.4.iso")
  在Settings|General|Storage|Controller:IDE下设置VBoxGuestAdditions_7.1.4.iso并且打开`Use Host I/O Cache`

    在虚拟机内部安装增强
    ```bash
    yum install gcc make perl kernel-devel
    mount /dev/cdrom /mnt/cdrom
    cd /mnt/cdrom
    ./VBoxLinuxAdditions.run
    unmount /mnt/cdrom
    ```
- 共享文件夹
  在Settings|Shared Folders下配置宿主机器共享目录到shared
  ```bash
  mount -t vboxsf shared /mnt/shared
  ```
  每次开机需要重新mount，最好加入到~/.bash_profile中

### 安装docker环境
当前版本`Docker version 26.1.3`
[https://docs.docker.com/engine/install/centos](https://docs.docker.com/engine/install/centos "https://docs.docker.com/engine/install/centos")

1. 安装docker
   ```bash
   yum -y install yum-utils
   yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
   yum install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
   ```
2. 设置docker镜像源
   `/etc/docker/daemon.json`
   ```json
   {
       "registry-mirrors": [
           "https://s0a2lqra.mirror.aliyuncs.com",
           "https://registry.docker-cn.com",
           "https://ccr.ccs.tencentyun.com"
       ],
       "insecure-registries": [
           "registry.x.com:5000" // 私有仓库地址，后面会安装
       ],
       "exec-opts": [
          "native.cgroupdriver=systemd" // 有systemd操作系统上必须选systemd作为cgroup驱动
       ]
   }
   ```
3. 启用CRI插件
   containerd.io默认禁用了CRI插件。
   这个命令`containerd config default > /etc/containerd/config.toml`可以输出containerd的所有默认配置。
   在config.toml文件中把`disabled_plugins = ["cri"]`注释掉，然后加上下面这些。
   ```ini
   version = 2
   enabled_plugins = ["cri"]
   [plugins]
     [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
       runtime_type = "io.containerd.runc.v2"
       [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
         SystemdCgroup = true
     [plugins."io.containerd.grpc.v1.cri"]
       sandbox_image = "registry.x.com:5000/google_containers/pause:3.9"
       [plugins."io.containerd.grpc.v1.cri".registry]
         config_path = "/etc/containerd/certs.d"
         [plugins."io.containerd.grpc.v1.cri".registry.configs]
           [plugins."io.containerd.grpc.v1.cri".registry.configs."registry.x.com:5000".auth]
             username = "admin"
             password = "123456"
   ```
   这里镜像和私有仓库采用了hosts.toml的方式配置在`/etc/containerd/certs.d`
   ```bash
   [root@k8s-master certs.d]# tree
   .
   ├── docker.io
   │   └── hosts.toml
   ├── registry.k8s.io
   │   └── hosts.toml
   └── registry.x.com:5000
       └── hosts.toml
   [root@k8s-master certs.d]# cat docker.io/hosts.toml
   server = "https://docker.io"
   [host."https://s0a2lqra.mirror.aliyuncs.com"]
     capabilities = ["pull", "resolve"]
   
   [host."https://registry.docker-cn.com"]
     capabilities = ["pull", "resolve"]
   ```
   ctr是containerd的客户端，和docker客户端功能几乎一致，但是很少用。
   可以通过`ctr images pull registry.aliyuncs.com/google_containers/pause:3.9`检查是否配置成功。
   注意：普通的Docker安装不需要这一步，将containerd作为k8s的运行时才需要。有了CRI插件后，已经不需要独立安装cri-dockerd了。
4. 启动docker
   ```bash
   systemctl start docker
   systemctl enable docker # 服务开机启动
   docker run hello-world # 检查是否安装成功
   ```
5. 常用命令
   ```bash
   docker info # 查看docker仓库所有核心参数
   docker images # 当前镜像
   docker pull
   docker tag
   docker push
   docker load # 导入镜像
   docker save # 导出镜像
   docker rmi # 删除镜像
   docker ps # 当前容器
   docker run
   docker stop
   docker ps -a -q -f status=exited | xargs docker rm # 删除所有已经退出的所有容器
   docker inspect # 查看容器内容
   docker logs # 查看容器日志
   docker exec # 在容器内执行命令
   ```

### 安装私有仓库
这里使用registry容器安装，也可以选择安装服务Docker Harbor。
Docker容器使用host网络模式和Host机器共享网络。共享Host机器上的目录`/data/registry`，配置时要注意容器内外目录的映射关系。

1. 准备账号和密码
   ```bash
   mkdir -p /data/registry
   yum install httpd-tools
   htpasswd -Bbn admin 123456 > /data/registry/htpasswd # 生成账号和密码
   ```
2. 准备配置文件  
  `/data/registry/config.yml`配置文件中，除最后的auth其他都是默认值。
   ```yaml
   version: 0.1
   log:
     fields:
       service: registry
   storage:
     cache:
       blobdescriptor: inmemory
     filesystem:
       rootdirectory: /var/lib/registry # 映射的仓库地址
   http:
     addr: :5000
     headers:
       X-Content-Type-Options: [nosniff]
   health:
     storagedriver:
       enabled: true
       interval: 10s
       threshold: 3
   auth:
     htpasswd:
       realm: basic-realm
       path: /var/lib/registry/htpasswd # 映射的账号文件
   ```
1. 安装容器
  将配置和仓库目录映射到外部主机上。 
   ```bash
   docker pull registry
   docker run -d --net=host -v /data/registry/config.yml:/etc/docker/registry/config.yml -v /data/registry:/var/lib/registry docker.io/registry
   docker login registry.x.com:5000 -u admin # 登录
   cat ~/.docker/config.json # 检查登录状态
   docker tag hello-world registry.x.com:5000/hello-world:1.0 # 验证推送
   docker push registry.x.com:5000/hello-world:1.0 # 验证推送
   docker pull registry.x.com:5000/hello-world:1.0 # 验证推送
   ```
   可以从浏览器查看仓库里的镜像`http://192.168.101.128:5000/v2/_catalog`
1. 制作成开机启动
   `/usr/lib/systemd/system/k8s-registry.service`
   ```ini
   [Unit]
   Description=k8s registry
   After=network.target
   
   [Service]
   ExecStart=docker run -d --net=host -v /data/registry/config.yml:/etc/docker/registry/config.yml -v /data/registry:/var/lib/registry docker.io/registry
   Restart=always
   
   [Install]
   WantedBy=multi-user.target
   ```

### K8s集群安装
使用的版本`Kubernetes v1.28.2`
[https://v1-28.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm](https://v1-28.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm "https://v1-28.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm")


##### 集群环境初始化

- 禁用交换分区swap
  ```bash
  swapon -s # 检查
  swapoff -a # 禁止
  ```
  以上是检查命令，永久生效需要在/etc/fstab中注释掉swap那一行。
- 关闭selinux
  `/etc/selinux/config`
  SELINUX=disabled
 关闭防火墙
  `systemctl disable firewalld`
- 内核参数修改
  `sysctl -p /etc/sysctl.d/k8s.conf`
  ```ini
  vm.swappiness=0
  net.ipv4.ip_forward=1
  net.bridge.bridge-nf-call-iptables=1
  net.bridge.bridge-nf-call-ip6tables=1     
  ```
- 添加内核模块
  ```bash
  modprobe overlay
  modprobe br_netfilter
  lsmod #确认命令
  ```
  永久生效`/etc/modules-load.d/k8s-mod.conf`
  ```bash
  overlay
  br_netfilter
  ```

##### 安装kube*程序
- kubeadm 集群初始化相关
- kubectl 服务管理
- kubelet 容器代理
k8s支持很多种运行时，其中之一是docker contianerd，端点为`unix:///run/containerd/containerd.sock`。
新的docker containerd已经支持CRI插件，不再需要单独安装cri-dockerd。

1. kubernetes阿里源
   `/etc/yum.repos.d/kubernetes.repo`文件
   ```ini
   [kubernetes]
   name=Kubernetes
   baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
   enabled=1
   gpgcheck=0
   repo_gpgcheck=0
   gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
   ```
2. 安装程序
   ```bash
   yum install -y kubeadm kubectl kubelet iproute-tc
   systemctl enable kubelet
   ```
   后面kubeadm执行时会自动拉起kubelet
3. crictl
   crictl是k8s体系cri接口的客户端。和docker客户端功能几乎一致，有基本的对应关系。
   下面的命令会将配置写入文件`/etc/crictl.yaml`
   ```bash
   crictl config runtime-endpoint unix:///run/containerd/containerd.sock
   crictl config image-endpoint unix:///run/containerd/containerd.sock
   ```

##### 初始化k8s集群

1. 预拉取镜像到本地仓库
   ```bash
   kubeadm config images list --kubernetes-version v1.28.2 # 查看安装过程依赖哪些镜像
   for i in `kubeadm config images list --kubernetes-version v1.28.2 | awk -F'/' '{print $NF}'`
   do
       docker pull registry.aliyuncs.com/google_containers/$i
       docker tag registry.aliyuncs.com/google_containers/$i registry.x.com:5000/google_containers/$i
       docker push registry.x.com:5000/google_containers/$i
       docker rmi registry.aliyuncs.com/google_containers/$i
   done # 将所有依赖拉取推送到本地仓库registry.x.com:5000
   ```
2. 在Master节点初始化控制平面
   ```bash
   kubeadm init \
        --kubernetes-version 1.28.2 \
        --apiserver-advertise-address=10.0.2.10 \
        --service-cidr=10.96.0.0/12 \
        --pod-network-cidr=10.244.0.0/16 \
        --image-repository registry.x.com:5000/google_containers \
        --ignore-preflight-errors=all
   ```
   service-cidr和pod-network-cidr是默认值，其他选项按实际场景设置。特别的，如果pod-network-cidr没有使用默认值，在安装网络插件时也需要对应的设置。
   该命令会通过端点自动探测运行时，同时在systemd机器上自动选择systemd作为cgroup驱动。
   `Your Kubernetes control-plane has initialized successfully!`
   查看当前所有pods
   ```bash
   [root@k8s-master ~]# kubectl get pods -A
   NAMESPACE      NAME                                 READY   STATUS                  RESTARTS      AGE
   kube-system    coredns-66977f494-7577q              0/1     Pending                 0             147m
   kube-system    coredns-66977f494-989q4              0/1     Pending                 0             147m
   kube-system    etcd-k8s-master                      1/1     Running                 2 (10m ago)   147m
   kube-system    kube-apiserver-k8s-master            1/1     Running                 2 (10m ago)   147m
   kube-system    kube-controller-manager-k8s-master   1/1     Running                 3 (10m ago)   147m
   kube-system    kube-proxy-d6qdb                     1/1     Running                 2 (10m ago)   147m
   kube-system    kube-scheduler-k8s-master            1/1     Running                 3 (10m ago)   147m
   ```
   `kubeadm init`会生成各种配置文件如`/var/lib/kubelet/config.yaml`，失败后停止kubelet，重复执行以上动作。
   重置初始化，停止所有服务使用`kubeadm reset -f` 
3. Node节点加入集群
   ```bash
   kubeadm join 10.0.2.10:6443 \
      --token pz5otp.z5homeufruaylgi7 \
      --discovery-token-ca-cert-hash sha256:168ebe6aa7bd2cf8f39f046965b023e80b9718eeaea1c5d8c95715afcd2a0014 \
      --ignore-preflight-errors=all
   ```
   查看当前集权的token和discovery-token-ca-cert-hash值。
   ```bash
   kubeadm token list # 24小时变化一个
   openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | \
   openssl rsa -pubin -outform der 2>/dev/null | \
   openssl dgst -sha256 -hex | sed 's/^.* //' # ca证书的hash值，kubeadm init时确定
   ```
4. 安装网络插件flannel
   上面的coredns是Pending状态，需要安装网络插件。
   这里选择Flannel（因为简单）。Calico支持更多高级设置，Cillium使用了eBPF。k8s的网络规范，对网络插件的要求见[https://kubernetes.io/docs/concepts/services-networking](https://kubernetes.io/docs/concepts/services-networking "https://kubernetes.io/docs/concepts/services-networking")
   
   ```bash
   # 从github下载镜像flannel/flannel到私有仓库
   wget https://github.com/flannel-io/flannel/releases/download/v0.26.2/flanneld-v0.26.2-amd64.docker
   docker load --input flanneld-v0.26.2-amd64.docker
   docker tag quay.io/coreos/flannel:v0.26.2-amd64 registry.x.com:5000/flannel/flannel:v0.26.2-amd64
   docker push registry.x.com:5000/flannel/flannel:v0.26.2-amd64
   # 从国内源下载镜像flannel/flannel-cni-plugin到私有仓库
   docker pull swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/flannel/flannel-cni-plugin:v1.6.0-flannel1
   docker tag swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/flannel/flannel-cni-plugin:v1.6.0-flannel1 registry.x.com:5000/flannel/flannel-cni-plugin:v1.6.0-flannel1
   docker push registry.x.com:5000/flannel/flannel-cni-plugin:v1.6.0-flannel1
   # 下载kube-flannel.yml
   wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
   kubectl apply -f kube-flannel.yml # 将这个文件里的源对应替换掉成私有仓库之后执行
   ```

   kube-flannel.yml里使用的vxlan组网模式。
   创建的资源有：kube-flannel（Namespace），flannel（ServiceAccount），flannel（ClusterRole），kube-flannel-cfg（ConfigMap），kube-flannel-ds（DaemonSet）。

   集群重新初始化时，flannel以及cni网络也需要清理
   ```bash
   kubectl delete -f kube-flannel.yml
   ip link set cni0 down
   ip link delete cni0
   ip link delete flannel.1
   rm -rf /run/flannel/
   rm -rf /var/lib/cni/
   rm -rf /etc/cni/
   iptables -F
   iptables -t nat -F
   ```

集群安装完毕后的状态
```bash
[root@k8s-master ~]# kubectl get nodes
NAME         STATUS   ROLES           AGE   VERSION
k8s-master   Ready    control-plane   16h   v1.28.2
k8s-node1    Ready    <none>          13h   v1.28.2
k8s-node2    Ready    <none>          13h   v1.28.2
k8s-node3    Ready    <none>          13h   v1.28.2
[root@k8s-master ~]# ip route
default via 10.0.2.1 dev enp0s3 proto static metric 100
10.0.2.0/24 dev enp0s3 proto kernel scope link src 10.0.2.10 metric 100 # enp0s3物理接口
10.244.0.0/24 dev cni0 proto kernel scope link src 10.244.0.1 # cni接口
10.244.1.0/24 via 10.244.1.0 dev flannel.1 onlink # flannel子网
10.244.2.0/24 via 10.244.2.0 dev flannel.1 onlink # flannel子网
10.244.3.0/24 via 10.244.3.0 dev flannel.1 onlink # flannel子网
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown # docker网桥
```


### 部署应用

##### 部署无状态服务Deployment
通过私有仓库部署一个nginx应用。配置亲和度使其尽量分配到不同的Node节点上。
```bash
[root@k8s-master ~]# kubectl apply -f nginx-deployment.yaml
deployment.apps/nginx-deployment configured
[root@k8s-master ~]# cat nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2 # 副本个数
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: registry.x.com:5000/nginx:latest # 私有仓库
        ports:
        - containerPort: 80
      affinity:
        nodeAffinity: # 节点亲和度
          requiredDuringSchedulingIgnoredDuringExecution: # 强制
            nodeSelectorTerms:
            - matchExpressions: # amd64机器的节点
              - key: kubernetes.io/arch
                operator: In
                values:
                - amd64
        podAntiAffinity: # pod反亲和性
          preferredDuringSchedulingIgnoredDuringExecution: # 软强制
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions: # master之外的其他节点
                - key: kubernetes.io/hostname
                  operator: NotIn
                  values:
                  - k8s-master
              topologyKey: kubernetes.io/hostname
```

通过`curl http://10.244.3.2`或者`curl http://10.244.2.3`可以访问Nginx服务。
注意在k8s-registry机器上是访问不到的，因为它没有加入集群，不属于k8s网络。

##### 对外暴露service
```bash
[root@k8s-master ~]# kubectl apply -f nginx-service.yaml
service/nginx-service configured
[root@k8s-master ~]# cat nginx-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort # 默认类型是ClusterIP，分配一个集群虚拟IP比如10.108.49.219，这种IP只能集群内访问。还需要GatewayAPI的配合
  selector:
    app: nginx
  ports: # 支持暴露为多个端口
    - name: http
      protocol: TCP
      port: 80 # 暴露服务端口
      targetPort: 80 # 后端nginx-deployment端口
      nodePort: 30000 # 控制平面默认分配范围：30000-32767
```
NodePort类型的Service可以通过任何一个集群主机的IP访问（kubelet-proxy的转发作用）。比如
```bash
curl -v http://10.0.2.10:30000 # master机器
curl -v http://10.0.2.13:30000 # 任一节点机器
```

##### 当前集群状态
```bash
[root@k8s-master ~]# kubectl get services -A -o wide
NAMESPACE     NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                  AGE   SELECTOR
default       kubernetes      ClusterIP   10.96.0.1       <none>        443/TCP                  19h   <none>
default       nginx-service   NodePort    10.108.49.219   <none>        80:30000/TCP             3h    app=nginx
kube-system   kube-dns        ClusterIP   10.96.0.10      <none>        53/UDP,53/TCP,9153/TCP   19h   k8s-app=kube-dns
[root@k8s-master ~]# kubectl get deployments -A -o wide
NAMESPACE     NAME               READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES                                                  SELECTOR
default       nginx-deployment   2/2     2            2           16h   nginx        registry.x.com:5000/nginx:latest                        app=nginx
kube-system   coredns            2/2     2            2           19h   coredns      registry.x.com:5000/google_containers/coredns:v1.10.1   k8s-app=kube-dns
[root@k8s-master ~]# kubectl get pods -A -o wide
NAMESPACE      NAME                                 READY   STATUS    RESTARTS       AGE     IP           NODE         NOMINATED NODE   READINESS GATES
default        nginx-deployment-7bf65ddb84-95rjq    1/1     Running   0              6m23s   10.244.3.2   k8s-node3    <none>           <none>
default        nginx-deployment-7bf65ddb84-98z6n    1/1     Running   0              6m31s   10.244.2.3   k8s-node2    <none>           <none>
kube-flannel   kube-flannel-ds-6bskk                1/1     Running   1 (139m ago)   15h     10.0.2.10    k8s-master   <none>           <none>
kube-flannel   kube-flannel-ds-wnspn                1/1     Running   1 (138m ago)   15h     10.0.2.13    k8s-node3    <none>           <none>
kube-flannel   kube-flannel-ds-x9hwb                1/1     Running   1 (138m ago)   15h     10.0.2.11    k8s-node1    <none>           <none>
kube-flannel   kube-flannel-ds-z7bpc                1/1     Running   1 (139m ago)   15h     10.0.2.12    k8s-node2    <none>           <none>
kube-system    coredns-66977f494-2jjgs              1/1     Running   1 (138m ago)   15h     10.244.1.8   k8s-node1    <none>           <none>
kube-system    coredns-66977f494-hm2qc              1/1     Running   1 (139m ago)   18h     10.244.0.3   k8s-master   <none>           <none>
kube-system    etcd-k8s-master                      1/1     Running   7 (139m ago)   18h     10.0.2.10    k8s-master   <none>           <none>
kube-system    kube-apiserver-k8s-master            1/1     Running   7 (139m ago)   18h     10.0.2.10    k8s-master   <none>           <none>
kube-system    kube-controller-manager-k8s-master   1/1     Running   12 (27m ago)   18h     10.0.2.10    k8s-master   <none>           <none>
kube-system    kube-proxy-8j9wh                     1/1     Running   2 (139m ago)   18h     10.0.2.10    k8s-master   <none>           <none>
kube-system    kube-proxy-9mlg7                     1/1     Running   1 (138m ago)   15h     10.0.2.13    k8s-node3    <none>           <none>
kube-system    kube-proxy-ptt4g                     1/1     Running   1 (139m ago)   15h     10.0.2.12    k8s-node2    <none>           <none>
kube-system    kube-proxy-tx692                     1/1     Running   1 (138m ago)   15h     10.0.2.11    k8s-node1    <none>           <none>
kube-system    kube-scheduler-k8s-master            1/1     Running   19 (27m ago)   18h     10.0.2.10    k8s-master   <none>           <none>
```


### 其他

- 命令自动补全
  在~/.bashrc里加入下面两行
  ```bash
  source <(kubeadm completion bash)
  source <(kubectl completion bash)
  ```
- 查看kubelet日志
  `journalctl -u kubelet -f`
- 查看pod内容
  `kubectl describe pod -n kube-flannel kube-flannel-ds-pnzx4`
- 查看容器
  ```bash
  crictl pods # pod列表
  crictl statsp # pod资源使用
  crictl inspectp 932e5f7017ae3 # 容器详情  
  crictl ps # 容器列表
  crictl stats # 容器资源使用
  crictl inspect eb5b29d988e9d # 容器详情
  crictl logs -f bc0c3ff985a57 # 容器日志
  ```
- kubeadm是怎么控制kubelet的？
  kubeadm会将kubelet需要的信息写到配置文件中，然后重启它。
  ```ini
  [root@k8s-master kubelet.service.d]# cat /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
  # Note: This dropin only works with kubeadm and kubelet v1.11+
  [Service]
  Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
  Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
  # This is a file that "kubeadm init" and "kubeadm join" generates at runtime, populating the KUBELET_KUBEADM_ARGS variable dynamically
  EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
  # This is a file that the user can use for overrides of the kubelet args as a last resort. Preferably, the user should use
  # the .NodeRegistration.KubeletExtraArgs object in the configuration files instead. KUBELET_EXTRA_ARGS should be sourced from this file.
  EnvironmentFile=-/etc/sysconfig/kubelet
  ExecStart=
  ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS  
  ```