### 前言
前面已经使用kubeadm初始化了一个k8s集群，部署了2个副本的Nignx Service，且外部可访问。但是对于Product环境，还是不够的。需要进一步做一些可观察性、可靠性相关的部署。

### 控制平面
控制平面就是k8s集群的大脑。作为一个高可用的基础设施，大部分组件如API Server/Controller/Scheduler都应该部署多个副本。
ETCD作为集群的数据中心，更是重中之中，推荐部署在k8s集群外部（不推荐默认的Stacked模式）：

- ETCD是集群所有信息的唯一来源
- ETCD还为k8s其他组件分的布式部署提供支持，比如Leader选举

![https://ping666.com/wp-content/uploads/2024/12/kubeadm-ha-topology-external-etcd.jpg](https://ping666.com/wp-content/uploads/2024/12/kubeadm-ha-topology-external-etcd.jpg "kubeadm-ha-topology-external-etcd.jpg")

可以通过`kubeadm config print init-defaults > k8s.yaml`输出默认的初始化参数。
`--component-configs KubeletConfiguration --component-configs KubeProxyConfiguration`参数可以给出多个具体组件的参数细节。
```yaml
apiVersion: kubeadm.k8s.io/v1beta3
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 1.2.3.4
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
  imagePullPolicy: IfNotPresent
  name: node
  taints: null
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  # 这里通过external参数配置外部ETCD入口
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.k8s.io
kind: ClusterConfiguration
kubernetesVersion: 1.28.0
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
scheduler: {}
```
通过以上文件控制`kubeadm init --config k8s.yaml`的行为，其他安装步骤和默认的一致。
一般部署3套控制平面，然后将API Server负载均衡后，`kubelet join`提供给其他kubelet节点使用。

### 持久化存储StorageClass
k8s还在支持的存储类型有很多：EmptyDir（默认）、hostPath（local类似但是可以使用裸设备）、nfs、persistentVolumeClaim。可以直接projected的类型有secret、serviceAccountToken、configMap、downwardAPI。
可以在集群中先创建一批PV，然后Pod中通过PVC申请绑定使用。更好的做法使用StorageClass动态申请。
查看集群存储信息的命令
`kubectl get pv,pvc,sc -A`

云环境一般直接购买ceph RDB服务，这里采用NFS的方式搭建持久化存储。
[https://v1-28.docs.kubernetes.io/docs/concepts/storage/storage-classes/#nfs](https://v1-28.docs.kubernetes.io/docs/concepts/storage/storage-classes/#nfs "https://v1-28.docs.kubernetes.io/docs/concepts/storage/storage-classes/#nfs")

- 部署NFS服务
  新建虚拟机10.0.2.100 k8s-ceph-rdb
  ```bash
  yum install nfs-utils rpcbind # 客户端只需要安装nfs-utils
  [root@k8s-ceph-rdb ~]# cat /etc/exports # 准备配置文件
  /data/nfs *(rw,sync,all_squash,anonuid=0,anongid=0)
  systemctl start nfs-server.service # 启动服务
  systemctl start rpcbind.service
  systemctl enable nfs-server.service
  systemctl enable rpcbind.service
  [root@k8s-ceph-rdb ~]# showmount -e 127.0.0.1 # 查看挂载目录
  Export list for 127.0.0.1:
  /data/nfs *
  firewall-cmd --zone=trusted --add-source=10.0.2.0/24 --permanent # 将客户端网段加入信任区
  firewall-cmd --reload
  firewall-cmd --list-all --zone=trusted
  ```
  客户端机器挂载测试。
  ```bash
  mkdir -p /data/shared
  mount -t nfs 10.0.2.100:/data/nfs /data/shared
  [root@k8s-registry shared]# df | grep nfs # 检查挂载点
  10.0.2.100:/data/nfs  17811456  2962688  14848768  17% /data/shared
  ```

- 部署StorageClass
  StorageClass需要一个provisioner（运行NFS客户端的Pod）转接到NFS服务。
  ```yaml
  apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
    name: nfs-client
  provisioner: k8s-sigs.io/nfs-subdir-external-provisioner # or choose another name, must match deployment's env PROVISIONER_NAME'
  parameters:
    archiveOnDelete: "false"
  ```
  k8s没有为NFS提供内置的provisioner，需要额外安装。
  这里使用[https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner "https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner")
  从这里下载部署需要的资源文件`https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/tree/master/deploy/*yaml`
  其中deployment.yaml需要做几处修改

  - 镜像改到到私有仓库
  - 修改NFS源
  - 关闭选举（只有1个副本，选举会失败）
  
  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: nfs-client-provisioner
    labels:
      app: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: default
  spec:
    replicas: 1
    strategy:
      type: Recreate
    selector:
      matchLabels:
        app: nfs-client-provisioner
    template:
      metadata:
        labels:
          app: nfs-client-provisioner
      spec:
        serviceAccountName: nfs-client-provisioner
        containers:
          - name: nfs-client-provisioner
            # 私有镜像
            image: registry.x.com:5000/sig-storage/nfs-subdir-external-provisioner:v4.0.2
            volumeMounts:
              - name: nfs-client-root
                mountPath: /persistentvolumes
            env:
              - name: PROVISIONER_NAME
                value: k8s-sigs.io/nfs-subdir-external-provisioner
              - name: NFS_SERVER
                value: 10.0.2.100
              - name: NFS_PATH
                value: /data/nfs
              # 只有1个副本，关闭选举
              - name: ENABLE_LEADER_ELECTION
                value: "false"
        volumes:
          - name: nfs-client-root
            # NFS服务配置
            nfs:
              server: 10.0.2.100
              path: /data/nfs
  ```
  然后安装。
  ```bash
  kubectl apply -k . # 运行当前目录下的kustomization.yaml
  ```
  每个k8s节点都需要安装NFS客户端nfs-utils。provisioner虽然是在Pod中mount的，但是和NFS服务交互的是节点上的客户端。

- 设置默认StorageClass
  将nfs-client设置为默认StorageClass，后面的Web管理端需要有一个默认的StorageClass。
  ```bash
  kubectl patch storageclass nfs-client -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}' # 默认StorageClass只会有一个
  [root@k8s-master nfs]# kubectl get sc # 检查
  NAME                   PROVISIONER                                   RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
  nfs-client (default)   k8s-sigs.io/nfs-subdir-external-provisioner   Delete          Immediate           false                  3h45m
  ```

### Web管理端
官方提供的管理端是kubernetes-dashboard。
第三方的管理端[https://github.com/portainer/portainer.git](https://github.com/portainer/portainer.git "https://github.com/portainer/portainer.git")
Kubernetes v1.28.0 Aug 16, 2023
Kubernetes v1.28.2 Sep 14, 2023
v1.28.2算比较新的版本，使用portainer/portainer-ce:2.21.5。

- 安装kubernetes-dashboard
  官方的kubernetes-dashboard使用helm管理。
  [https://get.helm.sh/helm-v3.16.4-linux-amd64.tar.gz](https://get.helm.sh/helm-v3.16.4-linux-amd64.tar.gz "https://get.helm.sh/helm-v3.16.4-linux-amd64.tar.gz")
  解压后将helm复制到/usr/local/bin/下。然后安装
  ```bash
  helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
  helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard
  ```
  阿里云的源中下载这个镜像失败。放弃。
  删除。
  `helm uninstall kubernetes-dashboard --namespace kubernetes-dashboard`

- 安装portainer
  ```bash
  docker pull swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/portainer/portainer-ce:2.21.5
  docker tag  swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/portainer/portainer-ce:2.21.5  registry.x.com:5000/portainer/portainer-ce:2.21.5
  docker push registry.x.com:5000/portainer/portainer-ce:2.21.5 # 下载镜像到私有仓库
  wget https://downloads.portainer.io/ce2-21/portainer.yaml
  kubectl apply -f portainer.yaml
  ```
  portainer会在NFS上申请10G的空间。
  ```bash
  kubectl get pv
  NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                 STORAGECLASS   REASON   AGE
  pvc-c1c2adfe-1a02-436e-bd6c-39dd186b0673   10Gi       RWO            Delete           Bound    portainer/portainer   nfs-client              63m
  ls -al /data/nfs/portainer-portainer-pvc-c1c2adfe-1a02-436e-bd6c-39dd186b0673
  total 44
  drwxrwxrwx 8 root root   153 Dec 24 22:21 .
  drwxr-xr-x 3 root root    74 Dec 24 23:05 ..
  drwx------ 2 root root     6 Dec 24 22:21 bin
  drwx------ 2 root root    37 Dec 24 22:21 certs
  drwx------ 2 root root    29 Dec 24 22:21 chisel
  drwx------ 2 root root     6 Dec 24 22:21 compose
  drwx------ 2 root root    25 Dec 24 22:21 docker_config
  -rw------- 1 root root 65536 Dec 24 23:19 portainer.db
  -rw------- 1 root root   227 Dec 24 22:21 portainer.key
  -rw------- 1 root root   190 Dec 24 22:21 portainer.pub
  drwx------ 2 root root     6 Dec 24 22:21 tls
  ```

- 管理端体验
  http://localhost:30777
  kubectl能看到的大部分内容，这里都有。强在够直观。
  ![https://ping666.com/wp-content/uploads/2024/12/portainer-web.png](https://ping666.com/wp-content/uploads/2024/12/portainer-web.png "portainer-web.png")

### 网络插件calico
flannel只适合在测试环境使用，实际生产环境中推荐使用calico，支持RBAC和网络策略等配置。
calico的版本和k8s有很强的对应关系，k8s1.28对应的版本为3.28。参考
[https://docs.tigera.io/calico/3.28.2/getting-started/kubernetes/requirements#kubernetes-requirements](https://docs.tigera.io/calico/3.28.2/getting-started/kubernetes/requirements#kubernetes-requirements "https://docs.tigera.io/calico/3.28.2/getting-started/kubernetes/requirements#kubernetes-requirements")

- 安装
  ```bash
  # 文件在这里https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/calico.yaml
  kubectl apply -f tigera-operator.yaml --server-side
  kubectl apply -f custom-resources.yaml --server-side
  kubectl apply -f calico.yaml --server-side
  ```

- 删除
  除了以上k8s资源外，还需要删除的内容有
  ```bash
  modprobe -r ipip # 删除tunl0
  rm -rf /etc/cni/net.d/*calico* # 删除cni下的calico相关内容 
  ```
  然后重启kubelet和coredns等。

### 监控Prometheus
[https://github.com/prometheus-operator/kube-prometheus](https://github.com/prometheus-operator/kube-prometheus "https://github.com/prometheus-operator/kube-prometheus")
kube-prometheus是prometheus项目在k8s环境的适配。主打端到端的一键安装，会在名字空间monitoring下部署以下组件：
- prometheus-operator 管理prometheus和alertmanager
- prometheus-k8s 采集和存储指标数据
- node-exporter 节点指标
- kube-state-metrics k8s对象指标
- blackbox-exporter 黑盒监控指标
- prometheus-adapter 适配k8s自定义指标  
- alertmanager 告警
- grafana 数据浏览

安装前需要修改的地方有
- grafana-deployment.yaml
  镜像grafana/grafana:11.2.0
- kubeStateMetrics-deployment.yaml
  镜像registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.13.0
- prometheusAdapter-deployment.yaml
  镜像registry.k8s.io/prometheus-adapter/prometheus-adapter:v0.12.0
- grafana-service.yaml
  修改类型以方便访问`type: NodePort`
  默认端口为3000  
- prometheus-service.yaml
  `type: NodePort`
  默认端口8080和9090
- prometheus-prometheus.yaml
  加入storageClass支持监控数据持久化。

```bash
wget https://github.com/prometheus-operator/kube-prometheus/archive/refs/tags/v0.14.0.zip
unzip v0.14.0.zip
cd kube-prometheus-0.14.0/
kubectl apply --server-side -f manifests/setup # 创建名字空间monitoring和10个自定义资源CustomResourceDefinition
kubectl wait \
	--for condition=Established \
	--all CustomResourceDefinition \
	--namespace=monitoring
kubectl apply -f manifests/ # 在monitoring下安装一批组件
kubectl delete networkpolicy --all -n monitoring # 删除全部网路策略，flannel不支持
```

### GateWay API
对外暴露组件已经废弃了Ingress。


**提示：**
镜像下载失败的终极解决方案
[https://docker.aityp.com](https://docker.aityp.com "https://docker.aityp.com")
手动下载到本地，然后加载到本地私有仓库，再手动修改yaml安装。
