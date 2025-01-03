### 前言
在k8s集群的进阶部署中，已经越来越感觉到从原生的kubectl命令部署应用服务非常繁琐。虽然有kustomize和helm等工具集协助，但是netpolicy，RBAC等还是非常不容易维护。
前面部署的k8s管理端portainer，是完全非侵入式的（不会在节点上部署agent），更多的是集群信息浏览，集群管理功能有限。有两个侵入式的管理端非常强大，一个是KubeSphere，一个是Rancher。前者聚焦k8s开发侧全流程，后者聚焦k8s运维侧管理。

### Rancher架构

Rancher的口号是`Run Kubernetes Everywhere`
提供了两套部署k8s集群的工具集：

- k3s
  目标是使用最少的资源跑起来k8s集群，一般小集群常用，比如开发和测试环境。
- rke
  Rancher Kubernetes Engine，企业级的k8s集群管理。

![https://ping666.com/wp-content/uploads/2025/01/rancher-architecture.png](https://ping666.com/wp-content/uploads/2025/01/rancher-architecture.png "rancher-architecture.png")

### Rancher部署
使用Docker方式部署Rancher，会将Rancher运行在一个k3s集群上。
这里在openEuler24.03LTS上使用Docker方式部署Rancher。目标是使用Rancher管理2个k8s集群：1个使用RKE创建的k8s集群，1个使用kubeadm创建的k8s集群。

- 部署Rancher
  ```bash
  docker pull docker.io/rancher/rancher:v2.9.3
  docker pull docker.io/rancher-agent:v2.9.3
  docker pull docker.io/rancher/shell:v0.2.2
  mkdir -p /data/rancher
  docker run --privileged -d --restart=unless-stopped -p 80:80 -p 443:443 -v /data/rancher:/var/lib/rancher -e CATTLE_AGENT_IMAGE="registry.x.com:5000/rancher/rancher-agent:v2.9.3" registry.x.com:5000/rancher/rancher:v2.9.3
  ```
  建议机器内存6G以上，同时启动时间比较长(10min)。
  ![https://ping666.com/wp-content/uploads/2025/01/rancher-web-home.png](https://ping666.com/wp-content/uploads/2025/01/rancher-web-home.png "rancher-web-home.png")
- 创建k8s集群
  点击Create创建集群，选Custom创建原生的k8s。然后将这一串命令在需要加入集群的机器上执行。
  `curl -kfL https://192.168.101.128/system-agent-install.sh | sudo sh -s - --server https://192.168.101.128 --label 'cattle.io/os=linux' --token d4s99njf27jjrfwrxwgwpkjkhkbrpnqn7wslrdn9sfxtkbxwhlkvsh --ca-checksum 335ba6e14a4b670d7c64620bf8877fc5289b26430fad7bc6be106c8243359649 --etcd --controlplane --worker`
  多添加的-k参数是为了忽略自签名证书。
  加入集群的机器上不需要做任何初始化，除了关闭防火墙和设置net.ipv4.net_forward=1
- 管理已有k8s集群
  点击Imported管理已有集群。在已有集群上执行以下命令
  `curl --insecure -sfL https://192.168.101.128/v3/import/cpvbr8hbj5w6qhswmrlmt48c6lvw8kspz5k6q89d2kkl8vvfs86qms_c-m-fw6t4wk2.yaml | kubectl apply -f -`
  会在被管理集群上创建名字空间cattle-system，包括cattle-cluster-agent、helm-operation等。

Rancher部署和管理k8s都非常简单，比较麻烦的是镜像拉取不到的问题。
