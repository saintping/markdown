### 前言
在开发侧基于Git、Jenkis、Junit等工具构件了一个完善的CI（Continuous Integration持续集成）环境之后，CD（Continuous Delivery持续交付）往往成为下一个大问题。

从系统集成测试，用户体验测试，预生产测试，到最后的线上发布都是不同的环境，很多配置都是环境相关的，一个配置有问题整个服务就work不起来。debug半天后发现要么是有人动了个文件，要么是有人动了项配置。程序员遇到这种问题的第一反应都是reset。在容器技术出来之前，重新部署服务是个很要命的事。一杯茶，一包烟，一个配置一整天。

Docker镜像 + K8s服务编排完美的解决了CD难题。
Docker镜像解决了：一次打包哪里都能运行（是不是和Java口号很重合(￣▽￣)"），而K8s解决了：给我个镜像还你个Ready的服务。一切意思上的Ready。

K8s开源项目地址[https://github.com/kubernetes/kubernetes.git](https://github.com/kubernetes/kubernetes.git "https://github.com/kubernetes/kubernetes.git")

### K8s架构

![k8s.png](https://ping666.com/wp-content/uploads/2024/10/k8s.png "k8s.png")

可以看到这个结构和Hadoop Yarn是很像的。K8s服务提供了：

- 自动部署
- 服务编排
- 服务发现
- 负载均衡
- 自动扩容

核心组件介绍：

- 控制平面
  - API Server
    所有用户操作的入口（比如来自kubectl的命令）。提供List-Watch，同时模块之间的访问都统一经过这里。无状态服务，多实例部署。
  - etcd
    持久化的KV存储系统，用户通过API Server把Kubernetes Object（比如deployment.yaml）写入etcd。一般以外部集群模式部署。
  - Scheduler
    服务编排。从调度队列获取命令，经过过滤和打分，将Pod下发到合适的Node上运行。无状态服务，多实例部署。
![k8s-schedule.png](https://ping666.com/wp-content/uploads/2024/10/k8s-schedule.png "k8s-schedule.png")
  - Controller
    实现集群的故障检测和自动恢复。Contoller是一个大循环程序：通过List-Watch对比实际状态和期望状态，确保保持一致。每种资源都对应一种Controller，比如Deployment Controller、ReplicaSet Controller、StatefulSet Controller等。多实例部署时内部Raft算法选举一个Leader生效。

- 工作节点Node
  - Kubelet
    控制平面在节点机器上的代理，接收命令和上报Pod状态。
  - Pod
    容器管理，在这里运行镜像。
  - Kube-proxy
    服务的网络代理，K8s的性能核心。

### 常见工作负载

- Deployments
  典型的无状态服务
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
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
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```
- StatefulSet
  有状态服务，比如挂载持久卷（Persistent Volumes）
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx # 必须匹配 .spec.template.metadata.labels
  serviceName: "nginx"
  replicas: 3 # 默认值是 1
  minReadySeconds: 10 # 默认值是 0
  template:
    metadata:
      labels:
        app: nginx # 必须匹配 .spec.selector.matchLabels
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: registry.k8s.io/nginx-slim:0.24
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
    metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "my-storage-class"
      resources:
        requests:
          storage: 1Gi
```
- Ingress
  对外暴露服务
- DaemonSet
  应用服务的伴随任务，比如logstash
- ReplicaSet
  副本服务
- Job
  一次性任务

### Docker镜像Image
Docker镜像的核心技术是UnionFS，一个支持分层Layer的文件系统。除了Image，Docker也有自己的容器编排技术Docker Compose/Machine/Swarm。不过自从K8s流行起来后，K8s编排 + Docker镜像已经成为主流。

![docker-architecture.jpg](https://ping666.com/wp-content/uploads/2024/11/docker-architecture.jpg "docker-architecture.jpg")

- 构建Docker镜像
  `docker build -t spring-boot-3 -f Dockerfile .`
  以下是Dockerfile文件的内容，将一个典型的spring boot服务打包成镜像。
```asp
FROM openjdk:17
ARG JAR_FILE=spring-boot-3-0.0.1-SNAPSHOT.jar
COPY ${JAR_FILE} app.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","/app.jar"]
```

- Docker常用命令
  Docker大部分命令在Docker Desktop界面上都有对应的功能。
```bash
docker images
docker search redis
docker pull redis
docker run --hostname=aa7070c1db5c --mac-address=02:42:ac:11:00:02 --env=PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin --env=GOSU_VERSION=1.17 --env=REDIS_VERSION=7.4.1 --env=REDIS_DOWNLOAD_URL=http://download.redis.io/releases/redis-7.4.1.tar.gz --env=REDIS_DOWNLOAD_SHA=bc34b878eb89421bbfca6fa78752343bf37af312a09eb0fae47c9575977dfaa2 --volume=/data --network=bridge --workdir=/data -p 6379:6379 --restart=no --runtime=runc -d redis:latest
docker ps
docker exec b833b75bc340 ls /
docker push spring-boot-3
```

- 国内镜像源
  Docker官方镜像仓库，网络太慢了，一般需要使用国内的镜像源。而且国内的镜像源也不稳定，需要随时准备更新和切换。
```json
{
  "registry-mirrors": [
    "https://dockerpull.org",
    "https://dockerhub.icu"
  ]
}
```

- 本地仓库
  官方镜像仓库一般只是用来拉取公共镜像，自己开发的私有镜像一般放在本地私有仓库。通过以下命令可以创建一个私有镜像仓库服务。
```bash
# 创建私有镜像源
docker pull registry
docker run -d -p 5000:5000 --restart=always --name local-registry registry:2
# 推送镜像到私有源
docker tag nginx localhost:5000/nginx
docker push localhost:5000/nginx
# 就是否推送成功
curl -X GET http://localhost:5000/v2/nginx/tags/list
```

### Minikube部署
如果只是在自己的开发机器上使用镜像和容器，Docker Desktop已经足够了。如果是为了学习和实践K8s编排部分，那么使用Minikube搭建一个本地K8s集群是一个有效途径。

- 安装Minikube
- 打开Windows虚拟化
  Windows需要专业版或者企业版才支持虚拟化。
`Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All`
- 创建集群
  官方使用的镜像base-image拉取很慢，先pull替代镜像anjone/kicbase到本地。
`minikube start --no-vtx-check --vm-driver=docker --base-image='anjone/kicbase' --image-mirror-country='cn' --registry-mirror='https://dockerpull.org' --kubernetes-version=v1.23.8`
- 用户界面
  `minikube dashboard`

- 集群状态
```bash
PS > kubectl get pods -A
NAMESPACE     NAME                               READY   STATUS             RESTARTS        AGE
kube-system   coredns-65c54cc984-w9lxd           0/1     CrashLoopBackOff   22 (43s ago)    90m
kube-system   etcd-minikube                      1/1     Running            1 (3m20s ago)   90m
kube-system   kube-apiserver-minikube            1/1     Running            1 (3m10s ago)   90m
kube-system   kube-controller-manager-minikube   1/1     Running            1 (3m20s ago)   90m
kube-system   kube-proxy-mdz8r                   1/1     Running            1 (3m20s ago)   90m
kube-system   kube-scheduler-minikube            1/1     Running            1 (3m10s ago)   90m
kube-system   storage-provisioner                1/1     Running            1 (3m20s ago)   90m
```

**注意：**

- K8s 1.24以后的版本，放弃了对DockerShim的支持
  需要额外部署一个cri-dockerd.exe，这个有点麻烦。
  可以在Docker Desktop上启动支持K8s，这大概会启动18个相关容器。然后就可以使用kubectl命令来操作K8s集群了。


### 部署服务

- 以deployment部署一个nginx
  `kubectl create deployment nginx --image=nginx:latest`
  可以通过`kubectl get deployment nginx -o yaml`来查看这个deployment的详情。
  可以看到deployment下的pod被删除后，K8s会自动再启动一个pod。这就是K8s的核心能力。

- 以NodePort对外暴露服务
  `kubectl apply -f nginx-nodeport.yaml`
  然后就可以访问nginx了http://localhost:30080
  nginx-nodeport.yaml文件内容如下：
```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
   app: nginx # app=nginx标签
  name: nginx-service # 本服务名
spec:
  ports:
  - port: 8080 # 服务映射端口
    name: nginx-service # 本服务名
    protocol: TCP
    targetPort: 80 # 要暴露的服务端口
    nodePort: 30080 # 对外暴露的端口
  selector:
    app: nginx # 要暴露的服务名
  type: NodePort
```

### 常用命令
更多命令在[https://minikube.sigs.k8s.io/docs/commands/](https://minikube.sigs.k8s.io/docs/commands/ "https://minikube.sigs.k8s.io/docs/commands/")

- Minikube集群状态
```bash
minikube start --no-vtx-check --vm-driver=docker --base-image='anjone/kicbase' --image-mirror-country='cn' --registry-mirror='https://dockerpull.org'
minikube stop
minikube delete --all --purge
minikube image ls --format table
minikube ssh
```

- Minikube内部K8s服务
  K8s的命令非常多，一般都直接使用工具来管理了，比如Portainer。
```bash
kubectl get nodes
kubectl describe node 'minikube'
kubectl get service
kubectl describe service 'kubernetes'
kubectl run nginx --image=nginx:latest --port=80
kubectl delete pod nginx
kubectl get deployments
kubectl create deployment nginx --image=nginx:latest
kubectl get deployment nginx -o yaml
kubectl apply -f nginx-nodeport.yaml
kubectl delete deployment 'nginx'
kubectl get pods -A
kubectl describe pod 'nginx'
kubectl attach POD
kubectl exec POD
kubectl logs POD
```

### 网络

- Docker容器网络
  Docker Desktop启动容器时默认使用bridge桥接模式。桥接模式网络隔离的最彻底，很方便测试环境使用，只是数据转发效率有点低。其他3种类型分别是none（无网络），host（和主机共享网络），container（和其他容器共享网络）。
![docker-bridge.png](https://ping666.com/wp-content/uploads/2024/11/docker-bridge.png "docker-bridge.png")

- K8s网络

### Istio
典型的微服务（比如Spring Cloud）都会集成以下几个核心功能：命名服务、配置服务和流控。K8S将这些通用的功能从业务微服务中剥离，部署成一个独立的容器服务SideCar，和业务微服务背靠背部署。

Istio架构如下：
![istio-architect.png](https://ping666.com/wp-content/uploads/2024/12/istio-architect.png "istio-architect.png")

- 优势
  - 命名服务、流控等通用逻辑对业务透明了，业务服务不需要再关注这些。这些工作将会交给DevOps工程师处理。
  - 这一套通用逻辑从微服务下移到基础设施层之后，对各种语言写微服务更友好了。
- 劣势
  - 多了一层，增加了系统复杂度。
  - 对DevOps工程师要求比较高，只有大平台才玩的起。
  - 像Java这种成熟的语言，这些功能原本都是现成的，剥离后开发对服务的掌控力度在减弱。

### K8s API
