# k8s_OpenKruise_config

之前做K8S的时候学的东西，最近要清电脑，删掉实在可惜，直接备份一下好了。同时可以在知乎查看 https://zhuanlan.zhihu.com/p/589008170

## **OpenKruise介绍、部署及核心功能实验**

OpenKruise 由 Alibaba 开源，是 Kubernetes 的一个标准扩展，它可以配合原生 Kubernetes 使用，对云原生应用的自动化，比如部署、发布、运维以及可用性防护做了很多贡献。

OpenKruise 是面向自动化场景的 Kubernetes 应用负载扩展控制器。而 Kruise 是 OpenKruise 项目的核心，它是一组控制器，可在应用程序工作负载管理上扩展和补充 Kubernetes 核心控制器。OpenKruise 弥补了 Kubernetes 在应用部署、升级、防护、运维等领域的不足。

## 实验环境

```
Mac Catalina
lens(k8s桌面端管理工具)
腾讯云EKS集群 (https://console.cloud.tencent.com/tke2/cluster?rid=1)
```

## **核心功能**

- 原地升级：原地升级是一种可以避免删除、新建 Pod 的升级镜像能力。它比原生 Deployment/StatefulSet 的重建 Pod 升级更快、更高效，并且避免对 Pod 中其他不需要更新的容器造成干扰
- Sidecar 管理：支持在一个单独的 CR 中定义 sidecar 容器，OpenKruise 能够帮你把这些 Sidecar 容器注入到所有符合条件的 Pod 中。这个过程和 Istio 的注入很相似，但是你可以管理任意你关心的 Sidecar
- 跨多可用区部署：定义一个跨多个可用区的全局 workload、容器，OpenKruise 会帮你在每个可用区创建一个对应的下属 workload。你可以统一管理 workload 的副本数、版本、甚至针对不同可用区采用不同的发布策略
- 镜像预热：支持用户指定在任意范围的节点上下载镜像

## **安装**

[中文文档](https://openkruise.io/zh/docs/installation/)

换成阿里源才顺利

helm install kruise openkruise/kruise --version 1.3.0 --set  manager.image.repository=openkruise-registry.cn-hangzhou.cr.aliyuncs.com/openkruise/kruise-manager

[在charts里指定需要的参数](https://openkruise.io/zh/docs/installation#可选-chart-安装参数)

## **架构**

整体架构如下：

![img](https://picx.zhimg.com/80/v2-6f15e1801c288405e76c720545e522bf_720w.png?source=d16d100b)





添加图片注释，不超过 140 字（可选）

OpenKruise所有功能都是通过CRD来提供的

### **CRD列表**

- CloneSet    提供更加高效、确定可控的应用管理和部署能力，支持优雅原地升级、指定删除、发布顺序可配置、并行/灰度发布等丰富的策略，可以满足更多样化的应用场景
- Advanced StatefulSet    基于原生 StatefulSet 之上的增强版本，默认行为与原生完全一致，在此之外提供了原地升级、并行发布（最大不可用）、发布暂停等功能
- SidecarSet  对 sidecar 容器做统一管理，在满足 selector 条件的 Pod 中注入指定的 sidecar 容器
- UnitedDeployment    通过多个 subset workload 将应用部署到多个可用区
- BroadcastJob    配置一个 job，在集群中所有满足条件的 Node 上都跑一个 Pod 任务
- Advanced DaemonSet  基于原生 DaemonSet 之上的增强版本，默认行为与原生一致，在此之外提供了灰度分批、按 Node label 选择、暂停、热升级等发布策略
- AdvancedCronJob     一个扩展的 CronJob 控制器，目前 template 模板支持配置使用 Job 或 BroadcastJob
- ImagePullJob        支持用户指定在任意范围的节点上下载镜像

## **实验**

### **一、部署**

部署服务的yaml文件

```
apiVersion: apps.kruise.io/v1alpha1
kind: CloneSet
metadata:
  name: demo
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: cs
  template:
    metadata:
      labels:
        app: cs
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
```

将上面命名为文件xxx.yaml，然后通过kubectl apply -f xxx.yaml创建

```
kubectl get cloneset demo

kubectl describe cloneset demo
```

创建成功之后通过 kubectl describe命令查看对应的 Events 信息，可以发现 cloneset-controller 是直接创建的 Pod，而原生的Deployment 是通过 ReplicaSet 去创建的 Pod

### **二、扩容**

```
apiVersion: apps.kruise.io/v1alpha1
kind: CloneSet
metadata:
  name: demo
  namespace: default
spec:
  minReadySeconds: 30
  scaleStrategy:
    maxUnavailable: 1  ## to scale, deployment还有一个maxSurge
  replicas: 5
  selector:
    matchLabels:
      app: cs
  template:
    metadata:
      labels:
        app: cs
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
```

minReadySeconds: 60 创建了一个pod之后一分钟才会创建第二个

### **三、缩容**

```
apiVersion: apps.kruise.io/v1alpha1
kind: CloneSet
metadata:
  name: demo
  namespace: default
spec:
  minReadySeconds: 30
  scaleStrategy:
    maxUnavailable: 1
    podsToDelete:
    - demo-9gmzs     # 优先删除这些
  replicas: 4
  selector:
    matchLabels:
      app: cs
  template:
    metadata:
      labels:
        app: cs
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
```

- 缩容时，CloneSet可以指定一些pod删除，而 StatefulSet 或者 Deployment 做不到:
- **StatefulSet 是根据序号来删除 Pod**，而 **Deployment/ReplicaSet 目前只能根据控制器里定义的排序来删除**。而 CloneSet 允许用户在缩小 replicas 数量的同时，指定想要删除的 Pod 名字
- 如果只是把name加入podsToDelete，而没有修改replicas的话，删完之后会再扩一个pod
- **apps.kruise.io/specified-delete: true** is also ok

- **podsToDelete**或 **apps.kruise.io/specified-delete: true**会有CloneSet 的 maxUnavailable/maxSurge来保护删除， 并且会触发 PreparingDelete生命周期的钩子：[官方doc](https://openkruise.io/docs/user-manuals/cloneset/#scaling-with-preparingdelete)

### **四、升级**

```
apiVersion: apps.kruise.io/v1alpha1
kind: CloneSet
metadata:
  name: demo
  namespace: default
spec:
  minReadySeconds: 30 
  updateStrategy:
    type: InPlaceIfPossible  #修改1：指定为最大限度的原地升级的方式
  scaleStrategy:
    maxUnavailable: 1  
  replicas: 4
  selector:
    matchLabels:
      app: cs
  template:
    metadata:
      labels:
        app: cs
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9 #修改2：把镜像版本替换了
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
```

原地升级：

- ReCreate : 删除旧 Pod 和它的 PVC，然后用新版本重新创建出来，这是默认的方式。 (deployment这里默认的升级策略是滚动升级，但这里CloneSet默认的策略是ReCreate。)
- InPlaceIfPossible : 会优先尝试原地升级 Pod，如果不行再采用重建升级
- InPlaceOnly : 只允许采用原地升级，因此，**用户只能修改上一条中的限制字段**，如果尝试修改其他字段会被拒绝

![img](https://pic1.zhimg.com/80/v2-ef1ceb156a7f9d826f30c7a74c7727fe_720w.png?source=d16d100b)





添加图片注释，不超过 140 字（可选）

区别：

- recreate:
- Pod name and did, 因为pod对象不同了（like deployment）
- uid， 因为对象不同，只是复用了同一个name（like statefulset）
- node name，调度节点大概率不同
- ip，同上

- inplace:
- 调度、分配 IP、挂载盘带来的开销
- 镜像拉取，like dockerfile的分层结构
- 一个pod中的container升级时，其他container不受影响

运行完04-yaml后，会发现pod的状态只有tag变了

可以通过下列命令查看运行情况

```
kubectl get po -l app=cs

kubectl describe cloneset demo 

kubectl describe po demo-xxxxx

kubectl get po demo-xxxxx -oyaml|grep image
```

### **五、灰度**

```
apiVersion: apps.kruise.io/v1alpha1
kind: CloneSet
metadata:
  name: demo
  namespace: default
spec:
  minReadySeconds: 30 
  updateStrategy:
    type: InPlaceIfPossible  
    partition: 2 #修改1：保留旧版本pod的数量
  scaleStrategy:
    maxUnavailable: 1  
  replicas: 4
  selector:
    matchLabels:
      app: cs
  template:
    metadata:
      labels:
        app: cs
    spec:
      containers:
      - name: nginx
        image: nginx:latest #修改2：把镜像版本替换了
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
```

如果安装的时候启用了[PreDownloadImageForInPlaceUpdate](https://openkruise.io/docs/installation#optional-feature-gate)这个feature，会在所有旧版本pod预热正在灰度的版本

```
kubectl explain cloneset.spec.updateStrategy.type

kubectl explain cloneset.spec.updateStrategy.inPlaceUpdateStrategy
```

**分批灰度**：修改partition（旧版本的pod数量），数字的话=(replicas - partition)，百分比的话=(replicas * (100% - partition))

```
kubectl apply -f 05-cloneset-updateStrategy-partition.yaml 

kubectl get pods -l app=cs -L controller-revision-hash

kubectl describe po demo-9gmzs
```

还可以指定优先级等，[https://openkruise.io/zh/docs/user-manuals/cloneset#%E5%8D%87%E7%BA%A7%E9%A1%BA%E5%BA%8F](https://openkruise.io/zh/docs/user-manuals/cloneset#升级顺序)

### 六、最大的坑

中间更换过一次eks集群，之后再kubectl apply -f xxx.yaml的时候便会报错，截止到发布这篇文章的时候，由于框架很新，基本都没有详细的解决方案，但可以根据GitHub这篇issue排查一下：https://github.com/openkruise/kruise/issues/535

报错信息：

![img](https://picx.zhimg.com/80/v2-3e7907757c634abfbbf5b0e21fe6f345_720w.png?source=d16d100b)



