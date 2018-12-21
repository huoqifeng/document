# Kubernetes and ICP (IBM Cloud Private) 1 - K8s

本文基于IBM Cloud Private(ICP)3.1.0和ICP自带的Kubernetes 1.11.1.  
参考： https://www.ibm.com/support/knowledgecenter/en/SSBS6K_3.1.0/getting_started/whats_new.html

## K8s预备知识

Kubernetes是容器编排系统事实上的标准，作为学习，本文主要介绍K8s在设计上的一些考虑，架构，代码机构，主要模块，和容易引起困惑的概念。。。

### 设计上的考虑

 - 声明式(Declarative)  
 Declarative 的意思是申明的、陈述的，与之相反的是 Imperative，Imperative 意思是命令式的。首先说一下 Kubernetes 设计完全是按照 Declarative 设计的。Declarative 的定义是用户设定期望的状态，系统会知道它需要执行什么操作，来达到期望的状态。
 - 异步(async)  
 所有的操作都是异步的。
 - 调和(Reconcile)
 Reconcile中文意思是 “调和”，“和解” 的意思，简单的说就是它不断使系统当的状态，向用户期望的状态移动。比如说，用户期望的 Replica 是三个，Controller 通过 Watch 发现期望的状态是 3 个，但实际观测到的 Replica 的是 2 个，所以它就会 Create 一个新的 Pod。然后 Controller 会继续 Watch 这些 Pod，当它发现 Create 完成了，就会更新 Status 到 3 个，使 Status 和 Spec 达到一致的状态。
 
### K8s架构
 
 ![Helm Repositories](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/helmInK8sAndICP.imgs/k8s-components.png) 

下面是主要的模块，有些是直接copy过来的。。。  

 - Scheduler  
   Scheduler 从 api-server 获取应该编排的 pod， 基于可用的 resources, QoS, data locality 和其他 policy 来获取和分配 Pod 应该在哪个节点运行，并把绑定的结果发送回 api-server。 
 - controller-manager     
    controller-manager 从 api-server 读取资源的状态并采取相应的动作以保持资源和他们的声明一致，包括几个： 
    - Node Controller:  
    Responsible for noticing and responding when nodes go down.
    - Replication Controller:  
    Responsible for maintaining the correct number of pods for every replication controller object in the system.
    - Endpoints Controller:  
    Populates the Endpoints object (that is, joins Services & Pods).
    - Service Account & Token Controllers:  
    Create default accounts and API access tokens for new namespaces.
 - apiserver  
   The only rest server run on master, all request must reach to apiserver, and api server communicate with etcd.
 - kubelet  
   A Kubernetes worker that runs on each minion. It watches Pods via kube-apiserver and looks for Pods that are assigned to itself. It then syncs these Pods if possible. The procedure of Syncing Pods requires resource provisioning (i.e. mount volume), talking with container runtime to manage Pod life cycle (i.e. pull images, run containers, check container health, delete containers and garbage collect containers)  
 - kube-proxy  
   A network proxy that reflects Service (defined in Kubernetes REST API) that runs on each node. Watches Service and Endpoint objects from kube-apiserver and modifies the underlying kernel iptable for routing and redirection.  
 - etcd  
   基于Raft一致性协议的高可靠的KV存储系统 vs ZooKeeper 基于一致性协议Zab（Zookeeper Atomic Broadcas）

 参考：  

 - https://kubernetes.io/docs/concepts/ 
 - https://www.jianshu.com/p/5aed73b288f7
 - https://www.kubernetes.org.cn/3031.html
 
### 代码结构  
  ![Helm Repositories](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/helmInK8sAndICP.imgs/k8s-src.png)  
 
  参考：  

 - https://github.com/kubernetes/kubernetes
 - https://github.com/opencontainers/runc


### K8s里面的一些常用资源

下面这些概念还是建议看一下列出的参考文档。。。  

 - Deployment.  
   create ReplicaSet
 - Service.  
   loadbalance pods
 - ReplicaSet.  
   Pods 编排状态的声明
 - Pod. 
   Contain containers
 - Container. 
 - Ingress.  
   Internet -> ingress -> service -> pod

 
参考：  

 - https://kubernetes.io/docs/concepts/services-networking/ingress/
 - https://kubernetes.io/docs/concepts/containers/images/
 - https://kubernetes.io/docs/concepts/services-networking/service/
 - https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/


### Service如何路由服务请求
 
 Service 的路由是比较容易引起困扰的地方，我们先从K8s的网络讲起。。。
 
 - 网络.  
 docker 通过 bridge 和 veth pair 能保证容器间的网络通信和基于IP的连接，kubernetes借助calico，flannel等网络层，实现跨node的containers的网络通信以及基于IP的连接，也就是说，所有K8s Clusters上的pod ip都在一个网络上，比如：
  
 ![img](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/helmInK8sAndICP.imgs/network-open-switch.png) 
 ![img](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/helmInK8sAndICP.imgs/network-direct-connect.png)
  
 在K8s的各个节点上，所有的pods的IP是可以互相访问到的，我们接着再来看看一个Service的路由，实现路由的主要模块是运行在每个节点上的kube-proxy，下面这段是copy自openshift，直接搬啦用吧。。。 
  
 - 服务(Service)和路由  
 Kubernetes services are an abstraction for pods, providing a stable, virtual IP (VIP) address. As pods may come and go, for example in the process of a rolling upgrade, services allow clients to reliably connect to the containers running in the pods, using the VIP. The virtual in VIP means it’s not an actual IP address connected to a network interface but its purpose is purely to forward traffic to one or more pods. Keeping the mapping between the VIP and the pods up-to-date is the job of kube-proxy, a process that runs on every node, which queries the API server to learn about new services in the cluster.
 
 来看一个例子，如果我们又一个 service 名字是sampleservice, 它的 vip: 172.30.40.155 端口： 80, 它对应的 pods ip 是: 172.17.0.2, 172.17.0.4， Pod 的监听端口是： 9876， kube-proxy在每个node上维护的iptables是这样的：  
 
 ```
 $ sudo iptables-save | grep simpleservice
-A KUBE-SEP-ASIN52LB5SMYF6KR -s 172.17.0.2/32 -m comment --comment "namingthings/simpleservice:" -j KUBE-MARK-MASQ
-A KUBE-SEP-ASIN52LB5SMYF6KR -p tcp -m comment --comment "namingthings/simpleservice:" -m tcp -j DNAT --to-destination 172.17.0.2:9876
-A KUBE-SEP-RP53IYKEFRDLQANZ -s 172.17.0.4/32 -m comment --comment "namingthings/simpleservice:" -j KUBE-MARK-MASQ
-A KUBE-SEP-RP53IYKEFRDLQANZ -p tcp -m comment --comment "namingthings/simpleservice:" -m tcp -j DNAT --to-destination 172.17.0.4:9876
-A KUBE-SERVICES -d 172.30.40.155/32 -p tcp -m comment --comment "namingthings/simpleservice: cluster IP" -m tcp --dport 80 -j KUBE-SVC-IKIIGXZ2IBFIBYI6
-A KUBE-SVC-IKIIGXZ2IBFIBYI6 -m comment --comment "namingthings/simpleservice:" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-ASIN52LB5SMYF6KR
-A KUBE-SVC-IKIIGXZ2IBFIBYI6 -m comment --comment "namingthings/simpleservice:" -j KUBE-SEP-RP53IYKEFRDLQANZ
```

通过这个iptables，对 172.30.40.155:80 的访问都被转发到 172.17.0.2:9876 或者 172.17.0.4:9876 上。

如图：  
![Helm Repositories](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/helmInK8sAndICP.imgs/svc-route.png) 

当然，通过kube-dns 也可以直接用service name访问， 比如：  
```
sampleservice:80
```


参考：  

 - https://kubernetes.io/docs/concepts/cluster-administration/networking/
 - https://blog.openshift.com/kubernetes-services-by-example/
 - https://kubernetes.io/docs/concepts/services-networking/service/
 
### Pod的生命周期以及Liveness和Readiness

Pod的状态有下面几种，从创建到销毁经过了不同的阶段和状态。

![Helm Repositories](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/helmInK8sAndICP.imgs/pod-lifecycle.png) 

K8s提供了一些Hook的方法可以在不同阶段执行一些定制的代码来实现一些功能，比如健康检查，优雅重启，failover等等，下面是一些常用的hook。。。

 - Liveness:  
   Container is running.
 - Readiness:  
   App is runing.
 - postStart  
   Pod start 之后执行的脚本或程序
 - preStop  
   Pod stop 之前执行的脚本或程序
   
参考：  

 - https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/
 - https://kubernetes.io/docs/tasks/configure-pod-container/attach-handler-lifecycle-event/

 
### logging

理论上讲，K8s的基于 ELK 的logging解决方案不需要在app层做任何事情，apps只要把log打到 stdout和stderr就可以了。
基于docker的实现，所有containers的stdout和stderr都会写在宿主机，也就是k8s worker node的如下目录：  
 
```
/var/log/containers/POD-NAME_NAMESPACE_CONTAINER-NAME.log
```

其中 POD-NAME， NAMESPACE， CONTAINER-NAME 我猜大家都知道是什么意思。收集log的是跑在每个节点上的fluentd， 什么是fluentd？

Fluentd as a agent can collect all the logs, will also adds some Kubernetes-specific information to the logs. For example, it adds labels to each log message to give the logs some metadata which can be critical in better managing the flow of logs across different sources and endpoints. 


下面是几种应用模式。。。  

 - Use a node-level logging agent that runs on every node.  
 标准模式，app把log打到stdout和stderr就完事大吉。。。  
![Helm Repositories](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/helmInK8sAndICP.imgs/log-1.png)  

 - Using a sidecar container with the logging agent

通过这个Sidecar可以把Containers里写到不同本地log的文件 streaming 到node上的统一的log文件上。。。

![Helm Repositories](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/helmInK8sAndICP.imgs/log-2.png) 

列一些怎么配置。。。

```
apiVersion: v1
kind: Pod
metadata:
  name: counter
spec:
  containers:
  - name: count
    image: busybox
    args:
    - /bin/sh
    - -c
    - >
      i=0;
      while true;
      do
        echo "$i: $(date)" >> /var/log/1.log;
        echo "$(date) INFO $i" >> /var/log/2.log;
        i=$((i+1));
        sleep 1;
      done
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  - name: count-log-1
    image: busybox
    args: [/bin/sh, -c, 'tail -n+1 -f /var/log/1.log']
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  - name: count-log-2
    image: busybox
    args: [/bin/sh, -c, 'tail -n+1 -f /var/log/2.log']
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  volumes:
  - name: varlog
    emptyDir: {}
```

创建了这个pod以后，可以查看到下面的结果:  


```
$ kubectl logs counter count-log-1
0: Mon Jan  1 00:00:00 UTC 2001
1: Mon Jan  1 00:00:01 UTC 2001
2: Mon Jan  1 00:00:02 UTC 2001
...

$ kubectl logs counter count-log-2
Mon Jan  1 00:00:00 UTC 2001 INFO 0
Mon Jan  1 00:00:01 UTC 2001 INFO 1
Mon Jan  1 00:00:02 UTC 2001 INFO 2
...
```

 - Include a dedicated sidecar container for logging in an application pod.  
 Pod里面的Sidecar(Container) 收集同一个Pod的log，同时发送到log server。
 
![Helm Repositories](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/helmInK8sAndICP.imgs/log-3.png) 

来看看怎么配：  
configure fluentd:  

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
data:
  fluentd.conf: |
    <source>
      type tail
      format none
      path /var/log/1.log
      pos_file /var/log/1.log.pos
      tag count.format1
    </source>

    <source>
      type tail
      format none
      path /var/log/2.log
      pos_file /var/log/2.log.pos
      tag count.format2
    </source>

    <match **>
      type google_cloud
    </match>
```

configure pod:  
 
```
apiVersion: v1
kind: Pod
metadata:
  name: counter
spec:
  containers:
  - name: count
    image: busybox
    args:
    - /bin/sh
    - -c
    - >
      i=0;
      while true;
      do
        echo "$i: $(date)" >> /var/log/1.log;
        echo "$(date) INFO $i" >> /var/log/2.log;
        i=$((i+1));
        sleep 1;
      done
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  - name: count-agent
    image: k8s.gcr.io/fluentd-gcp:1.30
    env:
    - name: FLUENTD_ARGS
      value: -c /etc/fluentd-config/fluentd.conf
    volumeMounts:
    - name: varlog
      mountPath: /var/log
    - name: config-volume
      mountPath: /etc/fluentd-config
  volumes:
  - name: varlog
    emptyDir: {}
  - name: config-volume
    configMap:
      name: fluentd-config
```

 - Exposing logs directly from the application. 

app直接发送log的模式不是很常用，不好reuse，不好建policy，不好。。。
![Helm Repositories](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/helmInK8sAndICP.imgs/log-4.png)

参考：  

- https://github.com/kubernetes/kubernetes/blob/master/cluster/addons/fluentd-elasticsearch/fluentd-es-configmap.yaml
- https://kubernetes.io/docs/concepts/cluster-administration/logging/ 
- https://timber.io/blog/collecting-application-logs-on-kubernetes/
- https://docs.fluentd.org/v0.12/articles/routing-examples 
 
 
### 服务网格（Service Mesh）& Istio

 - 为什么需要Service Mesh？   
   参考： https://zhuanlan.zhihu.com/p/43665210  
  
 - Istio是什么？  

![Helm Repositories](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/helmInK8sAndICP.imgs/istio-control-plane.png)  

 
参考：  
 
 -  https://istio.io/docs/concepts/what-is-istio/
 
 
### Serverless & Knative
目标： 技术上基于K8s CRD，实现serverless 平台 (function as a service)，自动完成代码到容器的构建。

![Helm Repositories](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/helmInK8sAndICP.imgs/knative.png) 

 - Build： 
   构建系统，把用户定义的函数和应用 build 成容器镜像
 - Serving： 
   服务系统，用来配置应用的路由、升级策略、自动扩缩容等功能
 - Eventing： 
   事件系统，用来自动完成事件的绑定和触发

参考：  

 - https://github.com/knative
 - https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/


介绍了K8s的基础知识，下面一节会介绍K8s的包管理工具 Helm Chart 以及怎么创建一个Helm Chart。。。








