# Helm Chart in Kubernetes and ICP (IBM Cloud Private)

本文基于 IBM Cloud Private（ICP）3.1.0 和ICP自带的Kubernetes 1.11.1.  
参考： https://www.ibm.com/support/knowledgecenter/en/SSBS6K_3.1.0/getting_started/whats_new.html

## K8s预备知识
### K8s架构

 - Declarative  
 Declarative 的意思是申明的、陈述的，与之相反的是 Imperative，Imperative 意思是命令式的。首先说一下 Kubernetes 设计完全是按照 Declarative 设计的。Declarative 的定义是用户设定期望的状态，系统会知道它需要执行什么操作，来达到期望的状态。
 - async  
 所有的操作都是异步的。
 - Reconcile
 Reconcile 中文意思是 “调和”，“和解” 的意思，简单的说就是它不断使系统当的状态，向用户期望的状态移动。比如说右边的例子，用户期望的 Replica 是三个，Controller 通过 Watch 发现期望的状态是 3 个，但实际观测到的 Replica 的是 2 个，所以它就会 Create 一个新的 Pod。然后 Controller 会继续 Watch 这些 Pod，当它发现 Create 完成了，就会更新 Status 到 3 个，使 Status 和 Spec 达到一致的状态。
 - 架构  
 ![Helm Repositories](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/helmInK8sAndICP.imgs/k8s-components.png) 
 - 代码结构  
  ![Helm Repositories](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/helmInK8sAndICP.imgs/k8s-src.png) 

 - Scheduler  
   Gets pending Pods from kube-apiserver, assigns a minion to the Pod on which it should run, and writes the assignments back to API server. kube-scheduler assigns minions based on available resources, QoS, data locality and other policies described in its driving algorithm  
 - controller-manager     
    Runs control loops that manage objects from kube-apiserver and perform actions to make sure these objects maintain the states described by their specs  
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

 - https://github.com/kubernetes/kubernetes
 - https://github.com/opencontainers/runc
 - https://kubernetes.io/docs/concepts/ 
 - https://www.jianshu.com/p/5aed73b288f7
 - https://www.kubernetes.org.cn/3031.html
 

### K8s里面的一些术语

 - Deployment.  
   create ReplicaSet
 - Service.  
   loadbalance pod
 - ReplicaSet.  
   Recoucil pods
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

### Deployment如何管理Pods

### Service如何路由服务请求
 
 - 网络.  
 docker 通过 bridge 和 veth pair 能保证容器间的网络通信和基于IP的连接，kubernetes借助calico，flannel等网络层，实现跨node的containers的网络通信以及基于IP的连接，也就是说，所有K8s Clusters上的pod ip都在一个网络上，比如：
  
 ![img](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/helmInK8sAndICP.imgs/network-openswitch.png) 
 ![img](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/helmInK8sAndICP.imgs/network-direct-connect.png)
  
 - 路由  
 Kubernetes services are an abstraction for pods, providing a stable, virtual IP (VIP) address. As pods may come and go, for example in the process of a rolling upgrade, services allow clients to reliably connect to the containers running in the pods, using the VIP. The virtual in VIP means it’s not an actual IP address connected to a network interface but its purpose is purely to forward traffic to one or more pods. Keeping the mapping between the VIP and the pods up-to-date is the job of kube-proxy, a process that runs on every node, which queries the API server to learn about new services in the cluster.
 
 来看一个例子，given svc vip is: 172.30.40.155, pods ip are: 172.17.0.2, 172.17.0.4
 
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

如图：  
![Helm Repositories](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/helmInK8sAndICP.imgs/svc-route.png) 

参考：  

 - https://kubernetes.io/docs/concepts/cluster-administration/networking/
 - https://blog.openshift.com/kubernetes-services-by-example/
 - https://kubernetes.io/docs/concepts/services-networking/service/
 
### Pod的生命周期以及Liveness和Readiness

![Helm Repositories](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/helmInK8sAndICP.imgs/pod-lifecycle.png) 

 - Liveness:  
   Container is running.
 - Readiness:  
   App is runung.
 - postStart
 - preStop

参考：  

 - https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/
 - https://kubernetes.io/docs/tasks/configure-pod-container/attach-handler-lifecycle-event/

 
### log

 - 架构
![Helm Repositories](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/helmInK8sAndICP.imgs/log-1.png)  
 - streaming

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
Now when you run this pod, you can access each log stream separately by running the following commands:  

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


参考：  

- https://github.com/kubernetes/kubernetes/blob/master/cluster/addons/fluentd-elasticsearch/fluentd-es-configmap.yaml
- https://kubernetes.io/docs/concepts/cluster-administration/logging/ 
 
### 服务网格（Service Mesh）& Istio

![Helm Repositories](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/helmInK8sAndICP.imgs/istio-control-plane.png)  

 
参考：  
 
 -  https://istio.io/docs/concepts/what-is-istio/
 
 
### Knative
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


## HelmChart 基础
### 从一个例子开始
 
![Helm Repositories](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/helmInK8sAndICP.imgs/sample-helm-code-structure.png)  

 - .tpl  
   定义的一些全局变量和方法，可以被别的yaml文件引用
 - .yaml  
   定义Kubernetes的资源，比如Deployment, pod, service...
 - Chart.yaml  
   Helm Chart的信息，比如名字，版本，维护者。
 - README.md
 - requirements.yaml  
   定义依赖的Chart。
 - values.yaml  
   定义yaml里面用到的变量，可以被命令行或者yaml文件覆盖。
 - values-metadata.yaml  
   ICP special。
 - charts 目录  
   依赖的Chart包。
 - templates 目录  
   定义chart资源的目录，主要的代码在这里写。。。
 - templates/tests 目录  
   定义一个或者多个Pod，部署完Chart以后，会生成这些Pod执行里面定义的测试程序。
 - templates/deployment.yaml  
   定义的Deployment。
 - templates/service.yaml  
   定义的Service，注意，yaml的文件名可以随意，不用跟定义的资源一致，当然，从维护的角度应该一致。
 - ibm_cloud_pak 目录   
   ICP Special。
 
   
   

 参考：  
 
 - https://github.com/IBM/charts/tree/master/stable/ibm-cem/charts/ibm-sch
 - https://github.com/IBM/charts/tree/master/stable/ibm-nodejs-sample 
 
 
### 与标准Helm Chart 比，ICP里多了什么？
 - values-metadata.yaml  
   这个文件定义了values.yaml的源文件，比如，变量名字，类型等，在Chart安装的时候会用来显示定制的变量以便让用户输入，后面有例子。
 - ibm_cloud_ppa/maniefest.yaml  
   生成PPA的定义文件，后面有解释PPA。。。
 - sch  
   IBM ICP Chart定义的公用的变量和方法，我们在写自己的Chart的时候可以引用。。。

 
 
 
### Helm Chart 最佳实践
略。。。  

参考：  
 
 - https://github.com/helm/helm/tree/master/docs   
 - https://github.com/helm/helm/tree/master/docs/chart_best_practices  
 - https://github.com/ibm/charts  

## 在ICP上的准备工做
### 安装ICP
假定已经安装了ICP。  
安装参考：https://www.ibm.com/support/knowledgecenter/SSBS6K_3.1.0/installing/install_containers_CE.html#prep_boot

### 安装 cloudctl
cloudcli 是一个命令行工具，安装cloudcli的同时也会安装kubectl。通过cloudcli和kubectl可以访问和控制bluemix（IBM Cloud）和 ICP的 K8s cluster.  
安装步骤参考： https://www.ibm.com/support/knowledgecenter/en/SSBS6K_3.1.0/manage_cluster/install_cli.html

### 安装 helm
要创建，打包，安装Helm Chart需要安装Helm客户端程序。  
安装步骤参考：https://www.ibm.com/support/knowledgecenter/en/SSBS6K_3.1.0/app_center/create_helm_cli.html

### 让cloudcli，docker和helm能访问ICP Master
修改client host并copy CA文件使得client安装的cloudcli，kubectl， docker 和helm 能访问 K8s master。  
 
 - edit /etc/hosts and add   
```
172.16.26.215    mycluster.icp
```
 - copy ca file
```
scp root@mycluster.icp:/etc/docker/certs.d/mycluster.icp\:8500/ca.crt ~/.docker/certs.d/mycluster.icp\:8500/ca.crt. 
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain ~/.docker/certs.d/mycluster.icp\:8500/ca.crt  
docker > restart
``` 
  
参考：   
 
 - https://www.ibm.com/support/knowledgecenter/en/SSBS6K_3.1.0/app_center/add_int_helm_repo_to_cli.html  
 - https://www.ibm.com/support/knowledgecenter/en/SSBS6K_3.1.0/manage_images/configuring_docker_cli.html  
 

## 创建和部署Helm Chart （in ICP）

### 创建Helm Chart
 - Helm 命令：  
      
   `helm create sampleApp`.  
   
   参考： https://docs.helm.sh/helm/
 - 从Sample拷贝
 - 按照目录结构分别创建文件。
 
我们以Project “ibm-nodejs-sample” 为例，下面的演示都是以这个为例子。  
例子地址： https://github.com/IBM/charts/tree/master/stable/ibm-nodejs-sample   
### 构建 Helm Chart
 - 用lint命令做静态检查：  
`helm lint ibm-nodejs-sample`  
 - 打包：   
`helm package ibm-nodejs-sample`  
 - 那么一个Helm Chart就会在当前目录构建出来：  
`ibm-nodejs-sample-1.2.1.tgz`. 

### 上传Helm Chart
打包的Helm Chart需要上传到一个Helm Chart Repositiry，任何的HTTP服务器只要提供了GET方法都可作为Helm Chart Repositiry，ICP可以通过 Helm -> Repositories 来添加或者删除Cluester里面的Helm Repo， 如下图：  
![Manage Helm Repositories](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/helmInK8sAndICP.imgs/manage-helm-repos.png).  
![Helm Repositories](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/helmInK8sAndICP.imgs/helm-repos.png)

我们用cloudcli把Chart上传到ICP本地的Chart repo， 命令如下：  

 - 登陆到ICP Master			    
`cloudctl login -a https://172.16.26.215:8443 --skip-ssl-validation`  
  
 - 上传Chart.  	  		
`cloudctl catalog load-chart --archive ibm-nodejs-sample-1.2.1.tgz`


### 安装Helm Chart
在K8s里面一般用命令行安装Chart： `helm install`。  
我们看看在ICP里怎么通过UI来安装，打开： Helm -> Repositories -> Catalog， 如下图：   
![app nodejs in catalog](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/helmInK8sAndICP.imgs/helm-catalog-nodejs.png).   

![nodejs configure](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/helmInK8sAndICP.imgs/nodejs-helm-configure.png)

![nodejs install](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/helmInK8sAndICP.imgs/nodejs-helm-install.png)

![manage workload release](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/helmInK8sAndICP.imgs/workload-helm-release.png)

![check in release](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/helmInK8sAndICP.imgs/nodejs-release.png)

### 上传Image
我们用	`kubectl get po` 命令查看新创建的pod会发现有错误，原因是ICP的worker node 不能访问image registry (docker hub),所以要把docker hub上的image上传到ICP的local image registry上， 后面会介绍另外一种方法： PPA

前面的准备工作已经保证了client端的docker可以访问ICP的docker server。同时在/etc/hosts map了ICP master IP 到domain： mycluster.icp   
 
下面是pull image， tag， push的命令：  
 
 - login icp docker server.  
 `docker login mycluster.icp:8500`
 - pull image from docker hub to local.  
 `docker pull ibmcom/icp-nodejs-sample:8`
 - pull the s390 image.   
 `docker pull ibmcom/icp-nodejs-sample-s390x:8`
 - tag the image.  
 `docker tag ibmcom/icp-nodejs-sample:8 mycluster.icp:8500/ibmcom/icp-nodejs-sample:8`
 - tag the s390x image.   
 `docker tag ibmcom/icp-nodejs-sample-s390x:8 mycluster.icp:8500/ibmcom/icp-nodejs-sample-s390x:8`
 - push the image from local to ICP docker registry.  
 `docker push mycluster.icp:8500/ibmcom/icp-nodejs-sample:8`
 - push the s390x image.  
 `docker push mycluster.icp:8500/ibmcom/icp-nodejs-sample-s390x:8`

				
	
我们到ICP的管理界面看看上传的image：  			
![images nodejs](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/helmInK8sAndICP.imgs/images-nodejs.png)				
				
				
				

### 修改image repo
这时候images都上传到ICP了，再来看`kubectl get po` pod还是有错误，因为image的URL发生了变化，我们再来改一下`Deployment`的`image url`.  
执行下面的命令：   
`kubectl edit deploy ibm-nodejs-sample-nodejssample-nodejs -n default`
替换下面的响应的`image url`:  
`mycluster.icp:8500/ibmcom/icp-nodejs-sample:8`.  
同时修改下面的几个变量：  
`- name: CLUSTER_CA_DOMAIN    
   value: mycluster.icp`    
`imagePullPolicy: IfNotPresent `


### 查看结果
再看pod的状态，结果就应该是这样的了：  

```
huoqifengdembp:document huoqifeng$ kubectl get po  
NAME                                                     READY   STATUS    RESTARTS   AGE  
ibm-nodejs-sample-nodejssample-nodejs-699d45cf49-274l8   1/1     Running   0          7h
```
再到ICP的管理界面：  
![images nodejs](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/helmInK8sAndICP.imgs/release-to-launch.png).  
点击 `launch` 就打开了这个sample app.  
![images nodejs](https://raw.githubusercontent.com/huoqifeng/document/master/k8s/helmInK8sAndICP.imgs/nodejs-launch.png)	

我们可以看到，这里IP地址是ICP Master的IP，port是前面创建的service的port。 
### 为Pod选择worker节点
默认的image是用的amd64，我们部署的cluster实际上包含两种worker node，x86节点和基于LAPR的s390x节点， 如下：  

```
huoqifengdembp:document huoqifeng$ kubectl get nodes --show-labels
NAME            STATUS   ROLES                          AGE   VERSION          LABELS
172.16.26.215   Ready    etcd,management,master,proxy   16d   v1.11.1+icp-ee   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,etcd=true,kubernetes.io/hostname=172.16.26.215,management=true,master=true,node-role.kubernetes.io/etcd=true,node-role.kubernetes.io/management=true,node-role.kubernetes.io/master=true,node-role.kubernetes.io/proxy=true,proxy=true,role=master
172.16.26.216   Ready    worker                         16d   v1.11.1+icp-ee   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=172.16.26.216,node-role.kubernetes.io/worker=true
172.16.32.185   Ready    worker                         15d   v1.11.1+icp-ee   beta.kubernetes.io/arch=s390x,beta.kubernetes.io/os=linux,kubernetes.io/hostname=172.16.32.185,node-role.kubernetes.io/worker=true

```

我们看到节点`172.16.26.216`的标签包含`beta.kubernetes.io/arch=amd64`, 节点`172.16.32.185`的标签包含`beta.kubernetes.io/arch=s390x`,这是因为216节点是x86架构，185节点是s390架构，他们在加入K8s cluster的时候会被自动加上响应的arch标签。  

现在我们就用这个标签，通过修改deployment来在创建pod的时候选择worker node。  
通过下面的命令编辑Deployment：  
```
kubectl edit deploy ibm-nodejs-sample-nodejssample-nodejs -n default
```   
增加下面的代码：  

```
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: beta.kubernetes.io/arch
                operator: In
                values:
                - s390x
```   
同时要修改相应的image：  

```
mycluster.icp:8500/ibmcom/icp-nodejs-sample-s390x:8
```


我们再来看，pod已经重新建立并且被分配到了s390的节点，我们可以用kubectl来查一下：   

```
huoqifengdembp:document huoqifeng$ kubectl describe node 172.16.32.185
Name:               172.16.32.185
Roles:              worker
Labels:             beta.kubernetes.io/arch=s390x
                    beta.kubernetes.io/os=linux
                    kubernetes.io/hostname=172.16.32.185
                    node-role.kubernetes.io/worker=true
Annotations:        node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Wed, 05 Dec 2018 12:53:28 +0800
Taints:             <none>
Unschedulable:      false
  Namespace                  Name                                                      CPU Requests  CPU Limits  Memory Requests  Memory Limits  AGE
  ---------                  ----                                                      ------------  ----------  ---------------  -------------  ---
  default                    ibm-nodejs-sample-nodejssample-nodejs-699d45cf49-274l8    100m (1%)     100m (1%)   128Mi (0%)       128Mi (0%)     8h
  kube-system                audit-logging-fluentd-ds-qf8g7                            0 (0%)        0 (0%)      0 (0%)           0 (0%)         15d
  kube-system                calico-node-49lfl                                         300m (3%)     0 (0%)      150Mi (0%)       0 (0%)         15d
  kube-system                k8s-proxy-172.16.32.185                                   0 (0%)        0 (0%)      0 (0%)           0 (0%)         15d
  kube-system                logging-elk-filebeat-ds-nv5sm                             0 (0%)        0 (0%)      0 (0%)           0 (0%)         15d
  kube-system                metering-reader-l4dk4                                     250m (2%)     0 (0%)      512Mi (1%)       0 (0%)         15d
  kube-system                monitoring-prometheus-nodeexporter-48bpk                  0 (0%)        0 (0%)      0 (0%)           0 (0%)         15d
  kube-system                nvidia-device-plugin-plrxg                                150m (1%)     0 (0%)      0 (0%)           0 (0%)         15d

```


当然，你也可以自己添加或者删除标签来实现，比如：  

```
kubectl label nodes 172.16.32.185 arch=lpar
kubectl label nodes 172.16.26.216 arch=x86
				
kubectl label nodes 172.16.32.185 arch-
kubectl label nodes 172.16.26.216 arch-
```

更多的选择选项请参考下面的文档：  

参考：  

 - https://kubernetes.io/docs/tasks/configure-pod-container/assign-pods-nodes/
 - https://kubernetes.io/docs/concepts/configuration/assign-pod-node/

 

### Scale pods
接下来我们来scale部署的pod数量：  
`kubectl scale deploy ibm-nodejs-sample-nodejssample-nodejs --replicas=2 -n default`  

查看结果：  

```
huoqifengdembp:document huoqifeng$ kubectl get po
NAME                                                     READY   STATUS    RESTARTS   AGE
ibm-nodejs-sample-nodejssample-nodejs-699d45cf49-274l8   1/1     Running   0          8h
ibm-nodejs-sample-nodejssample-nodejs-699d45cf49-qwk9v   1/1     Running   0          1d

```
### 为ICP制作ppa
为什么需要PPA？我们前面的例子也碰到了，当ICP的节点不能访问互联网的时候，必须手工的把image上传到ICP本地的registry，这带来很多不必要的麻烦，所以ICP提供了另外一种打包方式 -- PPA(Packaged Passport Advantage)， 通过PPA，Helm Chart和Docker Image可以被打包到一个压缩文件里面，比如：  

 - 打包命令：  
`cloudctl catalog create-archive`
 - 加载命令：  
`cloudctl catalog load-archive`

参考：  

 - https://www.ibm.com/support/knowledgecenter/en/SSBS6K_3.1.0/manage_cluster/cli_catalog_commands.html

 
### Helm Chart & PPA 开发流程
参考：  

- http://icp-content-playbook.rch.stglabs.ibm.com/publishing-content/







