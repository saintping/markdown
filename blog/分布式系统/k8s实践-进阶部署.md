### 前言
前面已经使用kubeadm初始化了一个k8s集群，部署了2个副本的Nignx Service，且外部可访问。但是对于Product环境，还是不够的。需要进一步做一些可观察性、可靠性相关的部署。

### 持久化存储StorageClass
k8s还在支持的存储类型有很多：EmptyDir（默认）、hostPath（local类似但是可以使用裸设备）、nfs、persistentVolumeClaim。可以直接projected的类型有secret、serviceAccountToken、configMap、downwardAPI。
可以在集群中先创建一批PV，然后Pod中通过PVC申请绑定使用。更好的做法使用StorageClass动态申请。
查看集群存储信息的命令
`kubectl get pv,pvc,sc -A`

这里采用StorageClass + NFS的方式搭建持久化存储。
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
  ```ini
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
  需要部署的资源文件都在这里`https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/tree/master/deploy/*yaml`
  下载到本地，替换掉deployment.yaml里面的镜像到私有仓库&修改NFS源。
  ```ini
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
            image: registry.x.com:5000/sig-storage/nfs-subdir-external-provisioner:v4.0.2 # 私有镜像
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
        volumes:
          - name: nfs-client-root
            nfs: # NFS服务配置
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


### Web管理端Dashboard
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
  wget https://downloads.portainer.io/ce2-21/portainer.yaml # 修改镜像到私有仓库
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

- 访问
  http://localhost:30777/
  kubectl能看到的大部分内容，这里都有。强在够直观。
  ![https://ping666.com/wp-content/uploads/2024/12/portainer-web.png](https://ping666.com/wp-content/uploads/2024/12/portainer-web.png "portainer-web.png")

### DaemonSet


### GateWay API


**提示：**
镜像下载失败的终极解决方案
[https://docker.aityp.com](https://docker.aityp.com "https://docker.aityp.com")
手动下载到本地，然后加载到本地私有仓库，再手动修改yaml安装。
